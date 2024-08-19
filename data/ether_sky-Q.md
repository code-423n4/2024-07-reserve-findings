# [L-1] The max sell amount should be calculated before checking the minimum trade volume

In `trades`, there is a `minimum trade volume` check on `line 54` to prevent `dust trades`, and a `max sell amount` check on `line 69` to avoid large `trades`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/TradeLib.sol#L54-L59
```
function prepareTradeSell(
    TradeInfo memory trade,
    uint192 minTradeVolume,
    uint192 maxTradeSlippage
) internal view returns (bool notDust, TradeRequest memory req) {}

54:    notDust = isEnoughToSell(
        trade.sell,
        trade.sellAmount,
        trade.prices.sellLow,
        minTradeVolume
    );

    uint192 s = trade.sellAmount;
    if (trade.prices.sellHigh != FIX_MAX) {
        uint192 maxSell = maxTradeSize(trade.sell, trade.buy, trade.prices.sellHigh);
        require(maxSell > 1, "trade sizing error");
69:        if (s > maxSell) s = maxSell;
    } else {
        require(trade.prices.sellLow == 0, "trade pricing error");
    }
}
```
However, the amount that passes the `minimum trade volume` check might be reduced by the `max sell amount` check, making it no longer satisfy the `minimum trade volume`. 
This is because the `minimum trade volume` check is only related to the `sell` token, while the `max sell amount` also is related to the buy token.
To avoid this issue, it's better to reverse the order of these checks.

# [L-2] The issuance of rToken may be reverted due to an incorrect approval amount

When issuing `rTokens`, there is no `slippage` check in the functions. 
This means the transaction could be executed after some time, potentially requiring a larger deposit amount than the depositor expected. 
If depositors approve the enough amount, a larger amount might be deducted for the specified `rToken` amount.

To prevent this, the code comments suggest using the `quote` function to determine the exact `approval` amount before issuing `rTokens`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L99-L100
```
/// Issue an RToken on the current basket, to a particular recipient
/// Do no use inifite approvals.  Instead, use BasketHandler.quote() to determine the amount
///     of backing tokens to approve.

function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {
}
```
However, these quoted amounts might be lower than the actual amounts needed because they don't account for the `issuance premium`. 
The `issuance premium` is only applied when the `assets` are refreshed (`line 368`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L368
```
function issuancePremium(ICollateral coll) public view returns (uint192) {
368:    if (!enableIssuancePremium || coll.lastSave() != block.timestamp) return FIX_ONE;

    try coll.savedPegPrice() returns (uint192 pegPrice) {
        if (pegPrice == 0) return FIX_ONE;
        uint192 targetPerRef = coll.targetPerRef(); // {target/ref}
        if (pegPrice >= targetPerRef) return FIX_ONE;
        return targetPerRef.safeDiv(pegPrice, CEIL);
    } catch {
        return FIX_ONE;
    }
}
```

When depositors use the `quote view` function, there's no guarantee that all `assets` have been correctly refreshed. 
If they haven't, the quoted amounts may be lower than required. 
If depositors only approve these lower amounts as suggested, the `issuance` will be reverted due to the `issuance premium` being applied when the tokens are refreshed in the `issueTo` function (`line 110`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L110
```
function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {
    require(amount != 0, "Cannot issue zero");

    // == Refresh ==
  
110:    assetRegistry.refresh();
}
```

To solve this, consider adding a function that combines the `asset` refreshing process with the `quote` functionality.

# [L-3] The `lastCollateralized` should be initialized as 1

In the `BasketHandler`, `lastCollateralized` is currently initialized as `0`, and this value is used during `custom redemption`. 
The first `basket ID` is `1`, not `0` (`line 666`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L666
```
function _switchBasket() private {
    disabled = true;
    bool success = _newBasket.nextBasket(_targetNames, config, assetRegistry);
    
    if (success) {
666:        nonce += 1;
        basket.setFrom(_newBasket);
        basketHistory[nonce].setFrom(_newBasket);
        timestamp = uint48(block.timestamp);
        disabled = false;
    }
}
```
Users might mistakenly select `0` as the `basketNonce` in `custom redemption`, which isn't checked (`line 537`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L537
```
function quoteCustomRedemption(
    uint48[] memory basketNonces,
    uint192[] memory portions,
    uint192 amount
) external view returns (address[] memory erc20s, uint256[] memory quantities) {
    require(basketNonces.length == portions.length, "invalid lengths");
  
    IERC20[] memory erc20sAll = new IERC20[](assetRegistry.size());
    ICollateral[] memory collsAll = new ICollateral[](erc20sAll.length);
    uint192[] memory refAmtsAll = new uint192[](erc20sAll.length);
  
    uint256 len; // length of return arrays

    for (uint48 i = 0; i < basketNonces.length; ++i) {
        require(
537:            basketNonces[i] >= lastCollateralized && basketNonces[i] <= nonce,
            "invalid basketNonce"
        );
    }
}
```
As a result, users would withdraw nothing from `basket 0`.

# [L-4] Rewards should be distributed before calculating the leaked amount in the StRSR contract

When withdrawing staked `RSR` from the `StRSR`, we calculate the `leaked amount`.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L336
```
function withdraw(address account, uint256 endId) external {
336:    leakyRefresh(rsrAmount);
    IERC20Upgradeable(address(rsr)).safeTransfer(account, rsrAmount);
}
```
If this amount exceeds the defined limit, we manually `refresh` the `assets` (`line 716`). 
This calculation currently only considers the staked and unstaked amounts, as seen in `line 704`, without including the `rewards`.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L704-L717
```
function leakyRefresh(uint256 rsrWithdrawal) private {
    uint48 lastRefresh = assetRegistry.lastRefresh(); // {s}
704:    uint256 totalRSR = stakeRSR + draftRSR + rsrWithdrawal; // {qRSR}
    uint192 withdrawal = _safeWrap((rsrWithdrawal * FIX_ONE + totalRSR - 1) / totalRSR); // {1}
  
    leaked = lastWithdrawRefresh != lastRefresh ? withdrawal : leaked + withdrawal;
    lastWithdrawRefresh = lastRefresh;
    if (leaked > withdrawalLeak) {
        leaked = 0;
        lastWithdrawRefresh = uint48(block.timestamp);
716:        assetRegistry.refresh();
    }
}
```

While it might be acceptable if this is a design choice, the issue is that `rewards` accumulated since the last update are also not included because the `payoutRewards` function isn't called during withdrawal. 
As a result, the `stakeRSR` value used in `line 704` is inaccurate.

Call the `payoutRewards` function within the `withdraw` function.

# [L-5] The warm-up period is not considered when switching from one valid basket to a new one

Typically, a `basket` is considered ready for operations like `rebalance` and `forwardRevenue` once the `warm-up` period has passed after the `basket`'s status becomes `SOUND`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L331-L335
```
function isReady() external view returns (bool) {
    return
        status() == CollateralStatus.SOUND &&
        (block.timestamp >= lastStatusTimestamp + warmupPeriod);
}
```
This `warm-up` period acts as a guarantee period for the `basket`.

However, when the owner switches from a valid `basket` to a new one, the `warm-up` period does not apply, and the new `basket` is considered ready immediately. 
It's unclear whether this is an intentional design choice.

# [L-6] The fees should be distributed before changing the recipient in the DAOFeeRegistry

The `DAO fees` are sent to the registered `recipient` (`line 180`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Distributor.sol#L183-L187
```
function distribute(IERC20 erc20, uint256 amount) external {
    DAOFeeRegistry daoFeeRegistry = main.daoFeeRegistry();
    if (address(daoFeeRegistry) != address(0)) {
        if (totalShares > paidOutShares) {
180:            (address recipient, , ) = main.daoFeeRegistry().getFeeDetails(address(rToken));
  
            if (recipient != address(0)) {
                IERC20Upgradeable(address(erc20)).safeTransferFrom(
                    caller,
                    recipient,
                    tokensPerShare * (totalShares - paidOutShares)
                );
            }
        }
    }
}
```
In the `DAOFeeRegistry`, the `owner` can change both the `fee recipient` and the `fee numerator` (`line 52`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/registry/DAOFeeRegistry.sol#L52
```
function setFeeRecipient(address feeRecipient_) external onlyOwner {
	if (feeRecipient_ == address(0)) {
		revert DAOFeeRegistry__InvalidFeeRecipient();
	}
	if (feeRecipient_ == feeRecipient) {
		revert DAOFeeRegistry__FeeRecipientAlreadySet();
	}

52:	feeRecipient = feeRecipient_;
	emit FeeRecipientSet(feeRecipient_);
}
```
Therefore, fees from the `RevenueTrader` should be distributed before making these changes.

# [L-7] Depositors can lose funds in the basket when `reweightable` is enabled

If `reweightable` is allowed in a `basket`, the owner can change the target weight by bypassing the check in `line 237`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L236-L248
```
function _setPrimeBasket(
    IERC20[] calldata erc20s,
    uint192[] memory targetAmts,
    bool disableTargetAmountCheck
) internal {
    requireGovernanceOnly();
    require(erc20s.length != 0 && erc20s.length == targetAmts.length, "invalid lengths");
    requireValidCollArray(erc20s); 

    if (
237:        (!reweightable || (reweightable && !disableTargetAmountCheck)) &&
        config.erc20s.length != 0
    ) {
        BasketLibP1.requireConstantConfigTargets(
            assetRegistry,
            config,
            _targetAmts,
            erc20s,
            targetAmts
        );
    }
}
```

Imagine below scenario:

1. A `basket` with a total target weight of `100` is created.
2. Depositors add funds to this `basket`.
3. The owner switches to a new `basket` with a target weight of `50`.

The `lastCollateralized` value is updated to the current `basket` nonce, as the `basket` is now fully collateralized (`line 189`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L187-L190
```
function trackStatus() public {
	if (reweightable && nonce > lastCollateralized && fullyCollateralized()) {
		emit LastCollateralizedChanged(lastCollateralized, nonce);
189:		lastCollateralized = nonce;
	}
}
```
When depositors withdraw their shares, they will receive only half of their original deposit amount.
Additionally, they cannot perform `custom redemptions` from the old `basket` because `lastCollateralized` was updated to reflect the latest `basket` nonce.

# [L-8] Reward tokens should be claimed before creating trades in the BackingManager and RevenueTraders

Some `collaterals` include `reward` tokens that can be claimed for both the `BackingManager` and `RevenueTraders`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/Trading.sol#L65-L68
```
function claimRewards() external {
    requireNotTradingPausedOrFrozen();
    RewardableLibP1.claimRewards(main.assetRegistry());
}
```
However, `rewards` are not claimed before initiating `trades`. 
As a result, `collaterals` may be sold with these unclaimed `rewards`. 
Additionally, if `reward` tokens are also registered in the `AssetRegistry`, they can be used in `trades`, potentially increasing the `trade` amount by including claimed `rewards`.

Claim `rewards` in the `rebalance` and `manageTokens` functions.

# [L-9] Recollateralization can be reverted if the Dutch auction is disabled for certain tokens

When unexpected results occur with the `Dutch auction`, we disable it for the affected `sell` and `buy` token pairs (`line 163, 167`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Broker.sol#L160-L168
```
function reportViolation() external {
    require(trades[_msgSender()], "unrecognized trade contract");
    ITrade trade = ITrade(_msgSender());
    TradeKind kind = trade.KIND();

    if (kind == TradeKind.BATCH_AUCTION) {
        emit BatchTradeDisabledSet(batchTradeDisabled, true);
        batchTradeDisabled = true;
    } else if (kind == TradeKind.DUTCH_AUCTION) {
        if (DutchTrade(address(trade)).origin() == backingManager) {
            IERC20Metadata sell = trade.sell();
            emit DutchTradeDisabledSet(sell, dutchTradeDisabled[sell], true);
163:            dutchTradeDisabled[sell] = true;
            
            IERC20Metadata buy = trade.buy();
            emit DutchTradeDisabledSet(buy, dutchTradeDisabled[buy], true);
167:            dutchTradeDisabled[buy] = true;
        }
    } 
}
```
However, this can prevent the `recollateralization` process in the `BackingManager`. 
The `BackingManager` selects the optimal next `trade pair` based on a consistent process, meaning the same pair is always chosen unless the `basket` status changes.

If the `Dutch auction` is disabled for one of the selected `trade pairs`, the `recollateralization` will be reverted (`line 274`), and there’s no way to change the `trade pair` if we rely solely on `Dutch auctions`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Broker.sol#L274
```
function newDutchAuction(
    TradeRequest memory req,
    TradePrices memory prices,
    ITrading caller
) private returns (ITrade) {
    require(
274:        !dutchTradeDisabled[req.sell.erc20()] && !dutchTradeDisabled[req.buy.erc20()],
        "dutch auctions disabled for token pair"
    );
}
```
This means we cannot exclude specific tokens from being selected.

To address this, it would be better to add a check to skip tokens for which the `Dutch auction` is disabled.

# [L-10] Tokens that are not registered can become locked in the BackingManager and RevenueTraders

Some unregistered tokens may be generated in the `BackingManager` and `RevenueTraders`.

- Certain `assets` can generate `reward` tokens, which may not be registered in the `AssetRegistry`.
-  `Assets` may become unregistered while a non-zero amount of these `assets` still exists in the `BackingManager`.
   https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/AssetRegistry.sol#L108-L121
```
function unregister(IAsset asset) external governance {
    require(_erc20s.contains(address(asset.erc20())), "no asset to unregister");
    require(assets[asset.erc20()] == asset, "asset not found");

    try basketHandler.quantity{ gas: _reserveGas() }(asset.erc20()) returns (uint192 quantity) {
        if (quantity != 0) basketHandler.disableBasket(); // not an interaction
    } catch {
        basketHandler.disableBasket();
    }

    _erc20s.remove(address(asset.erc20()));
    assets[asset.erc20()] = IAsset(address(0));
    emit AssetUnregistered(asset.erc20(), asset);
}   
```

However, we cannot use unregistered tokens, as all token usage requires registration in the `AssetRegistry` (`line 237`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L237
```
function forwardRevenue(IERC20[] calldata erc20s) external nonReentrant {
    uint256 length = erc20s.length;
    RevenueTotals memory totals = distributor.totals();

    for (uint256 i = 0; i < length; ++i) {
237:        IAsset asset = assetRegistry.toAsset(erc20s[i]);
        ...
    }
}
```
Even though these unregistered tokens might be part of old `baskets`, they won't be used if the `lastCollateralized` value has been updated.

# [L-11] The `lastCollateralized` value can be incorrectly updated in the `trackStatus` function

Old `baskets` can still be used in `custom redemptions`, and the `lastCollateralized` value acts as the `lower limit` for which `baskets` can be used (`line 537`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L537
```
function quoteCustomRedemption(
    uint48[] memory basketNonces,
    uint192[] memory portions,
    uint192 amount
) external view returns (address[] memory erc20s, uint256[] memory quantities) {
    require(basketNonces.length == portions.length, "invalid lengths");
  
    IERC20[] memory erc20sAll = new IERC20[](assetRegistry.size());
    ICollateral[] memory collsAll = new ICollateral[](erc20sAll.length);
    uint192[] memory refAmtsAll = new uint192[](erc20sAll.length);
  
    uint256 len; // length of return arrays

    for (uint48 i = 0; i < basketNonces.length; ++i) {
        require(
537:            basketNonces[i] >= lastCollateralized && basketNonces[i] <= nonce,
            "invalid basketNonce"
        );
}
```
This value is updated in the `trackStatus` function (line 189). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L189
```
function trackStatus() public {
    CollateralStatus currentStatus = status();

    if (currentStatus != lastStatus) {
        emit BasketStatusChanged(lastStatus, currentStatus);
        lastStatus = currentStatus;
        lastStatusTimestamp = uint48(block.timestamp);
    }

    if (reweightable && nonce > lastCollateralized && fullyCollateralized()) {
        emit LastCollateralizedChanged(lastCollateralized, nonce);
189:        lastCollateralized = nonce;
    }
}
```
Once `lastCollateralized` increases, we cannot use old `baskets` with a nonce lower than this value.

The status of `assets` is updated in the `refresh` function. 
However, the `trackStatus` function can be called by anyone without first calling `refresh`. 
If `lastCollateralized` is lower than the latest `basket nonce`, and someone calls `trackStatus` after some time, some assets may have become disabled. 
Normally, calling `refresh` would correctly update these statuses, but without it, the `assets` might return an outdated status (e.g., `SOUND` or `IFFY`), leading to the current `basket` being incorrectly marked as fully collateralized. 
As a result, `lastCollateralized` would incorrectly increase to the latest `basket nonce`.

If depositors want to `redeem`, the `redemption` process first calls the `refresh` function, which may identify that some `assets` are now disabled. 
As a result, they won't be able to proceed with the `redemption` as the current `basket` isn't fully collateralized. 
Additionally, since `lastCollateralized` has already been increased to the latest `basket nonce`, they also can't `redeem` from the `old baskets`.

# [L-12] Rewards should be distributed in the `resetStakes` function

When the `stake rate` becomes unsafe, the `owner` can reset both `staking` and `unstaking`.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L490-L499
```
function resetStakes() external {
    _requireGovernanceOnly();
    require(
        stakeRate <= MIN_SAFE_STAKE_RATE || stakeRate >= MAX_SAFE_STAKE_RATE,
        "rate still safe"
    );
  
    beginEra();
    beginDraftEra();
}
```
However, the `stake rate` is influenced by the amount of `staked RSR`, which can increase due to `rewards` (`line 610, 621~623`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L621-L623
```
function _payoutRewards() internal {
    if (totalStakes >= FIX_ONE) {
        uint192 payoutRatio = FIX_ONE - FixLib.powu(FIX_ONE - rewardRatio, numPeriods);

        payout = (payoutRatio * rsrRewardsAtLastPayout) / FIX_ONE;
610:        stakeRSR += payout;
    }
    payoutLastPaid += numPeriods;
    rsrRewardsAtLastPayout = rsrRewards();

621:    stakeRate = (stakeRSR == 0 || totalStakes == 0)
622:        ? FIX_ONE
623:        : uint192((totalStakes * FIX_ONE_256 + (stakeRSR - 1)) / stakeRSR);
}
```
Distributing these `rewards` could make the `stake rate` safe again, potentially avoiding the need for the `owner` to perform a reset.

Additionally, the `stake rate` and the `draft rate` (`unstaking`) are independent of each other. 
For example, the `draft rate` might have already been reset in the `seizeRSR` function, while the `stake rate` was not (`line 468`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L468
```
function seizeRSR(uint256 rsrAmount) external {
    uint256 draftRSRToTake = (draftRSR * rsrAmount + (rsrBalance - 1)) / rsrBalance;
    draftRSR -= draftRSRToTake;
    seizedRSR += draftRSRToTake;
    if (draftRSR != 0) {
        draftRate = uint192((FIX_ONE_256 * totalDrafts + (draftRSR - 1)) / draftRSR);
    }

    if (draftRSR == 0 || draftRate > MAX_DRAFT_RATE) {
        seizedRSR += draftRSR;
468:        beginDraftEra();
    }
}
```
Over time, `unstaking` may proceed safely, but the `stake rate` could still become unsafe. 
If the owner calls `resetStakes` under these conditions, `withdrawers` may experience unexpected losses.

# [L-13] In the `seizeRSR` function, any excess RSR should remain in the STRSR as rewards

Within the `BackingManager`, `RSR` is selected as a `sell token` only as a last resort. 
If the `BackingManager` lacks sufficient amounts to sell, it calls the `seizeRSR` function of the `STRSR` (`line 161`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L161
```
function rebalance(TradeKind kind) external nonReentrant {

    if (doTrade) {
        IERC20 sellERC20 = req.sell.erc20();

        if (sellERC20 == rsr) {
            uint256 bal = sellERC20.balanceOf(address(this));
161:            if (req.sellAmount > bal) stRSR.seizeRSR(req.sellAmount - bal);
        }
    } else {
        compromiseBasketsNeeded(basketsHeld.bottom);
    }
}
```
This results in losses that affect `stakers`, `unstakers`, and `rewards`. 
If the `stake` or `unstake rates` become too high due to these losses, the `rates` are reset, and the remaining `RSR` is sent to the `BackingManager` (`line 451, 467`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L451
```
function seizeRSR(uint256 rsrAmount) external {
   
    if (stakeRSR == 0 || stakeRate > MAX_STAKE_RATE) {
451:        seizedRSR += stakeRSR;
        beginEra();
    }

    if (draftRSR == 0 || draftRate > MAX_DRAFT_RATE) {
467:        seizedRSR += draftRSR;
        beginDraftEra();
    }

    IERC20Upgradeable(address(rsr)).safeTransfer(caller, seizedRSR);
}
```

However, I believe there’s no need to send this excess `RSR` to the `BackingManager`. 
Instead, it should remain in the `STRSR` as rewards to incentivize future `stakers`. 
Additionally, even if the excess `RSR` stays in the `STRSR`, it can still be used in future `recollateralization`. 
Moreover, any `RSR` sent to the `BackingManager` would eventually return to the `STRSR` through `forwardRevenue` function once the `basket` is fully collateralized.

# [L-14] Only the same kind of trades can happen in the recollateralization

In the `BackingManager`, anyone can call the `rebalance` function to initiate a `trade` of any kind.
However, once a `trade` is in progress, it's not possible to start another one (`line 120`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L120
```
function rebalance(TradeKind kind) external nonReentrant {
    requireNotTradingPausedOrFrozen();
  
    assetRegistry.refresh();

    require(
        _msgSender() == address(this) || tradeEnd[kind] < block.timestamp,
        "already rebalancing"
    );

120:    require(tradesOpen == 0, "trade open");
    require(basketHandler.isReady(), "basket not ready");
    require(block.timestamp >= basketHandler.timestamp() + tradingDelay, "trading delayed");
}
```
This means that only one `trade` can be ongoing in the `BackingManager` at any given time. 
As a result, someone can initiate any type of `trade`, and no other `trades` will impact the `recollateralization` process until that `trade` concludes.

Although this is a design choice, it presents an additional issue. 
If someone initiates a `Dutch auction`, all subsequent `trades` must also be `Dutch auctions` (`line 92`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L92
```
function settleTrade(IERC20 sell) public override(ITrading, TradingP1) returns (ITrade trade) {
    delete tokensOut[sell];
    trade = super.settleTrade(sell); // nonReentrant

    if (_msgSender() == address(trade)) {
92:        try this.rebalance(trade.KIND()) {} catch (bytes memory errData) {
            if (errData.length == 0) revert(); // solhint-disable-line reason-string
        }
    }
}
```
No one can switch to a `Batch auction` unless the `rebalance` function reverts or the `basket` becomes fully collateralized.

# [L-15] Staking is possible, but canceling an unstake is not allowed when the basket is frozen

While the `basket` is frozen, users can still `stake`.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L227-L238
```
function stake(uint256 rsrAmount) public {
    _notZero(rsrAmount);
    _payoutRewards();
 
    address caller = _msgSender();
    mintStakes(caller, rsrAmount);
    
    IERC20Upgradeable(address(rsr)).safeTransferFrom(caller, address(this), rsrAmount);
}
```
But they cannot cancel an `unstake` (`line 347`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L347
```
function cancelUnstake(uint256 endId) external {
347:    _requireNotFrozen();
    address account = _msgSender();
}
```
Since `staking` and canceling an `unstake` are essentially the same process, it's unclear if this behavior is intentional.

# [L-16] The BasketRange can vary depending on how the original top is selected

In `recollateralization`, we calculate the `optimal basket range`, which is crucial as it determines the next `trade pair`. 
If the `original top` exceeds the `basketsNeeded`, we set the `top` to `basketsNeeded` (`line 123`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/RecollateralizationLib.sol#L123
```
function basketRange(TradingContext memory ctx, Registry memory reg)
    internal
    view
    returns (BasketRange memory range)
{
    uint192 basketsNeeded = ctx.rToken.basketsNeeded(); // {BU}
    
    if (ctx.basketsHeld.top > basketsNeeded) {
123:        ctx.basketsHeld.top = basketsNeeded;
    }

    int256 deltaTop; // D18{BU} even though this is int256, it is D18
    for (uint256 i = 0; i < reg.erc20s.length; ++i) {
        if (reg.erc20s[i] == IERC20(address(ctx.rToken))) continue;
        (uint192 low, uint192 high) = reg.assets[i].price(); // {UoA/tok}

        {
            uint192 anchor = ctx.quantities[i].mul(ctx.basketsHeld.top, CEIL);
            if (anchor > ctx.bals[i]) {
                deltaTop -= int256(
168:                    uint256(low.mulDiv(anchor - ctx.bals[i], buPriceHigh, FLOOR))
                );
            } else {
                deltaTop += int256(
176:                    uint256(high.safeMulDiv(ctx.bals[i] - anchor, buPriceLow, CEIL))
                );
            }
        }
    }
    
    if (deltaTop < 0) {
        range.top = ctx.basketsHeld.top - _safeWrap(uint256(-deltaTop));
    } else {
        if (uint256(deltaTop) + ctx.basketsHeld.top > FIX_MAX) range.top = FIX_MAX;
        else range.top = ctx.basketsHeld.top + _safeWrap(uint256(deltaTop));
    }
}
```

For calculating the `optimal top`, we use the following approach:
- If a token's amount exceeds the `top`, we assume that we will sell tokens at a higher price and buy `BU` at a lower price (`line 176`).
- If a token needs to be purchased, we assume that we will sell `BU` at a higher price and buy this token at a lower price (`line 168`).

The `optimal top` calculation can vary in three ways:
1. Keeping the `original top` (skip `line 123`).
2. Selecting the maximum between the `original top` and `basketsNeeded`, as is currently done.
3. Assuming that sell all tokens exceed the `bottom` at a higher price and buying `BU` at a lower price.
It's challenging to determine which method is the best or most accurate.

I couldn't find a clear reason for choosing the current method, but I think it's valuable to compare the three approaches. 
In my opinion, the first method—keeping the `original top`—is more accurate for determining the `optimal top`. 
This approach can be imagined as borrowing some `BUs` to buy all tokens up to the `original top` and then repaying the borrowed `BUs`, which seems to align better with achieving an optimal result.

# [L-17] Proposers cannot cancel their proposals

Normally, `proposers` can `cancel` their `proposals` while they are in the `pending` status.
```
function cancel(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) public virtual override returns (uint256) {
    uint256 proposalId = hashProposal(targets, values, calldatas, descriptionHash);
    require(state(proposalId) == ProposalState.Pending, "Governor: too late to cancel");
    require(_msgSender() == _proposals[proposalId].proposer, "Governor: only proposer can cancel");
    return _cancel(targets, values, calldatas, descriptionHash);
}
```
However, the `cancel` function has been updated to disable this capability. 
Currently, only `proposals` from the old `era` can be canceled (`line 131`), but this situation will be rare since switching to the next `era` is infrequent.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/plugins/governance/Governance.sol#L124-L134
```
function cancel(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) public override(Governor, IGovernor) returns (uint256) {
    uint256 proposalId = _cancel(targets, values, calldatas, descriptionHash);
131:    require(!startedInSameEra(proposalId), "same era");

    return proposalId;
}
```

# [L-18] Staking benefits new stakers

In a typical staking system, when new depositors `stake` `assets`, their `shares` are rounded down.
However, in the `StRSR`, `shares` are rounded up (`line 623, 729`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L727-L730
```
function _payoutRewards() internal {
    stakeRate = (stakeRSR == 0 || totalStakes == 0)
        ? FIX_ONE
623:        : uint192((totalStakes * FIX_ONE_256 + (stakeRSR - 1)) / stakeRSR);
}

function mintStakes(address account, uint256 rsrAmount) private {
    uint256 newStakeRSR = stakeRSR + rsrAmount;
729:    uint256 newTotalStakes = (stakeRate * newStakeRSR) / FIX_ONE;
    uint256 stakeAmount = newTotalStakes - totalStakes;
    stakeRSR += rsrAmount;
    _mint(account, stakeAmount);
}
```
This means that `stakers` receive immediate benefits, even if it's just `1 wei`. 
Additionally, while the `stakeRate` value changes slightly with each staking event, we do not update it accordingly.
Currently, we only update the `stakeRate` in the `_payoutRewards` and `seizeRSR` functions.
We should either update the `stakeRate` in the `stake` and `unstake` functions or remove `stakeRate` and use a standard approach similar to `ERC4626`.

# [L-19] The `sqrt256` function can return an incorrect result

In this function, we iterate only `7` times because the result is a `128-bit` number, and `Newton's method` converges quadratically. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Fixed.sol#L756-L770
```
function sqrt256(uint256 x) pure returns (uint256 result) {
    // At this point, `result` is an estimation with at least one bit of precision. We know the true value has at
    // most 128 bits, since it is the square root of a uint256. Newton's method converges quadratically (precision
    // doubles at every iteration). We thus need at most 7 iteration to turn our partial result with one bit of
    unchecked {
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        result = (result + x / result) >> 1;
        uint256 roundedResult = x / result;
        if (result >= roundedResult) {
            result = roundedResult;
        }
    }
}
```
Quadratic convergence means that the error reduces by a factor of approximately `2` with each step, effectively doubling the precision.
But there is no guarantee that we will always get the correct result in `7` steps

However, this `sqrt` function is only used in the `_checkpointsLookup` function in the `StRSR`, so its accuracy is not critical (`line 140`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSRVotes.sol#L140
```
function _checkpointsLookup(Checkpoint[] storage ckpts, uint256 timepoint)
    private
    view
    returns (uint256)
{
    uint256 length = ckpts.length;
    uint256 low = 0;
    uint256 high = length;
    if (length > 5) {
140:        uint256 mid = length - MathUpgradeable.sqrt(length);
        if (ckpts[mid].fromTimepoint > timepoint) {
            high = mid;
        } else {
            low = mid + 1;
        }
    }
}
```

# [L-20] The `manageTokens` function can revert if there are not enough tokens to distribute.

In the `manageTokens` function, if `tokenToBuy` is included in the input `erc20` array, the function attempts to distribute these tokens (`line 130`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RevenueTrader.sol#L130
```
function manageTokens(IERC20[] calldata erc20s, TradeKind[] calldata kinds)
    external
    nonReentrant
    notTradingPausedOrFrozen
{
    uint256 len = erc20s.length;
    require(len != 0, "empty erc20s list");
    require(len == kinds.length, "length mismatch");
    RevenueTotals memory revTotals = distributor.totals();
    
    require(
        (tokenToBuy == rsr && revTotals.rsrTotal != 0) ||
            (address(tokenToBuy) == address(rToken) && revTotals.rTokenTotal != 0),
        "zero distribution"
    );

    for (uint256 i = 0; i < len; ++i) {
        if (erc20s[i] == tokenToBuy) {
130:            _distributeTokenToBuy();
            if (len == 1) return; // return early if tokenToBuy is only entry
        }
    }
}
```
However, if there aren't enough tokens to distribute, the function will revert (`line 134`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Distributor.sol#L134
```
function distribute(IERC20 erc20, uint256 amount) external {
    address caller = _msgSender();
    require(caller == rsrTrader || caller == rTokenTrader, "RevenueTraders only");
    require(erc20 == rsr || erc20 == rToken, "RSR or RToken");
    bool isRSR = erc20 == rsr; // if false: isRToken
    uint256 tokensPerShare;
    uint256 totalShares;
    {
        RevenueTotals memory revTotals = totals();
        totalShares = isRSR ? revTotals.rsrTotal : revTotals.rTokenTotal;
        if (totalShares != 0) tokensPerShare = amount / totalShares;
134:        require(tokensPerShare != 0, "nothing to distribute");
    }
}
```

To handle this more gracefully, it's better to wrap the call to the `_distributeTokenToBuy` function with a `try/catch` block.
This is important because users typically intend to distribute tokens before managing them.

# [L-21] Only versions that have already been deployed should be deprecated

In the `VersionRegistry`, the `deprecateVersion` function currently does not check whether the version has already been deployed. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/registry/VersionRegistry.sol#L59-L70
```
function deprecateVersion(bytes32 versionHash) external {
    if (!roleRegistry.isOwnerOrEmergencyCouncil(msg.sender)) {
        revert VersionRegistry__InvalidCaller();
    }
  
    if (isDeprecated[versionHash]) {
        revert VersionRegistry__AlreadyDeprecated();
    }
    
    isDeprecated[versionHash] = true;
  
    emit VersionDeprecated(versionHash);
}
```
Only deployed versions should be deprecated, so the function should be updated to ensure that undeployed versions cannot be deprecated.

# [L-22] The `totalDrafts` should be updated in the `withdraw` and `cancelUnstake` functions

Currently, in these functions, `totalDrafts` is not updated when the `rsrAmount` is 0. 
However, when the `draftAmount` is extremely small (in `line 324`), the `rsrAmount` can be `0`, leading to the omission of the `totalDrafts` update in `line 329`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L329
```
function withdraw(address account, uint256 endId) external {
324:    uint256 newTotalDrafts = totalDrafts - draftAmount;
    uint256 newDraftRSR = (newTotalDrafts * FIX_ONE_256 + (draftRate - 1)) / draftRate;
    uint256 rsrAmount = draftRSR - newDraftRSR;
  
329:    if (rsrAmount == 0) return;

    totalDrafts = newTotalDrafts;

    draftRSR = newDraftRSR;
}
```
This omission can break the invariant that `totalDrafts` should equal the sum of drafts for all users.

# [L-23] Users may redeem less than expected in the `redeemTo` function

In this function, users `redeem` tokens by burning their `rTokens`, and the redeemable amount depends on `basketsNeeded` and the `total supply` (`line 510`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L510
```
function _scaleDown(address account, uint256 amtRToken) private returns (uint192 amtBaskets) {
510:    amtBaskets = basketsNeeded.muluDivu(amtRToken, totalSupply()); // FLOOR
    emit BasketsNeededChanged(basketsNeeded, basketsNeeded - amtBaskets);
    basketsNeeded -= amtBaskets;

    _burn(account, amtRToken);
}
```
These values can change over time. 
For instance, `basketsNeeded` can be decreased by the `setBasketsNeeded` function. 
Although `setBasketsNeeded` can only be called when the `basket` isn't fully collateralized and the `redeemTo` function is restricted to when the `basket` is fully collateralized, it's still possible for the redeem transaction to execute after some time, resulting in users redeeming less than they initially expected.
It would be better to add minimum amounts that must be redeemed to prevent users from receiving less than expected.

# [L-24] The calculation should be rounded up in the `getHistoricalBasket` function

When calculating token quantities in the `basket`, all calculations are typically rounded up (`line 142`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L139-L143
```
function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {

    (address[] memory erc20s, uint256[] memory deposits) = basketHandler.quote(
        amtBaskets,
        true,
142:        CEIL
    );
}
```
However, in the `getHistoricalBasket` function, the calculation is currently rounded down (`line 722, 723`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L722-L723
```
function getHistoricalBasket(uint48 basketNonce)
    external
    view
    returns (IERC20[] memory erc20s, uint256[] memory quantities)
{
    for (uint256 i = 0; i < b.erc20s.length; ++i) {
        erc20s[i] = b.erc20s[i];
        try assetRegistry.toAsset(IERC20(erc20s[i])) returns (IAsset asset) {
            if (!asset.isCollateral()) continue; // skip token if no longer registered
  
            quantities[i] = b
            .refAmts[erc20s[i]]
722:            .safeDiv(ICollateral(address(asset)).refPerTok(), FLOOR)
723:            .shiftl_toUint(int8(asset.erc20Decimals()), FLOOR);
        } catch (bytes memory errData) {
            if (errData.length == 0) revert(); // solhint-disable-line reason-string
        }
    }
}
```

# [L-25] The calculation of `minBuyAmount` should be rounded down in the `prepareTradeSell` function

The `minBuyAmount` represents the minimum possible amount of the `buy` token that can be obtained from the `trade` in the worst-case scenario, where the `sell` token is sold at a lower price and the `buy` token is purchased at a higher price with `maximum trading slippage` applied. 
Therefore, `minBuyAmount` should be rounded down to reflect this worst-case scenario accurately.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/TradeLib.sol#L76-L80
```
function prepareTradeSell(
    TradeInfo memory trade,
    uint192 minTradeVolume,
    uint192 maxTradeSlippage
) internal view returns (bool notDust, TradeRequest memory req) {

    uint192 b = s.mul(FIX_ONE.minus(maxTradeSlippage)).safeMulDiv(
        trade.prices.sellLow,
        trade.prices.buyHigh,
        CEIL
    );
}
```
This is important because if the actual `buy amount` is less than the `minBuyAmount`, the `trade` will fail, making even a small difference (like `1 wei`) significant.

# [L-26]  The `swapRegistered` function should check whether the swapped assets are different

Governance can swap one `asset` for another, but currently, there is no check to ensure that these `assets` are different. 
If the tokens are the same, there will be no change in the `basket`, but the `basket` might be disabled in `line 94`.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/AssetRegistry.sol#L94
```
function swapRegistered(IAsset asset) external governance returns (bool swapped) {
    require(_erc20s.contains(address(asset.erc20())), "no ERC20 collision");

    try basketHandler.quantity{ gas: _reserveGas() }(asset.erc20()) returns (uint192 quantity) {
94:        if (quantity != 0) basketHandler.disableBasket(); // not an interaction
    } catch {
        basketHandler.disableBasket();
    }

    swapped = _registerIgnoringCollisions(asset);
}
```

# [L-27] Anyone can call the `manageTokens` function in the `RevenueTraders` contract

There is no `access control` in the `manageTokens` function, allowing any user to create any type of `trades` for any tokens. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RevenueTrader.sol#L109-L112
```
function manageTokens(IERC20[] calldata erc20s, TradeKind[] calldata kinds)
    external
    nonReentrant
    notTradingPausedOrFrozen
{
}
```
Since different `trades` produce varying results and some tokens are more beneficial in a `Dutch auction` than others, it would be better if only professional managers had access to this function.

# [L-28] The `price` function can revert when the auction status is not `OPEN`

In a `Dutch auction`, the `price` function returns the current `bid price` based on the current timestamp. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/plugins/trading/DutchTrade.sol#L359
```
function _price(uint48 timestamp) private view returns (uint192) {
    uint48 _startTime = startTime; // {s} gas savings
    uint48 _endTime = endTime; // {s} gas savings
    require(timestamp >= _startTime, "auction not started");
    require(timestamp <= _endTime, "auction over");

    return worstPrice;
}
```
However, this function should revert if the `auction status` is not `OPEN`. 
In other words, askers should not be able to obtain a `bid price` once the `auction` has concluded.

# [L-29] There is no check to ensure that `shortFreeze` is smaller than `longFreeze` in the `Auth` contract

Currently, there is only an `upper limit` check for `freeze` times, but no verification to ensure that `shortFreeze` is less than `longFreeze`. 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/mixins/Auth.sol#L205-L209
```
function setLongFreeze(uint48 longFreeze_) public onlyRole(OWNER) {
    require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE, "long freeze out of range");
    emit LongFreezeDurationSet(longFreeze, longFreeze_);
    longFreeze = longFreeze_;
}
```
To address this, add a check to ensure that `shortFreeze` is always smaller than `longFreeze`.

# [L-30] The exchange rate check should be included in the `melt` function

When `basketsNeeded` changes in the `setBasketsNeeded` function, the `exchange rate` is checked to ensure it does not exceed the set limit (`line 413`). 
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L413
```
function setBasketsNeeded(uint192 basketsNeeded_) external notTradingPausedOrFrozen {
    uint256 low = (FIX_ONE_256 * basketsNeeded_) / supply; // D18{BU/rTok}
    uint256 high = (FIX_ONE_256 * basketsNeeded_ + (supply - 1)) / supply; // D18{BU/rTok}
  
413:    require(low >= MIN_EXCHANGE_RATE && high <= MAX_EXCHANGE_RATE, "BU rate out of range");
}
```
However, the `melt` function currently lacks this check, even though the `exchange rate` increase.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L370-L375
```
function melt(uint256 amtRToken) external {
    address caller = _msgSender();
    require(caller == address(furnace), "furnace only");
    _burn(caller, amtRToken);
    emit Melted(amtRToken);
}
```
While the increase might be small, it's better to add this check to maintain consistency and prevent potential issues.

# [L-31] There is no check to ensure that the backup token has a valid `targetName`.

Backup tokens should have the same `targetName` as the tokens they are meant to back. 
However, this validation is missing in the `setBackupConfig` function.
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L284-L303
```
function setBackupConfig(
    bytes32 targetName,
    uint256 max,
    IERC20[] calldata erc20s
) external {
    requireGovernanceOnly();
    require(max <= MAX_BACKUP_ERC20S && erc20s.length <= MAX_BACKUP_ERC20S, "too large");
    requireValidCollArray(erc20s);
    BackupConfig storage conf = config.backups[targetName];
    conf.max = max;
    delete conf.erc20s;
    for (uint256 i = 0; i < erc20s.length; ++i) {
        assetRegistry.toColl(erc20s[i]); // reverts if not collateral
        conf.erc20s.push(erc20s[i]);
    }
    emit BackupConfigSet(targetName, max, erc20s);
}
```

# [L-32] An overflow can occur in the `mul` function.

In this function, two `uint192` numbers are multiplied. 
Since the result can exceed the `uint256` limit, an overflow may happen (`line 264`).
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Fixed.sol#L264
```
function mul(
    uint192 x,
    uint192 y,
    RoundingMode rounding
) internal pure returns (uint192) {
264:    return _safeWrap(_divrnd(uint256(x) * uint256(y), FIX_SCALE, rounding));
}
```