[TOC]



# 押注和抽探索卡

```java
	/**
	 * 下注
	 * @param data 下注信息
	 * @param user 用户
	 */
	private void bet(Map<String, String> data, User user) {
		try {
			int status = 0;
			String desc = "";
			// if(user.getUserGameTimeM().getBetStageType().isBetStage())
			// user.getUserGameM().userBetRecord(data);
			// else
			// status= -1;
			// 0成功 -1非押注阶段
			// 判断押注是否符合规则
			Boolean isDataSuc = user.getUserGameM().userBetRecord(data);
			if (!isDataSuc) {
				status = -99;
				desc = "下注失败,押注错误";
			}
			// 判断押注是否为0
			if (status == 0 && user.getUserGameM().getTotalBetCoin() == 0) {
				status = -1;
				desc = "下注失败,暂无押注";
			}
			// 获取总押注
			Double totalBet = user.getUserGameM().getTotalBetCoin().doubleValue();
			if (status == 0 && !user.galaOut(totalBet)) {
				status = -2;
				desc = "下注失败,交易失败";
			}
			String prize = "{}";
			if (status == 0) {
				// 计算奖励发奖
				OpenPrizeClient openPrizeClient = user.getUserGameTimeM().openPrize();
				prize = JsonUtils.entity2Json(openPrizeClient);
			}
			SenderMsg.userMsg(user, Protocol.bet(status, desc, prize));
			if (status == 0) {
				if (totalBet < 1000)
					return;
				// 出探索卡概率 EAT_BET= .1 EAT_PRIZE_YUNYING= .003 HARDRATE= .3
				double rate = totalBet * (SysConstant.EAT_BET + SysConstant.EAT_PRIZE_YUNYING) * .1
						* SysConstant.HARDRATE / 500;
				if (rate > .5)
					rate = .5;
				double random = RandomUtils.nextDouble();
				boolean flag = rate > random;
				log.info(Msg.printMsgInfo(user, "探索卡概率[" + rate + "], 随机概率[" + random + "], 是否发放探索卡[" + flag + "]",
						"判断是否抽探索卡"));
				if (flag) {
					UserBalanceM userBalanceM = user.getUserBalanceM();
					String addr = userBalanceM.getAddr();
					String itemId = ItemType.EXPLORE_CARD.getItemId();
					String amount = "1";
					String userId = userBalanceM.getUserId();
					// 抽卡
					baseInvoke.getHttpInterManager().addItemPost(addr, userId, itemId, amount);
											baseInvoke.getPoolInfoManager().getPoolInfo().itemExploreCardCntAdd(Integer.valueOf(amount));
				}
			}
		} catch (Exception e) {
			log.error("", e);
		}
	}
```

# 用户押注记录并计算总押注

```java

	/**
	 * 用户押注记录并计算总押注
	 * 
	 * @param data
	 *            押注数据
	 */
	public Boolean userBetRecord(Map<String, String> data) {
		try {
			String tempBetData = "";
			// 获取总压住
			for (Entry<BetType, Integer> entry : curBetMap.entrySet()) {
				BetType key = entry.getKey();
				Integer betCoin = data.get(key.getId()) == null ? 0 : Integer.parseInt(data.get(key.getId()));
				curBetMap.put(key, betCoin);
				totalBetCoin += betCoin;
				tempBetData += key.getId() + "-" + betCoin + ",";
			}
			logger.debug("### totalBetCoin: " + totalBetCoin);
			logger.debug("### curBetMap: " + tempBetData.substring(0, tempBetData.length() - 1));
			// 获取最小押注结果
			final int betCoinMin = BetCoinType.findBetCoinMin();
			// 判断是否符合规则
			if (totalBetCoin % betCoinMin != 0) {
				logger.error(JsonUtils.entity2Json(data), "押注数据错误[用户自定义接口,取消本次押注] ### parameter");
				resetData();
				return false;
			}
			return true;
		} catch (Exception e) {
			logger.error("", e);
		}
		return false;
	}

```

# 扣除gala

```java
	/**
	 * 扣除gala
	 * @param gala 
	 * @return 是否扣款成功
	 */
	public boolean galaOut(Double gala) {
		return userBalanceM.galaOut(gala);
	}

	/**
	 * http 扣除gala
	 * @param gala
	 * @return
	 */
	public Boolean galaOut(final Double gala) {
		final HttpInterManager httpInterManager= baseInvoke.getHttpInterManager();
		final ContractManager contractManager= baseInvoke.getContractManager();
		Future<String> futureGalaOut= executorPool.submit(new Callable<String>() {
			@Override
			public String call() throws Exception {
				Thread.currentThread().setName("thread_galaOut_"+ user.getNickName());
				return httpInterManager.galaOut(keyStore, pwd, addr, gala);
			}
		});
		Boolean verify= false;
		String hash= "";
		try {
			// 获得交易hash
			hash = futureGalaOut.get();
		} catch (InterruptedException e) {
			log.error("", e);
		} catch (ExecutionException e) {
			log.error("", e);
		}
		if(StringUtils.isEmpty(hash))
			return verify;
		SenderMsg.userMsg(user, Protocol.oper_hash(hash, GalaOperType.OUT));
		// 确认交易hash
		verify= contractManager.getSmartEventCodeByHash(hash);
		return verify;
	}
	
```

# 开奖

```java
	/**
	 * 开奖
	 * 
	 * @return 奖励结果对象
	 */
	public final OpenPrizeClient openPrize() {
		// 每局标示
		everyId = IdGenerator.generate();
		// 获取开奖结果
		OpenPrize openPrize = user.getUserGameM().userBet();
		if (openPrize == null) {
			return new OpenPrizeClient(null);
		}
		// 生成客户端需要数据
		OpenPrizeClient openPrizeClient = new OpenPrizeClient(openPrize);
		// 发奖并记录数据
		user.getUserGameM().sendPrize(everyId);
		// 重置数据
		user.getUserGameM().resetData();

		return openPrizeClient;
	}

```

## 计算押注进入奖池 计算费用

```java
	/**
	 * 计算押注进入奖池 计算费用
	 */
	public OpenPrize userBet() {
		try {
			// 计算每个押注奖励
			for (Entry<BetType, Integer> entry : curBetMap.entrySet())
				prizeMap.put(entry.getKey(), entry.getValue() * entry.getKey().getMul());
			// 根据奖池获取开奖
			openPrize = PoolManager.addPoolAndCalcPrize(totalBetCoin, user.getUserDataInfo(), prizeMap);
			logger.debug("### userDataInfo ### " + user.getUserDataInfo().toString());
			logger.debug("### openPrize ### " + openPrize.toString());
			return openPrize;
		} catch (Exception e) {
			logger.error("", e);
		}
		return null;
	}

```

## 操作奖池

```java
	/**
	 * 操作奖池
	 * 
	 * @param betCoin
	 *            押注总金额
	 * @param userDataInfo
	 *            用户数据
	 * @param prizeMap
	 *            押注对应奖励
	 * @return 当前操作累加后的用户数据
	 */
	public static synchronized OpenPrize addPoolAndCalcPrize(long betCoin, UserDataInfo userDataInfo,
			Map<BetType, Double> prizeMap) {
		// 总押注
		final long totalCoin = betCoin;
		// 增加奖池
		Pool.operPool(totalCoin);

		// 数据记录
		userDataInfo.addAddPool(totalCoin);
		userDataInfo.addTotalBet(totalCoin);

		// 根据当前奖池踢出高于当前奖池奖励
		final Map<BetType, Double> openPrizeMap = new HashMap<>(prizeMap);
		for (Entry<BetType, Double> entry : prizeMap.entrySet()) {
			BetType betType = entry.getKey();
			Double openPrize = entry.getValue();
			if (!Pool.checkPool(openPrize+2))
				openPrizeMap.remove(betType);
		}
		OpenPrize openPrize = null;
		// 获取开奖结果
		if (!openPrizeMap.isEmpty())
			openPrize = calcPrize(openPrizeMap);
		else
			log.error(JsonUtils.entity2Json(prizeMap) + " ### " + PoolManager.getPool(),
					"开奖错误[奖池不够开不出奖励] ### 押注信息 ### 操作奖池");
		if (openPrize != null) {
			// 扣除奖池 每次扣除增加2个gala
			if (openPrize.getPrize() != 0d)
				Pool.operPool(-(openPrize.getPrize() + 2));

			// 计算开奖费用
			final Double prize = openPrize.getPrize();
			// jp EAT_PRIZE_JP= .01
			final Double eatJp = prize * SysConstant.EAT_PRIZE_JP;
			// 平台运营 EAT_PRIZE_YUNYING= .003
			final Double eatPrizeYunying = prize * SysConstant.EAT_PRIZE_YUNYING;
			// 基金会 EAT_PRIZE_JIJINHUI= .001
			final Double eatPrizeJijinhui = prize * SysConstant.EAT_PRIZE_JIJINHUI;
			// 星球主 EAT_PRIZE_XINGQIUZU= .006
			final Double eatPrizeXingqiu = prize * SysConstant.EAT_PRIZE_XINGQIUZU;
			// 得到实际奖励
			final Double realPrize = prize - eatPrizeYunying - eatPrizeJijinhui - eatPrizeXingqiu - eatJp;

			// 数据记录
			userDataInfo.addEatPrizeYunying(eatPrizeYunying);
			userDataInfo.addEatPrizeJijinhui(eatPrizeJijinhui);
			userDataInfo.addEatPrizeXingqiu(eatPrizeXingqiu);
			userDataInfo.addEatJp(eatJp);

			PoolJp.operPool(eatJp);
			final PoolInfo poolInfo = poolInfoManager.getPoolInfo();
			poolInfo.addEatPrizeYunying(eatPrizeYunying);
			poolInfo.addEatPrizeJijinhui(eatPrizeJijinhui);
			poolInfo.addEatPrizeXingqiu(eatPrizeXingqiu);

			openPrize.setPrize(realPrize);
		}

		return openPrize;
	}

```

## 计算开奖结果

```java
	/**
	 * 计算开奖结果 
	 * 
	 * @param prizeMap
	 *            每个押注获得奖励
	 * @return 开奖结果
	 */
	private static OpenPrize calcPrize(Map<BetType, Double> prizeMap) {
		try {
			final List<BetType> list = new ArrayList<>(prizeMap.keySet());
			String rateStr = "";
			for (BetType betType : list)
				rateStr += betType.getRate() + ",";
			rateStr = rateStr.substring(0, rateStr.length() - 1);
			int index = ProbUtils.randomStr(rateStr);
			BetType betTypeRes = list.get(index);
			Double prize = prizeMap.get(betTypeRes);

			return new OpenPrize(betTypeRes, prize, prize);
		} catch (Exception e) {
			log.error("", e);
		}
		return null;
	}
```

## 发送奖励

```java
	/**
	 * 发奖
	 */
	public void sendPrize(String everyId) {
		try {
			if (openPrize == null)
				return;
			// 奖励结果
			Double addCoin = openPrize.getPrize();
			final PrizeManager prizeManager = baseInvoke.getPrizeManager();
			// 调用合约增加gala
			prizeManager.addPrize(user, OperType.OPEN_PRIZE, new IdWithCnt(PrizeType.GALA.getId(), addCoin.toString()));
			// 历史数据 大于50重置
			if (openPrizeHis.size() >= SysConstant.OPEN_HIS_CNT)
				openPrizeHis.clear();
			openPrizeHis.addFirst(openPrize.getBetTypeRes().getId());
			// 全局发送用户获奖消息
			if (openPrize.getPrize() != 0)
				SenderMsg.allServerMsgUserSay(user, JsonUtils.entity2Json(new UserSayDto(user, openPrize)));
			// 发送历史消息
			SenderMsg.userMsg(user, Protocol.update_openHis(JsonUtils.list2Json(openPrizeHis)));
			// 数据记录
			final UserBetInfoManager userBetInfoManager = baseInvoke.getUserBetInfoManager();
			UserBetInfo userBetInfo = new UserBetInfo(user, openPrize.getBetTypeRes().getId(), openPrize.getPrize(),
					totalBetCoin, everyId);
			for (Entry<BetType, Integer> entry : curBetMap.entrySet()) {
				BetType betType = entry.getKey();
				Integer value = entry.getValue();
				ReflectUtils.setMethod(userBetInfo, betType.getId(), value);
			}
			userBetInfoManager.saveOrUpdateUserBetInfo(userBetInfo);

			saveTotalBet();
		} catch (Exception e) {
			logger.error("", e);
		}
	}
```
