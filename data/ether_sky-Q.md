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