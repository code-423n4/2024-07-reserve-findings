# QA for Reserve

## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [L-01](#l-01-strsrs-leakyrefresh-function-uses-outdated-stakersr-causing-inaccurate-refresh-triggers) | StRSR's leakyRefresh function uses outdated stakeRSR, causing inaccurate refresh triggers |
| [L-02](#l-02-incorrect-math-leakyrefresh) | Incorrect math `leakyRefresh()` |
| [L-03](#l-03-latestversion-variable-not-updated-on-deprecation-leading-to-potentially-returning-deprecated-versions) | `latestVersion` variable not updated on deprecation leading to potentially returning deprecated versions |
| [L-04](#l-04-potential-loss-of-collateral-value-for-the-protocol-due-to-unrestricted-dutch-auction-initiation-during-high-gas-prices) | Potential loss of collateral value for the protocol due to unrestricted Dutch auction initiation during high gas prices |
| [L-05](#l-05-throttle-should-be-handled-more-gracefully) | Throttle should be handled more gracefully |
| [L-06](#l-06-double-entry-tokens-can-be-unintentionally-swept-causing-loss-of-collateral) | Double-entry tokens can be unintentionally swept, causing loss of collateral |
| [L-07](#l-07-strsr-permit-flaw-allows-cross-era-token-approvals-because-it-doesnt-account-for-era) | `StRSR permit()` flaw allows cross-era token approvals because it doesn't account for era |
| [L-08](#l-08-bu-price-band-invariant-not-enforced-in-recollateralizationlib-basketrange) | BU Price band invariant not enforced in `RecollateralizationLib::basketRange` |
| [L-09](#l-09-incomplete-price-check-in-nexttradepair-function-allows-trading-of-partially-unpriced-assets) | Incomplete price check in `nextTradePair` function allows trading of partially unpriced assets |
| [L-10](#l-10-short_freezer-role-revocation-occurs-before-freeze-application-potentially-leaving-system-vulnerable) | SHORT_FREEZER role revocation occurs before freeze application, potentially leaving system vulnerable |
| [L-11](#l-11-inconsistent-price-freshness-check-for-rtoken-enables-dutch-auctions-with-potentially-stale-prices) | Inconsistent price freshness check for `RToken` enables dutch auctions with potentially stale prices |
| [L-12](#l-12-irreversible-asset-deprecation) | Irreversible asset deprecation |
| [L-13](#l-13-redemption-quantities-not-adjusted-for-mempool-time-leading-to-potential-over-redemption) | Redemption quantities not adjusted for mempool time leading to potential over-redemption |
| [L-14](#l-14-rsr-remains-seizable-even-after-unstaking-delay-expiration) | RSR remains seizable even after unstaking delay expiration |
| [L-15](#l-15-inconsistent-long_freezer-role-and-longfreezes-mapping-handling-compromises-the-freeze-mechanism) | Inconsistent `LONG_FREEZER` role and `longFreezes` mapping handling compromises the freeze mechanism |
| [L-16](#l-16-furnace-melt-function-vulnerable-to-front-running-during-freezes-reducing-redemption-value) | Furnace `melt()` function vulnerable to front-running during freezes reducing redemption value |
| [L-17](#l-17-new-rtoken-issuers-risk-losing-collateral-during-undercollateralization-periods) | New RToken issuers risk losing collateral during undercollateralization periods |
| [L-18](#l-18-inconsistency-between-doc-and-code-regarding-rsr-usage-in-recollateralization) | Inconsistency between doc and code regarding RSR usage in recollateralization |
| [L-19](#l-19-immutable-component-addresses-prevent-critical-updates-and-upgrades) | Immutable component addresses prevent critical updates and upgrades |
| [L-20](#l-20-underestimation-of-custom-redemption-amounts-due-to-floor-rounding) | Underestimation of custom redemption amounts due to floor rounding |
| [L-21](#l-21-the-nonce-architecture-of-the-delegatebysig-function-isnt-useful) | The `nonce` architecture of the `delegateBySig()` function isn't useful |
| [L-22](#l-22-historical-basket-redemption-arbitrage-allows-value-extraction-during-market-volatility) | Historical basket redemption arbitrage allows value extraction during market volatility |
| [L-23](#l-23-signature-encoding-for-erc1271-contract-wallets-could-lead-to-failed-transactions) | Signature encoding for ERC1271 contract wallets could lead to failed transactions |
| [L-24](#l-24-fragile-enum-ordering-dependency-in-isbettersurplus-function) | Fragile enum ordering dependency in `isBetterSurplus` function |
| [L-25](#l-25-unnecessary-coupling-of-issuance-and-redemption-throttles-in-issueto-function) | Unnecessary coupling of issuance and redemption throttles in `issueTo` function |


## [L-01] StRSR's leakyRefresh function uses outdated stakeRSR, causing inaccurate refresh triggers

### Impact
The `leakyRefresh` function calculates `totalRSR` without accounting for recently accrued rewards, potentially leading to inaccurate timing of asset registry refreshes. This could result in premature or delayed refresh calls, affecting protocol's responsiveness to asset composition changes. 

### Proof of Concept
In the leakyRefresh function, totalRSR is calculated as follows:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L700-L718
```solidity
function leakyRefresh(uint256 rsrWithdrawal) private {
    // ...
    uint256 totalRSR = stakeRSR + draftRSR + rsrWithdrawal; // {qRSR}
    uint192 withdrawal = _safeWrap((rsrWithdrawal * FIX_ONE + totalRSR - 1) / totalRSR); // {1}
    // ...
    if (leaked > withdrawalLeak) {
        // ...
        assetRegistry.refresh();
    }
}
```

This calculation doesn't include recently accrued rewards, which are typically added to `stakeRSR` in the `_payoutRewards` function:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L609-L610

```solidity
function _payoutRewards() internal {
    // ...
    payout = (payoutRatio * rsrRewardsAtLastPayout) / FIX_ONE;
    stakeRSR += payout;
    // ...
}
```

By not calling `_payoutRewards` before calculating `totalRSR`, `leakyRefresh` may use an outdated stakeRSR value, leading to an inaccurate calculation of the withdrawal percentage and potentially incorrect decisions about when to trigger a refresh.

### Recommended Mitigation Steps
`leakyRefresh` function should call `_payoutRewards` at the beginning of its execution. This will ensure that stakeRSR is up-to-date with all accrued rewards before calculating totalRSR, leading to more accurate refresh trigger decisions. I believe that the slight increase in gas cost from this additional call is outweighed by the improved accuracy and fairness in the protocol


## [L-02] Incorrect math `leakyRefresh()` 

### Impact
The `leakyRefresh()` function incorrectly calculates the withdrawal percentage, leading to an overestimation of the total withdrawn amount. This causes the refresh mechanism to trigger more frequently/earlier than intended.

### Proof of Concept
Take a look at the `leakyRefresh()` function:
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L700-L718

```solidity
function leakyRefresh(uint256 rsrWithdrawal) private {
    uint48 lastRefresh = assetRegistry.lastRefresh(); // {s}

    // Assumption: rsrWithdrawal has already been taken out of draftRSR
    uint256 totalRSR = stakeRSR + draftRSR + rsrWithdrawal; // {qRSR}
    uint192 withdrawal = _safeWrap((rsrWithdrawal * FIX_ONE + totalRSR - 1) / totalRSR); // {1}

    // == Effects ==
    leaked = lastWithdrawRefresh != lastRefresh ? withdrawal : leaked + withdrawal;
    lastWithdrawRefresh = lastRefresh;

    if (leaked > withdrawalLeak) {
        leaked = 0;
        lastWithdrawRefresh = uint48(block.timestamp);

        /// == Refresh ==
        assetRegistry.refresh();
    }
}
```

The problem is that `totalRSR` includes `draftRSR`, which represents RSR already in the process of being withdrawn. As withdrawals occur, `totalRSR` decreases, causing the calculated `withdrawal` percentage to increase disproportionately.

For example, if the `withdrawalLeak` threshold is set to 30% and the initial `totalRSR` is 1,000,000:
1. The first user withdraws 5% (50,000 RSR). The calculation is correct: 50,000 / 1,000,000 = 5%.
2. After 4 such withdrawals, `totalRSR` is reduced to 800,000.
3. The 5th user also withdraws 50,000 RSR, but now the calculation becomes: 50,000 / 800,000 = 6.25%.
4. This process continues, with each withdrawal being calculated as a higher percentage.
5. The cumulative `leaked` amount reaches 30% after only about 5-6 withdrawals, instead of the expected 6 withdrawals (6 * 5% = 30%).

This leads to more frequent refreshes than intended.

### Recommended Mitigation Steps
Modify the `leakyRefresh()` function to calculate the withdrawal percentage based on the total RSR at the start of each refresh period, rather than the current total. Implement a new variable to store the initial total RSR at the beginning of each refresh period, and use this value for withdrawal percentage calculations. Update this initial value only when a refresh occurs. This will make sure that withdrawal percentages are calculated consistently throughout each refresh period, preventing premature refresh triggers.



## [L-03] `latestVersion` variable not updated on deprecation leading to potentially returning deprecated versions

### Impact
The `getLatestVersion()` function may return a deprecated version as the latest one, which can mislead users and other contracts relying on this function to get the most recent non-deprecated version.

### Proof of Concept
Take a look at `deprecateVersion` function: https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/registry/VersionRegistry.sol#L59-L70

```solidity
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

Also look at `getLatestVersion` Function
```solidity
function getLatestVersion()
    external
    view
    returns (
        bytes32 versionHash,
        string memory version,
        IDeployer deployer,
        bool deprecated
    )
{
    versionHash = latestVersion;
    deployer = deployments[versionHash];
    version = deployer.version();
    deprecated = isDeprecated[versionHash];
}
```

The `deprecateVersion` function marks a version as deprecated but does not update the `latestVersion` variable if the deprecated version is the latest one. Consequently, the `getLatestVersion()` function may return a deprecated version as the latest one, which is not the expected behavior.

Consider this scenario:

1. Version A is registered and becomes the latest version.
2. Version A is later deprecated using deprecateVersion.
3. Calling getLatestVersion still returns Version A, even though it's deprecated.

### Recommended Mitigation Steps
Modify the `deprecateVersion` function to update the `latestVersion` when the current latest version is being deprecated. Implement a mechanism to find the most recent non-deprecated version and set it as the new `latestVersion`. Additionally, consider adding a check in `getLatestVersion` to return the most recent non-deprecated version, even if `latestVersion` hasn't been updated correctly. This ensures that the contract always provides consistent and up-to-date version information.





## [L-04] Potential loss of collateral value for the protocol due to unrestricted Dutch auction initiation during high gas prices

### Impact
A malicious actor can call `rebalance` with TradeKind for dutch auction when gas prices are big to make losses for the protocol. Because bidding with Dutch auction is costly for users, system will receive much less tokens for the trade than expected.

This issue arises from the combination of unrestricted access to the rebalance function and the higher gas costs associated with Dutch auction participation, which can deter bidders or lead to lower bids to offset gas costs.

### Proof of Concept
The issue stems from the `rebalance` function in the `BackingManagerP1` contract:
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L107-L172

```solidity
function rebalance(TradeKind kind) external nonReentrant {
    requireNotTradingPausedOrFrozen();

    // ... (other checks)

    (
        bool doTrade,
        TradeRequest memory req,
        TradePrices memory prices
    ) = RecollateralizationLibP1.prepareRecollateralizationTrade(ctx, reg);

    if (doTrade) {
        // ... (prepare trade)

        // Execute Trade
        ITrade trade = tryTrade(kind, req, prices);
        tradeEnd[kind] = trade.endTime(); // {s}
        tokensOut[sellERC20] = trade.sellAmount(); // {tok}
    } else {
        // Haircut time
        compromiseBasketsNeeded(basketsHeld.bottom);
    }
}
```
`BackingManager::rebalance` can be called by anyone. Users then provide type of auction that will be used: `Gnosis` or `Dutch`.
The difference between them is that Gnosis can be settled by anyone(he pays for gas), while Dutch auction should be settled by bidder. And this settle process is costly, because it will call rebalance for user again.

Because Dutch auction takes a lot of gas from user that means that they will pay less amount for the traded assets to compensate that gas. The more busy main net and bigger gas prices, the less bidders would like to pay.

Malicious user in times when gas prices are high can call(frontrun another users) BackingManager.rebalance with Dutch auction in order to make the protocol lose part of their collateral.

A malicious actor could monitor gas prices and front-run other transactions to call `rebalance` with `TradeKind.DUTCH_AUCTION` when gas prices spike, potentially forcing the system into unfavorable trades.

### Recommended Mitigation Steps
Any of the following could be considered:

1. Introduce a gas price threshold mechanism that restricts Dutch auctions during periods of high gas prices.
2. Implement access control on the `rebalance` function, limiting it to trusted roles or contracts.
3. Add a dynamic slippage protection mechanism that adjusts based on current gas prices and market conditions.





## [L-05] Throttle should be handled more gracefully

### Impact
The current implementation of [Throttle.useAvailable()](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Throttle.sol#L37-L65) reverts when the requested amount exceeds the available limit, even by as little as 1 wei. This behavior may frustrate users and dissuade them from attempting to issue `RTokens` in the future. Consequently, the protocol could miss opportunities to increase `RToken` liquidity and generate revenue through collateral lending and yield accumulation.

### Proof of Concept
Let's consider the following scenario:

Firstly, Bob checks the available issuance amount by calling `RToken.issuanceAvailable()` and proceeds to call `issue()` with the maximum permissible `RToken` amount for the current basket:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L429-L431
```solidity
    /// @return {qRTok} The maximum issuance that can be performed in the current block
    function issuanceAvailable() external view returns (uint256) {
        return issuanceThrottle.currentlyAvailable(issuanceThrottle.hourlyLimit(totalSupply()));
    }
```

Bob's transaction is unintentionally or maliciously front-run by another user's `issue()` call, causing Bob's transaction to fail.

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Throttle.sol#L57-L59
```solidity
        if (amount > 0) {
            require(uint256(amount) <= available, "supply change throttled");
            available -= uint256(amount);
```

This sequence repeats, potentially leading Bob to abandon his attempts and seek alternative investment platforms.

### Recommended Mitigation Steps
Consider refactoring the `issueTo()` function this way:

```diff
+        uint256 availableAmount = issuanceAvailable();
+        if (amount > availableAmount) amount = availableAmount;
+        require(amount > 0, "Issuance amount must be positive");
        issuanceThrottle.useAvailable(supply, int256(amount));
        redemptionThrottle.useAvailable(supply, -int256(amount));
```

This modification would resolve the issue while informing the caller of the actual issued amount through the emitted Issuance event. The existing require check at the beginning of the function logic could be removed if deemed unnecessary.







## [L-06] Double-entry tokens can be unintentionally swept, causing loss of collateral

### Impact

The current implementation of the `BackingManager`, `AssetRegistry`, and `BasketHandler` contracts does not account for double-entry tokens (tokens with multiple addresses, like TUSD). This can cause unintentional sweeping of excess tokens through secondary addresses, potentially causing a loss of collateral for the protocol. The impact is high as it could result in significant financial losses and undermine the protocol's collateralization.

### Proof of Concept

The issue stems from the combination of how tokens are registered, how their quantities are calculated, and how excess tokens are forwarded. 

1. In AssetRegistry.sol, tokens are registered without considering multiple addresses:

```solidity
function register(IAsset asset) external governance returns (bool) {
    return _register(asset);
}

function _register(IAsset asset) internal returns (bool registered) {
    require(
        !_erc20s.contains(address(asset.erc20())) || assets[asset.erc20()] == asset,
        "duplicate ERC20 detected"
    );
    registered = _registerIgnoringCollisions(asset);
}
```

2. In BasketHandler.sol, the `quantity` function doesn't account for multiple addresses:

```solidity
function quantity(IERC20 erc20) public view returns (uint192) {
    try assetRegistry.toColl(erc20) returns (ICollateral coll) {
        return _quantity(erc20, coll, CEIL);
    } catch {
        return FIX_ZERO;
    }
}
```

3. In [BackingManager.sol](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L241-L265), the `forwardRevenue` function calculates excess based on the registered address:

```solidity
uint192 req = needed.mul(basketHandler.quantity(erc20s[i]), CEIL);
if (bal.gt(req)) {
    uint256 delta = bal.minus(req).shiftl_toUint(int8(asset.erc20Decimals()));
    uint256 tokensPerShare = delta / (totals.rTokenTotal + totals.rsrTotal);
    if (tokensPerShare == 0) continue;

    if (totals.rsrTotal != 0) {
        erc20s[i].safeTransfer(address(rsrTrader), tokensPerShare * totals.rsrTotal);
    }
    if (totals.rTokenTotal != 0) {
        erc20s[i].safeTransfer(
            address(rTokenTrader),
            tokensPerShare * totals.rTokenTotal
        );
    }
}
```

For a double-entry token like TUSD, if only one address is registered, the `quantity` function will return zero for the unregistered address. This leads to `req` being zero, causing all tokens held at the secondary address to be considered excess and swept away.

### Recommended Mitigation Steps
Modify the AssetRegistry to support multiple addresses for a single token. This could be implemented as a mapping of secondary addresses to primary addresses.
 Also Update the token registration process in AssetRegistry to check for and properly handle double-entry tokens, ensuring all addresses are associated with the same asset.
Modify the BasketHandler's `quantity` function to account for the total balance across all addresses of a double-entry token.
Update the BackingManager's `forwardRevenue` function to use the total balance across all addresses when calculating excess amounts for double-entry tokens.








## [L-07] `StRSR permit()` flaw allows cross-era token approvals because it doesn't account for era

### Impact
This flaw allows signed permits to remain valid across different eras in the StRSR contract. If the protocol undergoes a reset (incrementing the era), previously signed permits would still be valid for the new era. This could lead to unintended token approvals and potentially compromise the access control mechanisms intended to be reset with each new era. 

>The impact is partially mitigated by the deadline parameter(i.e if it is set to a reasonable value), but the core issue of cross-era validity remains.

### Proof of Concept
The issue stems from the `permit()` function not including the `era` in its signature hash:
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L922-L942

```solidity
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public {
    require(block.timestamp <= deadline, "ERC20Permit: expired deadline");

    bytes32 structHash = keccak256(
        abi.encode(_PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline)
    );

    PermitLib.requireSignature(owner, _hashTypedDataV4(structHash), v, r, s);

    _approve(owner, spender, value);
}
```

The `_approve()` function, called at the end of `permit()`, uses the current `era` when setting the allowance:

```solidity
function _approve(
    address owner,
    address spender,
    uint256 amount
) internal {
    _notZero(owner);
    _notZero(spender);

    _allowances[era][owner][spender] = amount;
    emit Approval(owner, spender, amount);
}
```

If the `era` is incremented (which happens in the `beginEra()` function), a previously signed permit would still be valid and could be used to set an approval in the new era, potentially bypassing intended restrictions or resets associated with the era change.

### Recommended Mitigation Steps
Consider including the current `era` in the `structHash` calculation within the `permit()` function. This ensures that permits are only valid for the specific era in which they were signed. And modify the `permit()` function to incorporate the `era` like this:

```solidity
bytes32 structHash = keccak256(
    abi.encode(_PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline, era)
);
```

Additionally, update the `_PERMIT_TYPEHASH` to include the era in its structure. This change will invalidate all permits when the era changes, aligning with the contract's era-based reset mechanism.








## [L-08] BU Price band invariant not enforced in `RecollateralizationLib::basketRange`

### Impact
The `basketRange` function in `RecollateralizationLib.sol` does not enforce the invariant that the distance between `range.top` and `range.bottom` strictly decreases with each trade. This could lead to scenarios where the BU price band does not narrow as expected resulting in inefficient or incorrect recollateralization trades.

### Proof of Concept
From the [docs](https://github.com/code-423n4/2024-07-reserve/blob/main/docs/recollateralization.md?plain=1) specifically on line 35,

> As trades complete, the distance between the top and bottom of the BU price band _strictly decreases_; it should not even remain the same (assuming the trade cleared for nonzero volume).

The `basketRange` function calculates `range.top` and `range.bottom` but does not enforce the invariant that the distance between them strictly decreases:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/RecollateralizationLib.sol#L224-L227

```solidity
function basketRange(TradingContext memory ctx, Registry memory reg)
    internal
    view
    returns (BasketRange memory range)
{
    // Existing logic to calculate range.top and range.bottom...

    // ==== (3/3) Enforce (range.bottom <= range.top <= basketsNeeded) ====
    if (range.top > basketsNeeded) range.top = basketsNeeded;
    if (range.bottom > range.top) range.bottom = range.top;
}
```

The function calculates `range.top` and `range.bottom` based on the current state and potential outcomes of trades. However, it does not ensure that the distance between `range.top` and `range.bottom` strictly decreases with each trade. This could lead to situations where the BU price band does not narrow as expected, violating the invariant described in the documentation.

### Recommended Mitigation Steps
Add a check in the `basketRange` function to compare the previous and current values of `range.top` and `range.bottom`. Ensure that the distance between them strictly decreases with each trade. If the distance does not decrease, adjust the values accordingly or throw an error. 








## [L-09] Incomplete price check in `nextTradePair` function allows trading of partially unpriced assets

### Impact
The `nextTradePair` function only checks if the high price of an asset is zero before considering it for trading. This incomplete check allows assets with a zero low price but non-zero high price to be considered for trading. 

### Proof of Concept
Take a look at  the `nextTradePair` function: https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/RecollateralizationLib.sol#L302
```solidity
function nextTradePair(
    TradingContext memory ctx,
    Registry memory reg,
    BasketRange memory range
) private view returns (TradeInfo memory trade) {
    // ... (earlier code omitted for brevity)

    for (uint256 i = 0; i < reg.erc20s.length; ++i) {
        // ... (other conditions omitted)

        if (ctx.bals[i].gt(needed)) {
            (uint192 low, uint192 high) = reg.assets[i].price(); // {UoA/sellTok}

            if (high == 0) continue; // skip over worthless assets

            // {UoA} = {sellTok} * {UoA/sellTok}
            uint192 delta = ctx.bals[i].minus(needed).mul(low, FLOOR);

            // ... (rest of the function)
        }
    }
}
```

The function only checks if `high == 0` before continuing with calculations. If `low` is zero but `high` is non-zero, the asset will not be skipped. This leads to `delta` being calculated as zero, potentially causing the asset to be incorrectly considered for trading.

This contradicts the contract's own warning:

```solidity
// Warning: If the trading algorithm is changed to trade unpriced (0, FIX_MAX) assets it can
//          result in losses in GnosisTrade. Unpriced assets should not be sold in rebalancing.
```

### Recommended Mitigation Steps
Modify the price check in the `nextTradePair` function to skip assets with either a zero low price or a zero high price:

```diff
function nextTradePair(
    TradingContext memory ctx,
    Registry memory reg,
    BasketRange memory range
) private view returns (TradeInfo memory trade) {
    // ... (earlier code unchanged)

    for (uint256 i = 0; i < reg.erc20s.length; ++i) {
        // ... (other conditions unchanged)

        if (ctx.bals[i].gt(needed)) {
            (uint192 low, uint192 high) = reg.assets[i].price(); // {UoA/sellTok}

-           if (high == 0) continue; // skip over worthless assets
+           if (low == 0 || high == 0) continue; // skip over worthless or unpriced assets

            // {UoA} = {sellTok} * {UoA/sellTok}
            uint192 delta = ctx.bals[i].minus(needed).mul(low, FLOOR);

            // ... (rest of the function unchanged)
        }
    }
}
```








## [L-10] SHORT_FREEZER role revocation occurs before freeze application, potentially leaving system vulnerable

### Impact
The current implementation of the `freezeShort()` function in the `Auth.sol` contract revokes the SHORT_FREEZER role before applying the freeze. This can lead to a situation where the role is revoked but the freeze fails to apply, leaving the system in an unintended state. This discrepancy between implementation and doc behavior could result in:

1. Loss of SHORT_FREEZER capabilities without the intended system freeze.
2. Reduced ability to respond to emergencies if the SHORT_FREEZER role is unexpectedly lost.

### Proof of Concept
The [doc](https://reserve.org/protocol/smart_contracts/#system-states-and-roles) states: 
> "When the SHORT_FREEZER call freezeShort(), they relinquish their SHORT_FREEZER role, and can only be re-granted the role by the OWNER (governance)."

However, the [current implementation in `Auth.sol` is](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/mixins/Auth.sol#L124-L128):

```solidity
function freezeShort() external onlyRole(SHORT_FREEZER) {
    // Revoke short freezer role after one use
    _revokeRole(SHORT_FREEZER, _msgSender());
    freezeUntil(uint48(block.timestamp) + shortFreeze);
}
```

In this implementation, the SHORT_FREEZER role is revoked before the `freezeUntil()` function is called. If `freezeUntil()` reverts (e.g., if the new freeze time is not greater than the current `unfreezeAt`), the SHORT_FREEZER will lose their role without the system being frozen.

### Recommended Mitigation Steps
Reverse the order of operations in the `freezeShort()` function to ensure the freeze is applied before revoking the role:

```diff
function freezeShort() external onlyRole(SHORT_FREEZER) {
-   // Revoke short freezer role after one use
-   _revokeRole(SHORT_FREEZER, _msgSender());
    freezeUntil(uint48(block.timestamp) + shortFreeze);
+   // Revoke short freezer role after successful freeze
+   _revokeRole(SHORT_FREEZER, _msgSender());
}
```








## [L-11] Inconsistent price freshness check for `RToken` enables dutch auctions with potentially stale prices

### Impact
The current implementation allows Dutch auctions to proceed with potentially stale RToken prices, while enforcing stricter freshness checks on other assets. This may lead to mispriced auctions, creating unfair trading conditions and potential exploitation opportunities. 

### Proof of Concept
This emanates from the [`pricedAtTimestamp`](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Broker.sol#L297-L301) function, which is called by `newDutchAuction` to verify price freshness:

```solidity
function pricedAtTimestamp(IAsset asset) private view returns (bool) {
    return asset.lastSave() == block.timestamp || address(asset.erc20()) == address(rToken);
}
```


This function returns true in two cases: 
a. If the asset's `lastSave()` is equal to the current block.timestamp 
b. If the asset is the RToken

The issue arises from the special treatment of the RToken:
For regular assets, the function checks if the price was updated in the current block (lastSave() == block.timestamp).
For the RToken, it always returns true without any timestamp check.
The potential problem:
This implementation assumes that the RToken's price is always up-to-date or that using a potentially stale RToken price is acceptable.
In reality, the RToken's price could be outdated, but the function would still consider it "priced at timestamp", potentially allowing a Dutch auction to start with stale price data.

Look at this:
This function always returns true for RToken without checking its price update timestamp:

```solidity
address(asset.erc20()) == address(rToken)
```

In contrast, other assets are checked for updates in the current block:

```solidity
asset.lastSave() == block.timestamp
```

The `newDutchAuction` function relies on this check:

```solidity
require(
    pricedAtTimestamp(req.sell) && pricedAtTimestamp(req.buy),
    "dutch auctions require live prices"
);
```

This allows Dutch auctions involving RToken to proceed even if its price data is outdated, while other assets are subject to stricter freshness requirements.

### Recommended Mitigation Steps
Implement consistent price freshness checks for all assets, including RToken. Introduce a configurable maximum age for price data and apply it uniformly across all assets. Consider adding a `lastPriceUpdate` function to the RToken interface to standardize price update tracking. Modify the `pricedAtTimestamp` function to use this consistent approach, ensuring that all assets, including RToken, are checked for price freshness within an acceptable time frame before allowing a Dutch auction to proceed. 










## [L-12] Irreversible asset deprecation 

### Impact

The current implementation of asset deprecation in the AssetPluginRegistry contract lacks reversibility, which can lead to permanent loss of asset validity even in cases of accidental deprecation or some sort of maintenance.

### Proof of Concept

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/registry/AssetPluginRegistry.sol#L107-L123

```solidity
function deprecateAsset(address _asset) external {
    if (!roleRegistry.isOwnerOrEmergencyCouncil(msg.sender)) {
        revert AssetPluginRegistry__InvalidCaller();
    }

    isDeprecated[_asset] = true;
}

function isValidAsset(bytes32 _versionHash, address _asset) external view returns (bool) {
    if (!isDeprecated[_asset]) {
        return _isValidAsset[_versionHash][_asset];
    }

    return false;
}
```

Once an asset is deprecated using the `deprecateAsset` function, there's no mechanism to reverse this action. The `isValidAsset` function always returns `false` for deprecated assets, regardless of their previous validity status for specific versions. This creates a permanent and irreversible state change that doesn't account for potential needs to reactivate assets or correct mistaken deprecations.

### Recommended Mitigation Steps

Modify the `deprecateAsset` function to accept a boolean parameter indicating whether to deprecate or undeprecate an asset. This change allows for both deprecation and reactivation of assets. Additionally, consider updating the `isValidAsset` function to incorporate more nuanced logic that takes into account both the deprecation status and the version-specific validity of an asset. 









## [L-13] Redemption quantities not adjusted for mempool time leading to potential over-redemption

### Impact
Users may receive more collateral tokens than expected during redemptions if the transaction stays in the mempool for an extended period

### Proof of Concept
The [doc](https://github.com/code-423n4/2024-07-reserve/blob/main/docs/system-design.md?plain=1) states:

> On the other hand, while a redemption is pending in the mempool, the quantities of collateral tokens the redeemer will receive steadily decreases. If a furnace melting happens in that time the quantities will be increased, causing the redeemer to get more than they expected.

Albeit, the implementation in `BasketHandler.sol` does not account for this dynamic behavior. The `quote` and `quoteCustomRedemption` functions calculate redemption quantities based on the current state at the time of calling, without any way to adjust for mempool time:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L482-L512

```solidity
function quote(
        uint192 amount,
        bool applyIssuancePremium,
        RoundingMode rounding
    ) public view returns (address[] memory erc20s, uint256[] memory quantities) {
        uint256 length = basket.erc20s.length;
        erc20s = new address[](length);
        quantities = new uint256[](length);

        for (uint256 i = 0; i < length; ++i) {
            erc20s[i] = address(basket.erc20s[i]);
            ICollateral coll = assetRegistry.toColl(IERC20(erc20s[i]));

            // {tok} = {tok/BU} * {BU}
            uint192 q = _quantity(basket.erc20s[i], coll, rounding).safeMul(amount, rounding);

            // Prevent toxic issuance by charging more when collateral is under peg
            if (applyIssuancePremium) {
                uint192 premium = issuancePremium(coll); // {1} always CEIL by definition

                // {tok} = {tok} * {1}
                if (premium > FIX_ONE) q = q.safeMul(premium, rounding);
            }

            // {qTok} = {tok} * {qTok/tok}
            quantities[i] = q.shiftl_toUint(
                int8(IERC20Metadata(address(basket.erc20s[i])).decimals()),
                rounding
            );
        }
    }
```

This function calculates the redemption quantities at the time it's called, but these quantities are not adjusted when the actual redemption transaction is processed. If the transaction spends time in the mempool, the redeemer may receive more tokens than they should, based on the current state of the protocol at the time of transaction execution.

### Recommended Mitigation Steps
Implement a mechanism to recalculate redemption amounts at the time of transaction execution, rather than relying solely on the initially quoted amounts. Maybe by storing a timestamp with the quote and adjusting the quantities based on the time elapsed between the quote and the execution








## [L-14] RSR remains seizable even after unstaking delay expiration

### Impact
The protocol implements an unstaking delay, as described in the documentation:

>Un-staking RSR comes with a delay, which is configurable by governance, and predicted to usually be between about 7 and 30 days. This delay is necessary so that in the event of a default, the staked RSR will remain in the staking contract for long enough to allow the RToken to seize any RSR it needs to cover losses.

The delay's purpose is to protect against RToken defaults. However, the current implementation allows RSR to be seized even after the delay period has expired, as long as the withdraw() function hasn't been called. This extends the risk exposure for stakers beyond the intended staking cycle.

The idea behind RSR staking is that stakers receive a portion of revenue in exchange for assuming some risk of RToken defaults. Similar to insurance, the premium should correspond to a specific coverage period. Once this period (staking duration + delay) ends, the coverage cycle should conclude. The current mechanism exposes stakers to additional risk without commensurate compensation.

### Proof of Concept
The draftRSR is only updated when the `withdraw()` function is called:
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSR.sol#L323-L327

```solidity
function withdraw(address account, uint256 endId) external notPausedOrFrozen {
    // ... (code omitted for brevity)
    uint256 newTotalDrafts = totalDrafts - draftAmount;
    uint256 newDraftRSR = (newTotalDrafts * FIX_ONE_256 + (draftRate - 1)) / draftRate;
    uint256 rsrAmount = draftRSR - newDraftRSR;
```

In the `seizeRSR()` function, the entire draftRSR is considered seizable:

```solidity
function seizeRSR(uint256 rsrAmount) external notPausedOrFrozen {
    // ... (code omitted for brevity)
    uint256 draftRSRToTake = (draftRSR * rsrAmount + (rsrBalance - 1)) / rsrBalance;
    draftRSR -= draftRSRToTake;
    seizedRSR += draftRSRToTake;
```

Consequently, even after the unstaking delay has passed, the unwithdrawn RSR can still be seized if withdraw() hasn't been called. This subjects stakers to unintended risk without additional compensation.

Consider a scenario where Bob stakes 100 RSR for a year, earning 10 RSR in revenue. With a 14-day unstaking delay, Bob unstakes his 100 RSR but doesn't withdraw on day 15. In principle, these 110 RSR should now belong to Bob and be isolated from the RToken's risks. However, if the RToken defaults on day 15, these 110 RSR could still be seized, resulting in an unintended loss for Bob.

### Recommended Mitigation Steps
Implement a mechanism to exclude RSR from the draftRSR once the corresponding unstaking delay has expired, regardless of whether `withdraw()` has been called. 









## [L-15] Inconsistent `LONG_FREEZER` role and `longFreezes` mapping handling compromises the freeze mechanism

### Impact
The inconsistent management of the `LONG_FREEZER` role and the `longFreezes` mapping in the Auth contract can lead to unexpected behavior in the protocol's freezing mechanism. This could potentially allow accounts to retain or regain freezing capabilities when they shouldn't, or lose these capabilities prematurely, undermining the intended access control and security measures of the protocol.

### Proof of Concept
There are three main scenarios where the inconsistency occurs:

1. Role revocation without resetting `longFreezes`:
When the LONG_FREEZER role is revoked, the `longFreezes` value for the account is not reset:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/mixins/Auth.sol#L87-L95

```solidity
function grantRole(bytes32 role, address account)
    public
    override(AccessControlUpgradeable, IAccessControlUpgradeable)
    onlyRole(getRoleAdmin(role))
{
    require(account != address(0), "cannot grant role to address 0");
    if (role == LONG_FREEZER) longFreezes[account] = LONG_FREEZE_CHARGES;
    _grantRole(role, account);
}
```

The inherited `revokeRole` function doesn't reset `longFreezes`, leaving a non-zero value for accounts that no longer have the LONG_FREEZER role.

2. Role renunciation without resetting `longFreezes`:
Similar to revocation, when an account renounces the LONG_FREEZER role, the `longFreezes` value is not reset. The inherited `renounceRole` function doesn't handle this.

3. Refilling `longFreezes` without checks:
The `grantRole` function can be used to refill `longFreezes` for an existing LONG_FREEZER without any checks as seen [here](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/mixins/Auth.sol#L93-L94):
```solidity
if (role == LONG_FREEZER) longFreezes[account] = LONG_FREEZE_CHARGES;
```

This allows unlimited refills of freeze charges, which may not be the intended behavior.

These inconsistencies can lead to situations where:
- Accounts without the LONG_FREEZER role still have non-zero `longFreezes` values.
- LONG_FREEZER accounts can have their `longFreezes` values refilled indefinitely.
- The contract's invariant that only accounts with the LONG_FREEZER role should have non-zero `longFreezes` values is violated.

### Recommended Mitigation Steps
1. Override the `revokeRole` function to reset `longFreezes`:

```solidity
function revokeRole(bytes32 role, address account) public override {
    if (role == LONG_FREEZER) longFreezes[account] = 0;
    super.revokeRole(role, account);
}
```

2. Override the `renounceRole` function to reset `longFreezes`:

```solidity
function renounceRole(bytes32 role, address account) public override {
    if (role == LONG_FREEZER) longFreezes[account] = 0;
    super.renounceRole(role, account);
}
```

3. Modify the `grantRole` function to prevent refilling `longFreezes` for existing LONG_FREEZER accounts:

```solidity
function grantRole(bytes32 role, address account) public override {
    require(account != address(0), "cannot grant role to address 0");
    if (role == LONG_FREEZER) {
        require(longFreezes[account] == 0, "account already has LONG_FREEZER charges");
        longFreezes[account] = LONG_FREEZE_CHARGES;
    }
    _grantRole(role, account);
}
```

Alternatively, if refilling is intended, document this behavior clearly and consider adding a separate function for refilling that's subject to appropriate access controls.








## [L-16] Furnace `melt()` function vulnerable to front-running during freezes reducing redemption value

### Impact
Users redeeming RTokens may receive less value than expected if a freeze occurs between transaction submission and execution. This discrepancy arises because the `melt()` function, which increases the assets available for redemption, is skipped during a freeze. The impact can be significant, especially after long periods of inactivity or consecutive freezes.

### Proof of Concept
The `Furnace.sol` contract contains a `melt()` function that's called during redemption to increase available assets:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Furnace.sol#L65-L79

```solidity
function melt() public {
    if (uint48(block.timestamp) < uint64(lastPayout + 1)) return;

    // # of whole periods that have passed since lastPayout
    uint48 numPeriods = uint48((block.timestamp) - lastPayout);

    // Paying out the ratio r, N times, equals paying out the ratio (1 - (1-r)^N) 1 time.
    uint192 payoutRatio = FIX_ONE.minus(FIX_ONE.minus(ratio).powu(numPeriods));

    uint256 amount = payoutRatio.mulu_toUint(lastPayoutBal);

    lastPayout += numPeriods;
    lastPayoutBal = rToken.balanceOf(address(this)) - amount;
    if (amount != 0) rToken.melt(amount);
}
```

There's no mechanism in this contract to:
1. Prevent `melt()` from being called during a freeze
2. Allow users to specify whether they're okay with skipping the `melt()` call during redemption

If a freeze occurs after a user submits a redemption transaction but before it's executed, the `melt()` call will be skipped (assuming it's wrapped in a try-catch block in the redemption process). This results in the user receiving less value than expected, as the additional assets from melting are not included.

### Recommended Mitigation Steps

Modify the redemption process to allow users to specify whether they're willing to proceed with redemption if `melt()` cannot be called if they don't specifically allow it,  then proceed to revert.








## [L-17] New RToken issuers risk losing collateral during undercollateralization periods

### Impact

When the protocol is undercollateralized, new RToken issuers face unfair risks. They may lose their entire collateral contribution or, at best, have to wait for recollateralization to complete before redeeming. This situation creates an imbalance where new issuers unknowingly subsidize existing token holders, potentially deterring participation in the protocol.

### Proof of Concept

The `issueTo` function in RToken.sol allows new issuance without checking if the protocol is fully collateralized:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L105-L162

```solidity
function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {
    require(amount != 0, "Cannot issue zero");

    assetRegistry.refresh();

    require(basketHandler.isReady(), "basket not ready");
    uint256 supply = totalSupply();

    // ... (issuance logic)

    _scaleUp(recipient, amtBaskets, supply);

    for (uint256 i = 0; i < erc20s.length; ++i) {
        IERC20Upgradeable(erc20s[i]).safeTransferFrom(
            issuer,
            address(backingManager),
            deposits[i]
        );
    }
}
```

The function only checks if the basket is ready, not if it's fully collateralized. This allows new issuers to deposit collateral even when the protocol is undercollateralized.

While regular redemption is prevented during undercollateralization:

```solidity
function redeemTo(address recipient, uint256 amount) public notFrozen {
    // ...
    require(basketHandler.fullyCollateralized(), "partial redemption; use redeemCustom");
    // ...
}
```

The `redeemCustom` function still allows partial redemptions during undercollateralization:

```solidity
function redeemCustom(
    address recipient,
    uint256 amount,
    uint48[] memory basketNonces,
    uint192[] memory portions,
    address[] memory expectedERC20sOut,
    uint256[] memory minAmounts
) external notFrozen {
    // ...
    // Prorate redemption
    for (uint256 i = 0; i < erc20s.length; ++i) {
        uint256 prorata = mulDiv256(
            IERC20(erc20s[i]).balanceOf(address(backingManager)),
            amount,
            supply
        );

        if (prorata < amounts[i]) amounts[i] = prorata;
    }
    // ...
}
```

This creates a situation where new issuers may not be able to fully redeem their newly minted RTokens, potentially losing some or all of their collateral.

### Recommended Mitigation Steps

Implement a check in the `issueTo` function to ensure the protocol is fully collateralized before allowing new issuance. by adding a requirement similar to the one in `redeemTo`. Also, consider adding clear warnings or preventing `redeemCustom` during undercollateralization periods.







## [L-18] Inconsistency between doc and code regarding RSR usage in recollateralization

### Impact
The discrepancy between the doc and the actual implementation could lead to misunderstandings about the protocol's behavior during recollateralization. This might result in incorrect assumptions by users, auditors, or developers about when and how the protocol takes a haircut.

### Proof of Concept
The [doc](https://github.com/code-423n4/2024-07-reserve/blob/main/docs/recollateralization.md?plain=1) states:

>If there does not exist a trade that meets these constraints, then the protocol "takes a haircut", which is a colloquial way of saying it reduces `RToken.basketsNeeded()` to its current BU holdings."

However, the `nextTradePair()` function in RecollateralizationLib.sol includes the following code:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/RecollateralizationLib.sol#L355-L376

```solidity
// Use RSR if needed
if (address(trade.sell) == address(0) && address(trade.buy) != address(0)) {
    (uint192 low, uint192 high) = reg.assets[rsrIndex].price(); // {UoA/RSR}

    // if rsr does not have a registered asset the below array accesses will revert
    if (
        high != 0 &&
        TradeLib.isEnoughToSell(
            reg.assets[rsrIndex],
            ctx.bals[rsrIndex],
            low,
            ctx.minTradeVolume
        )
    ) {
        trade.sell = reg.assets[rsrIndex];
        trade.sellAmount = ctx.bals[rsrIndex];
        trade.prices.sellLow = low;
        trade.prices.sellHigh = high;
    }
}
```

This code shows that the protocol attempts to sell RSR as a last resort before taking a haircut, which is not mentioned in the documentation. This creates a discrepancy between the expected and actual behavior of the protocol during recollateralization.

### Recommended Mitigation Steps
Update the documentation to accurately reflect the implementation. Add a section explaining that if no suitable trade is found among the collateral assets, the protocol will attempt to sell available RSR to acquire the needed asset before resorting to a haircut. 








## [L-19] Immutable component addresses prevent critical updates and upgrades

### Impact

The inability to update component addresses after initialization poses significant risks to the protocol's long-term viability and security. If vulnerabilities are discovered in any component contracts or if upgrades become necessary, the current design offers no room to replace these components. This limitation could lead to the entire system becoming vulnerable or outdated, 

### Proof of Concept

The `ComponentRegistry` contract initializes all component addresses in the `__ComponentRegistry_init` function, which can only be called once during contract initialization:

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/mixins/ComponentRegistry.sol#L18-L29

```solidity
function __ComponentRegistry_init(Components memory components_) internal onlyInitializing {
    _setBackingManager(components_.backingManager);
    _setBasketHandler(components_.basketHandler);
    _setRSRTrader(components_.rsrTrader);
    _setRTokenTrader(components_.rTokenTrader);
    _setAssetRegistry(components_.assetRegistry);
    _setDistributor(components_.distributor);
    _setFurnace(components_.furnace);
    _setBroker(components_.broker);
    _setStRSR(components_.stRSR);
    _setRToken(components_.rToken);
}
```

All setter functions, such as `_setRToken`, are private and can only be called within this initialization function:

```solidity
function _setRToken(IRToken val) private {
    require(address(val) != address(0), "invalid RToken address");
    emit RTokenSet(rToken, val);
    rToken = val;
}
```

This design means that once the contract is initialized, there's no way to update any component address, even if critical issues are discovered or upgrades become necessary.

### Recommended Mitigation Steps

Implement public setter functions for each component, protected by appropriate access control mechanisms (e.g., `onlyOwner` or a governance system). This change would allow authorized parties to update component addresses when necessary, improving the system's adaptability and long-term security. Additionally, consider implementing a time-lock or multi-signature requirement for these updates to provide an extra layer of security and transparency for users.








## [L-20] Underestimation of custom redemption amounts due to floor rounding

### Impact

The `quoteCustomRedemption` function uses FLOOR rounding when calculating redemption amounts for historical baskets. This can lead to users receiving less collateral than they are entitled to when redeeming RTokens, potentially allowing malicious actors to extract value from the system over time through multiple redemptions. The impact is amplified as the rounding down occurs at multiple stages of the calculation.

### Proof of Concept

The issue is in two main areas of the `quoteCustomRedemption` function:

1. When calculating the linear combination of historical baskets:  https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L562

```solidity
uint192 amt = portions[i].mul(b.refAmts[b.erc20s[j]], FLOOR);
```

2. When calculating the final redemption quantities: https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L597-L599

```solidity
quantities[i] = amount
    .safeMulDiv(refAmtsAll[i], collsAll[i].refPerTok(), FLOOR)
    .shiftl_toUint(int8(collsAll[i].erc20Decimals()), FLOOR);
```

In both cases, FLOOR rounding is used, which consistently rounds down the calculated amounts. This means that for each token in the basket, and for each historical basket used in the redemption, the user may receive slightly less than they should. The cumulative effect of these rounding errors can be significant, especially for large redemptions or when executed frequently.

### Recommended Mitigation Steps
Consider replacing all instances of FLOOR rounding with CEIL rounding in the `quoteCustomRedemption` function when calculating redemption amounts. This will make sure that users receive at least the amount of collateral they are entitled to. 








## [L-21] The `nonce` architecture of the `delegateBySig()` function isn't useful

### Proof of Concept
The user who needs to use this function must know the next nonce value in each operation and add it to the arguments. If they cannot find it, the function will revert. We can probably get this by querying the nonce on the blockchain during the front-end, but this is not an architecturally correct design and seriously consumes resources.

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSRVotes.sol#L166-L183

### Recommended Mitigation Steps
As best practice, we can provide practicality by using the design pattern that is used in many projects. 

```diff
  166  function delegateBySig(
  167      address delegatee,
- 168      uint256 nonce,
  169      uint256 expiry,
  170      uint8 v,
  171      bytes32 r,
  172      bytes32 s
  173  ) public {
  174      require(block.timestamp <= expiry, "signature expired");
+         uint256 currentNonce = _delegationNonces[signer];
  175      address signer = ECDSAUpgradeable.recover(
  176          _hashTypedDataV4(keccak256(abi.encode(_DELEGATE_TYPEHASH, delegatee, nonce, expiry))),
  177          v,
  178          r,
  179          s
  180      );
- 181      require(nonce == _useDelegationNonce(signer), "invalid nonce");
+         _delegationNonces[signer] = currentNonce + 1;
  182      _delegate(signer, delegatee);
  183  }
```

This would require adding a `_delegationNonces` mapping to store nonces for each signer:

```solidity
mapping(address => uint256) private _delegationNonces;
```









## [L-22] Historical basket redemption arbitrage allows value extraction during market volatility

### Proof of Concept

`quoteCustomRedemption` allows users to redeem RTokens based on historical basket compositions. While this feature is intended to provide flexibility, it could potentially be exploited in a way that wasn't intended:

The function allows redemption based on any basket nonce between `lastCollateralized` and the current `nonce` (line 537).

There's a [comment](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L540-L544) acknowledging a known limitation:

```solidity
// Known limitation: During an ongoing rebalance it may possible to redeem
// on a previous basket nonce for _more_ UoA value than the current basket.
// This can only occur for index RTokens, and the risk has been mitigated
// by updating `lastCollateralized` on every assetRegistry.refresh().
```

However, this mitigation might not be sufficient in all cases. If there's a significant price change in the collateral assets between basket rebalances, a user could potentially "cherry-pick" the most advantageous historical basket to redeem against, potentially receiving more value than they should. This issue is exacerbated by the fact that the function doesn't check the current prices of the assets when calculating redemption quantities. It only uses the historical reference amounts.
In extreme market conditions or during rapid price fluctuations, this could lead to a situation where users drain value from the protocol by strategically timing their redemptions and choosing the most profitable historical baskets.

In clear explaanations, the [`quoteCustomRedemption`](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L521-L601) function in the BasketHandler allows users to redeem RTokens based on historical basket compositions. This feature can be exploited during periods of high market volatility or during basket rebalancing:

1. A user monitors the market for significant price changes in the collateral assets.
2. When a large price discrepancy occurs between the current and a previous basket composition, the user calls `quoteCustomRedemption` with a carefully selected historical `basketNonce`.
3. The function calculates redemption quantities based on historical reference amounts without considering current market prices:

```solidity
quantities[i] = amount
    .safeMulDiv(refAmtsAll[i], collsAll[i].refPerTok(), FLOOR)
    .shiftl_toUint(int8(collsAll[i].erc20Decimals()), FLOOR);
```

4. The user receives more value in collateral assets than the current market value of their RTokens, effectively extracting value from the protocol.

To demonstrate:
- Assume Basket A (current): 1 ETH ($2000) + 2000 USDC
- Assume Basket B (historical): 1 ETH ($1000) + 1000 USDC
- If ETH price doubles to $2000, a user could redeem using Basket B, receiving 1 ETH + 1000 USDC (total value $3000) for RTokens worth only $2000 in the current basket.

### Recommended Mitigation Steps

Consider implementing a price adjustment mechanism in the `quoteCustomRedemption` function. This should compare the current market prices of collateral assets with their prices at the time of the historical basket's creation. Redemption quantities should be adjusted based on these price differences to ensure that users receive fair value regardless of which historical basket they choose for redemption. Additionally, consider implementing a cooldown period after basket rebalances before allowing custom redemptions, and potentially limiting the time window for historical basket redemptions to reduce the impact of long-term price volatility.








## [L-23] Signature encoding for ERC1271 contract wallets could lead to failed transactions

### Impact

The current implementation of the `requireSignature` function in the `PermitLib` library incorrectly encodes signatures for ERC1271 contract wallets. This can result in failed transactions or unauthorized access when interacting with certain smart contract wallets because they may reject the improperly formatted signature. 

### Proof of Concept

Issue is in the `requireSignature` function: https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Permit.sol#L10-L34

```solidity
if (AddressUpgradeable.isContract(owner)) {
    require(
        IERC1271Upgradeable(owner).isValidSignature(hash, abi.encodePacked(r, s, v)) ==
            0x1626ba7e,
        "ERC1271: Unauthorized"
    );
}
```

The issue is that the `isValidSignature` function for ERC1271 contracts expects the signature in a specific format, which is typically the concatenation of r, s, and v in that order. However, the current implementation uses `abi.encodePacked(r, s, v)`, which might not be the correct format for all ERC1271 implementations.

Some ERC1271 contracts might expect the signature in a different format, such as:

1. `bytes signature = abi.encodePacked(r, s, v)` (which is the current implementation)
2. `bytes signature = abi.encodePacked(v, r, s)` (more common for Ethereum signatures)
3. Or even as separate parameters: `(bytes32 r, bytes32 s, uint8 v)`
This inconsistency could lead to valid signatures being rejected for some contract wallets, even if they are actually valid.

### Recommended Mitigation Steps

Consider modifying the signature encoding for contract wallets.

```diff
 if (AddressUpgradeable.isContract(owner)) {
+    bytes memory signature = abi.encodePacked(r, s, v);
     require(
-        IERC1271Upgradeable(owner).isValidSignature(hash, abi.encodePacked(r, s, v)) ==
+        IERC1271Upgradeable(owner).isValidSignature(hash, signature) ==
             0x1626ba7e,
         "ERC1271: Unauthorized"
     );
 }
```








## [L-24] Fragile enum ordering dependency in `isBetterSurplus` function

### Proof of Concept

The [`isBetterSurplus` function](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/mixins/RecollateralizationLib.sol#L381-L399) relies on a specific ordering of the `CollateralStatus` enum:

```solidity
function isBetterSurplus(
    MaxSurplusDeficit memory curr,
    CollateralStatus other,
    uint192 surplusAmt
) private pure returns (bool) {
    // NOTE: If the CollateralStatus enum changes then this has to change!
    if (curr.surplusStatus == CollateralStatus.DISABLED) {
        return other == CollateralStatus.DISABLED && surplusAmt.gt(curr.surplus);
    } else if (curr.surplusStatus == CollateralStatus.SOUND) {
        return
            other == CollateralStatus.DISABLED ||
            (other == CollateralStatus.SOUND && surplusAmt.gt(curr.surplus));
    } else {
        // curr is IFFY
        return other != CollateralStatus.IFFY || surplusAmt.gt(curr.surplus);
    }
}
```

This implementation assumes a specific priority order (DISABLED > SOUND > IFFY) based on the enum's definition order. If the enum order changes, the function's logic will break.

### Recommended Mitigation Steps

Refactor the `isBetterSurplus` function to explicitly check enum values in the desired priority order, rather than relying on their implicit ordering. This can be achieved by using a series of if-else statements that compare against each enum value directly, ensuring the function remains correct even if the enum definition changes in the future. 









## [L-25] Unnecessary coupling of isuance and redemption throttles in `issueTo` function

### Impact
The current implementation of the `issueTo` function in the `RTokenP1` contract unnecessarily couples the issuance and redemption throttles. This can lead to potential manipulation in the sense that by tying the redemption throttle to issuance operations, the contract is creating an unnecessary dependency between these two operations and a malicious actor could potentially use this mechanism to manipulate the redemption throttle by performing issuance operations, even if they don't intend to hold the tokens.

### Proof of Concept
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L105-L155

```solidity
function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {
    require(amount != 0, "Cannot issue zero");

    // == Refresh ==

    assetRegistry.refresh();

    // == Checks-effects block ==

    address issuer = _msgSender(); // OK to save: it can't be changed in reentrant runs

    // Ensure basket is ready, SOUND and not in warmup period
    require(basketHandler.isReady(), "basket not ready");
    uint256 supply = totalSupply();

    // Revert if issuance exceeds either supply throttle
    issuanceThrottle.useAvailable(supply, int256(amount)); // reverts on over-issuance
    redemptionThrottle.useAvailable(supply, -int256(amount)); // shouldn't revert

    // ... (rest of the function)
}
```

### Recommended Mitigation Steps
Consider only check and update the issuance throttle during issuance operations. Also handle the redemption throttle separately, only checking and updating it during actual redemption operations.
Lastly, if theres a need to balance issuance and redemption rates, implement this at a higher level rather than coupling these operations directly.