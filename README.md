[TOC]



# Start and Drawan exploration card

```java
	/**
	 * Select a planet
	 * @param data 
	 * @param user 
	 */
	private void bet(Map<String, String> data, User user) {
		try {
			int status = 0;
			String desc = "";
			// if(user.getUserGameTimeM().getBetStageType().isBetStage())
			// user.getUserGameM().userBetRecord(data);
			// else
			// status= -1;
			// 0succeed -1no bet placed
			// check if all in line with betting rules
			Boolean isDataSuc = user.getUserGameM().userBetRecord(data);
			if (!isDataSuc) {
				status = -99;
				desc = "Betting failed,wrong betting";
			}
			// check if the betting amount is 0
			if (status == 0 && user.getUserGameM().getTotalBetCoin() == 0) {
				status = -1;
				desc = "Betting failed,no bet placed for now";
			}
			// Get the total amount of bet placed on planets
			Double totalBet = user.getUserGameM().getTotalBetCoin().doubleValue();
			if (status == 0 && !user.galaOut(totalBet)) {
				status = -2;
				desc = "Betting failed,no bet placed for now";
			}
			String prize = "{}";
			if (status == 0) {
				// Calculate bonus
				OpenPrizeClient openPrizeClient = user.getUserGameTimeM().openPrize();
				prize = JsonUtils.entity2Json(openPrizeClient);
			}
			SenderMsg.userMsg(user, Protocol.bet(status, desc, prize));
			if (status == 0) {
				if (totalBet < 1000)
					return;
				// get an exploration card odds EAT_BET= .1 EAT_PRIZE_YUNYING= .003 HARDRATE= .3
				double rate = totalBet * (SysConstant.EAT_BET + SysConstant.EAT_PRIZE_YUNYING) * .1
						* SysConstant.HARDRATE / 500;
				if (rate > .5)
					rate = .5;
				double random = RandomUtils.nextDouble();
				boolean flag = rate > random;
				log.info(Msg.printMsgInfo(user, "get an exploration card odds[" + rate + "], randomized rate[" + random + "], whether sent to winners[" + flag + "]",
						"Check if one drop an exploration card?"));
				if (flag) {
					UserBalanceM userBalanceM = user.getUserBalanceM();
					String addr = userBalanceM.getAddr();
					String itemId = ItemType.EXPLORE_CARD.getItemId();
					String amount = "1";
					String userId = userBalanceM.getUserId();
					// draw
					baseInvoke.getHttpInterManager().addItemPost(addr, userId, itemId, amount);
											baseInvoke.getPoolInfoManager().getPoolInfo().itemExploreCardCntAdd(Integer.valueOf(amount));
				}
			}
		} catch (Exception e) {
			log.error("", e);
		}
	}
```

# Transaction history and total amount

```java

	/**
	 * history of bet placed by users and total betting amount
	 * 
	 * @param data
	 *            bet data
	 */
	public Boolean userBetRecord(Map<String, String> data) {
		try {
			String tempBetData = "";
			// get total amount of bet
			for (Entry<BetType, Integer> entry : curBetMap.entrySet()) {
				BetType key = entry.getKey();
				Integer betCoin = data.get(key.getId()) == null ? 0 : Integer.parseInt(data.get(key.getId()));
				curBetMap.put(key, betCoin);
				totalBetCoin += betCoin;
				tempBetData += key.getId() + "-" + betCoin + ",";
			}
			logger.debug("### totalBetCoin: " + totalBetCoin);
			logger.debug("### curBetMap: " + tempBetData.substring(0, tempBetData.length() - 1));
			// get minimal betting amount
			final int betCoinMin = BetCoinType.findBetCoinMin();
			// check if betting rules are met
			if (totalBetCoin % betCoinMin != 0) {
				logger.error(JsonUtils.entity2Json(data), "wrong betting data[user’s custom port,bet canceled]  ### parameter");
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

# Deduct gala

```java
	/**
	 * deduct gala
	 * @param gala 
	 * @return check if it is successfully
	 */
	public boolean galaOut(Double gala) {
		return userBalanceM.galaOut(gala);
	}

	/**
	 * http deduct gala
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
			// get transaction hash
			hash = futureGalaOut.get();
		} catch (InterruptedException e) {
			log.error("", e);
		} catch (ExecutionException e) {
			log.error("", e);
		}
		if(StringUtils.isEmpty(hash))
			return verify;
		SenderMsg.userMsg(user, Protocol.oper_hash(hash, GalaOperType.OUT));
 		// confirm transaction hash
		verify= contractManager.getSmartEventCodeByHash(hash);
		return verify;
	}
	
```

# “Start” button

```java
	/**
	 * tag in each round of “Start”
	 * 
	 * @return check out the result
	 */
	public final OpenPrizeClient openPrize() {
		// each round mark
		everyId = IdGenerator.generate();
		// check out the result
		OpenPrize openPrize = user.getUserGameM().userBet();
		if (openPrize == null) {
			return new OpenPrizeClient(null);
		}
		// generate the data required by client-side
		OpenPrizeClient openPrizeClient = new OpenPrizeClient(openPrize);
		// distribute the bonus and record data
		user.getUserGameM().sendPrize(everyId);
		// Reset data
		user.getUserGameM().resetData();

		return openPrizeClient;
	}

```

## To calculate the prize pool and cost

```java
	/**
	 * To calculate the prize pool and cost
	 */
	public OpenPrize userBet() {
		try {
			// calculate Gala reward for each round
			for (Entry<BetType, Integer> entry : curBetMap.entrySet())
				prizeMap.put(entry.getKey(), entry.getValue() * entry.getKey().getMul());
			// winner set based on prize pool
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

## How prize pool works

```java
	/**
	 * the way price pool works
	 * 
	 * @param betCoin
	 *            total amount of Gala
	 * @param userDataInfo
	 *            user data
	 * @param prizeMap
	 *            prize mapping 
	 * @return 
	 */
	public static synchronized OpenPrize addPoolAndCalcPrize(long betCoin, UserDataInfo userDataInfo,
			Map<BetType, Double> prizeMap) {
		// total amount of Gala
		final long totalCoin = betCoin;
		// expand prize pool
		Pool.operPool(totalCoin);

		// data record
		userDataInfo.addAddPool(totalCoin);
		userDataInfo.addTotalBet(totalCoin);

		// remove prize higher than current prize pool
		final Map<BetType, Double> openPrizeMap = new HashMap<>(prizeMap);
		for (Entry<BetType, Double> entry : prizeMap.entrySet()) {
			BetType betType = entry.getKey();
			Double openPrize = entry.getValue();
			if (!Pool.checkPool(openPrize+2))
				openPrizeMap.remove(betType);
		}
		OpenPrize openPrize = null;
		// access the result
		if (!openPrizeMap.isEmpty())
			openPrize = calcPrize(openPrizeMap);
		else
			log.error(JsonUtils.entity2Json(prizeMap) + " ### " + PoolManager.getPool(),
		"Start” error[more Gala needed to be staked into the prize pool] ### bet info ### assess prize pool")
		if (openPrize != null) {
			// 2 gala consumed per “start”
			if (openPrize.getPrize() != 0d)
				Pool.operPool(-(openPrize.getPrize() + 2));

			// calculate reward 
			final Double prize = openPrize.getPrize();
			// jp EAT_PRIZE_JP= .01
			final Double eatJp = prize * SysConstant.EAT_PRIZE_JP;
			// Platform operation  EAT_PRIZE_YUNYING= .003
			final Double eatPrizeYunying = prize * SysConstant.EAT_PRIZE_YUNYING;
			// Foundation EAT_PRIZE_JIJINHUI= .001
			final Double eatPrizeJijinhui = prize * SysConstant.EAT_PRIZE_JIJINHUI;
			// CG planet owners EAT_PRIZE_XINGQIUZU= .006
			final Double eatPrizeXingqiu = prize * SysConstant.EAT_PRIZE_XINGQIUZU;
			// get reward
			final Double realPrize = prize - eatPrizeYunying - eatPrizeJijinhui - eatPrizeXingqiu - eatJp;

			// Data record
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

## Calculate award

```java
	/**
	 * Calculate the amount of prize 
	 * 
	 * @param prizeMap
	 *            award per “Start”
	 * @return result
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

## issue reward

```java
	/**
	 * give award
	 */
	public void sendPrize(String everyId) {
		try {
			if (openPrize == null)
				return;
			// result
			Double addCoin = openPrize.getPrize();
			final PrizeManager prizeManager = baseInvoke.getPrizeManager();
			// increase Gala by call of smart contract
			prizeManager.addPrize(user, OperType.OPEN_PRIZE, new IdWithCnt(PrizeType.GALA.getId(), addCoin.toString()));
			// reset when > 50
			if (openPrizeHis.size() >= SysConstant.OPEN_HIS_CNT)
				openPrizeHis.clear();
			openPrizeHis.addFirst(openPrize.getBetTypeRes().getId());
			// inform player award      
			if (openPrize.getPrize() != 0)
				SenderMsg.allServerMsgUserSay(user, JsonUtils.entity2Json(new UserSayDto(user, openPrize)));
			// send past records
			SenderMsg.userMsg(user, Protocol.update_openHis(JsonUtils.list2Json(openPrizeHis)));
			// data record
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
