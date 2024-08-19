## [Low-1] User can be in loss if they issue after the default.

The `RToken::issue(...)` function can be called when the `BasketHandler.isReady()` is true. But When a collateral is defaulted, and new primary basket is set, but the `BackingManger::rebalance(...)` is not called yet, the `issue(...)` function can still be called as the `BasketHandler.isReady()` will start to return true when the `warmup` period is passed after changing the basket. It does not matter if the `rToken` is fully collateralized or not. So in that case, if the user issues tokens at that time, they will have to deposit the token in the new basket ratio as before the `rebalance(...)`, the `RToken::basketNeeded` is not changed and the ratio will be `1:1` with rToken issued (only true if the default is first time or no revenue is distributed). But they cannot redeem the tokens as in that case, the fully collateralized check will be done. Now, when the `rebalance(...)` is called, it will for sure reduce the `RToken::basketNeeded` and now if the user tries to redeem, they will get less tokens than what they deposited when the tokens were issued.

**Here is a test for PoC:**

```js
            it.only("Asset can be issued when basket imbalanced", async ()=>{
              // set the basket to have only token0
              await basketHandler.connect(owner).setPrimeBasket([token0.address], [fp('1')]);

              // refresh the basket
              await basketHandler.refreshBasket();

              // pass the warmup period
              await advanceTime(config.warmupPeriod.toString());

              // register backup collateal token1
              await assetRegistry.connect(owner).register(backupCollateral1.address);

              // add backing tokens to the basket
              await basketHandler.connect(owner).setBackupConfig(ethers.utils.formatBytes32String('USD'), bn(1), [
                  await backupCollateral1.erc20()
              ]);

              // alice issue some tokens
              await rToken.connect(alice).issue(bn('100e18'));

              // backing manager balance should be equal to initialBal
              // expect(await token0.balanceOf(backingManager.address)).to.be.eql(initialBal);

              // set the token0 price to default
              await setOraclePrice(collateral0.address, bn('0.5e8'));

              // refresh the assetRegistry
              await assetRegistry.refresh();

              // basketHandler should be iffy
              expect(await basketHandler.status()).to.be.eql(CollateralStatus.IFFY);

              // skip time to make the basket default
              await advanceTime(await collateral0.delayUntilDefault());

              // basket should be defaulted
              expect(await basketHandler.status()).to.be.eql(CollateralStatus.DISABLED);

              // skip warmup period
              await advanceTime(config.warmupPeriod.toString());

              // refresh the basket
              await basketHandler.connect(alice).refreshBasket();

              // assetRegistry should be refreshed
              await assetRegistry.refresh();

              // skip warmup period
              await advanceTime(config.warmupPeriod.toString());

              // basketHandler should be SOUND
              expect(await basketHandler.status()).to.be.eql(CollateralStatus.SOUND);

              // // alice issue tokens here:
              await rToken.connect(alice).issue(bn('100e18'));

              console.log("balance of RToken alice: ", await rToken.balanceOf(alice.address));
              console.log("balance of collateral0: ", await backupToken1.balanceOf(backingManager.address));
              console.log("Basket Needed: ", await rToken.basketsNeeded());

              // skip the trading delay
              await advanceTime(config.tradingDelay.toString());
              
              // rebalance
              await backingManager.rebalance(TradeKind.DUTCH_AUCTION);

              // skip the time to make the trade to be happen at the worst price
              await advanceTime(config.dutchAuctionLength.toNumber() - 10);

              // get the trade contract
              const trade = await ethers.getContractAt(
                'DutchTrade',
                await backingManager.trades(token0.address)
              )

              // bob places the bid
              await (await ethers.getContractAt("ERC20Mock", await backupCollateral1.erc20())).connect(bob).approve(trade.address, bn("10000e18"));
              await trade.connect(bob).bid()

              // get the basket ranges
              console.log(await basketHandler.basketsHeldBy(backingManager.address));

              // advance time to make the basket ready
              await advanceTime(config.warmupPeriod.toString());

              // basket should be fully collateralized now:
              expect(await basketHandler.fullyCollateralized()).to.be.true;

              console.log("new basket Needed",await rToken.basketsNeeded());

              // alice's balances of token before the redeem
              const aliceBalBeforetoken1 = await backupToken1.balanceOf(alice.address);

              // alice redeems the rToken
              await rToken.connect(alice).redeem(bn("100e18"));

              // alice's balances of token after the redeem
              const aliceBalAftertoken0 = await token0.balanceOf(alice.address);
              const aliceBalAftertoken1 = await backupToken1.balanceOf(alice.address);

              console.log(aliceBalBeforetoken1.sub(aliceBalAftertoken1).toString());

          })
```

**Mitigiation:**

This is actually a good thing for the protocol as it will help in increasing the `rToken::basketNeeded` but from the stand point of the user, this is actually a quite harmful thing. This issue can be solved on UI by showing the warning message that the basket is in rebalance mode and the new tokens should not be issued in that case. But it could still be problematic if the reserve contracts are integrated into new project. So there should atleast be warning in the function natspac about the pros and cons of the function.


## [Low-2] A deprecated asset can be added again in the Basket.

If an `RToken` is using `AssetPluginRegistry`, then when a new asset is registered in the `AssetRegistry`, a call will be made to the `AssetPluginRegistry` to see if the asset is deprecated or not. This plugin registry is helpful in case a malicious plugin is added to the contract. But When a new primary Basket is set in the `BasketHandler`, it does not check if the basket contains a deprecated asset from `AssetPluginRegsitry`. That means a deprecated asset can be added to the primary and refrence basket. This could be quite harmful in case the owner is governance contract and a malicious proposal go through it. 

**Here is a test For Poc:**

```js
            it("deprecated asset can be added to the basket", async()=>{
                // deploy asset plugin registry
                const assetPluginRegistry = await ethers.getContractFactory('AssetPluginRegistry')
                const versionRegistry = await ethers.getContractFactory('VersionRegistry');
                const roleRegistry = await ethers.getContractFactory('RoleRegistry');
                const roleRegistryInstance = await roleRegistry.deploy();
                const versionRegistryInstance = await versionRegistry.deploy(roleRegistryInstance.address);
                const assetPluginRegistryInstance = await assetPluginRegistry.deploy(versionRegistryInstance.address);

                // set asset registry in main
                await main.setAssetPluginRegistry(assetPluginRegistryInstance.address);

                // set the version in the version registry
                await versionRegistryInstance.connect(owner).registerVersion(deployer.address)

                // encode the version string using ABI encoding
                const encodedVersion = ethers.utils.solidityPack(['string'], [await deployer.version()]);

                // compute the keccak256 hash of the encoded string
                const versionHash = ethers.utils.keccak256(encodedVersion);
                
                // register the assets
                await assetPluginRegistryInstance.connect(owner).registerAsset(token0.address, [versionHash]);
                await assetPluginRegistryInstance.connect(owner).registerAsset(token1.address, [versionHash]);
                await assetPluginRegistryInstance.connect(owner).registerAsset(token2.address, [versionHash]);
                await assetPluginRegistryInstance.connect(owner).registerAsset(token3.address, [versionHash]);

                // update the basket to only have token0
                await basketHandler.connect(owner).setPrimeBasket([token0.address], [fp('1')])

                // refresh the prime basket
                await basketHandler.connect(owner).refreshBasket();
                
                // deprecate asset token1
                await assetPluginRegistryInstance.connect(owner).deprecateAsset(token1.address)

                // update the basket to have token1
                await basketHandler.connect(owner).setPrimeBasket([token1.address], [fp('1')])

                // refresh the prime basket
                await basketHandler.connect(owner).refreshBasket();
            })
```

**Mitigation:**

Add checks to ensure that the asset is not deprecated.

## [Low-3] Rewards distribution can be front-runned with RSR token deposit in `StRSR` in order to receive the same amount of rewards (or more) as other but not bearing any risk of the defaults from the beginning.

When new RSR rewards are distributed to the StRSR contract, The `stakeRSR` is increased with the value of rewards amount since the last `payoutLastPaid`:

```solidity
    function _payoutRewards() internal {
        if (block.timestamp < payoutLastPaid + 1) return;
        uint48 numPeriods = uint48(block.timestamp) - payoutLastPaid;

        uint192 initRate = exchangeRate();
        uint256 payout;

        // Do an actual payout if and only if enough RSR is staked!
        if (totalStakes >= FIX_ONE) {
            // Paying out the ratio r, N times, equals paying out the ratio (1 - (1-r)^N) 1 time.
            // Apply payout to RSR backing
            // payoutRatio: D18 = FIX_ONE: D18 - FixLib.powu(): D18
            // Both uses of uint192(-) are fine, as it's equivalent to FixLib.sub().
@>            uint192 payoutRatio = FIX_ONE - FixLib.powu(FIX_ONE - rewardRatio, numPeriods);

            // payout: {qRSR} = D18{1} * {qRSR} / D18
@>            payout = (payoutRatio * rsrRewardsAtLastPayout) / FIX_ONE;
@>            stakeRSR += payout;
        }
```

But it could be really unfair with the existing depositors. Because since the `stakeRSR` is increased directly, the `stakeRate` will decrease. Which means more `RSR` can be received for `stRSR`. And hence the following can happen:

1. Alice deposits at `T-0`, `100e18` RSR.
2. At time `T-100`, let's say there were new rewards distributed.
3. Bob saw that and he decide to front run the rewards distribution.
4. Bob deposits `100e18` frontrunning the rewards distribution.
5. Now let's say he waited for a week to get rewards accumulated as they are distributed over time.
6. Now if the  alice and bob withdraw their tokens at the same time, both of them will get the same amount of rewards. 

Even though he didn't stake his token from the beginning, he received the same amount of rewards. On the other hand, alice deposited from the beginning and had to bear the risk of losing her tokens.

Bob can even deposit big amount of tokens and could receive large portion of the rewards.

This could be more unfair if `unstakingDelay` is less very less. The acceptable range is between `2 mins` to `1 year`. so if it is set to lets say `15 mins` a user after accumulating rewards can withdraw quickly.

**Proof Of Concept:**

Test demonstrates the scenario where bob recieved bigger share of the rewards by depositing bigger amount.

```js
            it.only("Staker can stake just before the rewards are distributed and earn equal or more rewards", async()=>{
              // setupup new basket with only token0
              await basketHandler.connect(owner).setPrimeBasket([token0.address], [fp('1')]);

              // refresh the basket
              await basketHandler.refreshBasket();

              // pass the warmup period
              await advanceTime(config.warmupPeriod.toString());

              // alice issue some tokens
              await rToken.connect(alice).issue(bn('100e18'));

              // backing manager balance should be equal to initialBal. And should be fully collateralized
              expect(await basketHandler.fullyCollateralized()).to.be.true;
              expect(await basketHandler.status()).to.be.eql(CollateralStatus.SOUND);
              expect(await basketHandler.isReady()).to.be.true;


              // alice stakes some RSR
              await stRSR.connect(alice).stake(bn('100e18'));
              
              // advance 1 week
              await advanceTime(604800);

              // bob stakes some RSR just before the new rewards
              await stRSR.connect(bob).stake(bn('10000e18'));

              // new rewards came in the form of RSR
              await rsr.connect(owner).mint(stRSR.address, bn('100e18'));

              // call the payout Rewards function
              await stRSR.connect(owner).payoutRewards();

              // advance time
              await advanceTime(604800);
              
              // after accruing some rewards, bob and alice unstake their RSR
              // bob unstakes
              await stRSR.connect(bob).unstake(bn('10000e18'));

              // alice unstakes
              await stRSR.connect(alice).unstake(bn('100e18'));

              // pass the delay period
              await advanceTime(config.unstakingDelay.toString());

              // bob and alice's RSR balance before
              const bobBalBefore = await rsr.balanceOf(bob.address);
              const aliceBalBefore = await rsr.balanceOf(alice.address);

              // alice and bob withdraws their RSR
              await stRSR.connect(alice).withdraw(alice.address, 1);
              await stRSR.connect(bob).withdraw(bob.address, 1);

              // bob and alice's RSR balance after
              const bobBalAfter = await rsr.balanceOf(bob.address);
              const aliceBalAfter = await rsr.balanceOf(alice.address);

              console.log("Bob's RSR balance: ", bobBalAfter.sub(bobBalBefore).toString());
              console.log("Alice's RSR balance: ", aliceBalAfter.sub(aliceBalBefore).toString());
              console.log("Rewards Bob Earned: ", bobBalAfter.sub(bn("100000e18")).toString());
              console.log("Rewards Alice Earned: ", aliceBalAfter.sub(bn("100000e18")).toString());
            })

```

**Mitigation:**

To mitigate this The following strategy could be used:
keep the new stakes in a pending queue and do not add them directly to the stakes. And introduce a new delay period that this pending stake have to pass in order to be eligible for the rewards and get added to the `stakeRSR`. 
Or use some other strategy.


## [Low-4] `BackingManager::rebalance(...)` can be called by anyone which can cause loss of value in case of Market shifts.

Currently the only conditions that should met in order to call `BackingManager::rebalance(...)` is that new basket should be ready and there should not be any existing trade for a `TradeKind`. As long as these are met, anyone can come and call the rebalance function:

```solidity
    function rebalance(TradeKind kind) external nonReentrant {
        requireNotTradingPausedOrFrozen();


        // == Refresh ==
        assetRegistry.refresh();


        // DoS prevention:
        // unless caller is self, require that the next auction is not in same block
        require(
@>            _msgSender() == address(this) || tradeEnd[kind] < block.timestamp,
            "already rebalancing"
        );


```

GitHub: [107-119](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L107C1-L119C1)

But this could be very dangerous thing from the perspective of the protocol. Because let's say market is in bad condition and need some time in order to recover. But due to no restriction in the function, anyone can call it and make the trade even at the bad market prices causing the loss of value to the protocol.

Currently there is a trading pause that could be applied to prevent this from happening, but trading pause is very crucial thing as it pauses the most of the functionality of the protocol. Also After having a communication with the protocol team, they replied that trading pause is there mostly for the cases when  there is some kind of bug in the protocol or something serious like this happens. 

**Mitigation**

There should be some kind of restriction to the function so that we don't have to pause the whole trading and could get rid of the issue.