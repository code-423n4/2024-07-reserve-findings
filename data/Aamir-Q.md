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
