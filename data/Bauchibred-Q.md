# QA Report for **Reserve Core**

## Table of Contents

| Issue ID                                                                                                                 | Description                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| [QA-01](<#qa-01-gnosistrade#init()s-minbuyamountperorder-calculation-should-be-based-on-asset-value-not-token-fraction>) | `GnosisTrade#init()`'s `minBuyAmountPerOrder` calculation should be based on asset value not token fraction |
| [QA-02](#qa-02-baskethandler.warmupperiod-can-be-changed-during-warm-up-period-disrupting-system-readiness)              | BasketHandler.warmupPeriod can be changed during warm-up period disrupting system readiness                 |
| [QA-03](#qa-03-consider-init-ing-gnosistrade-before-transfers-are-done)                                                  | Consider `init` ing GnosisTrade before transfers are done                                                   |
| [QA-04](#qa-04-reorg-attacks-can-lead-to-user-fund-trapping-and-incorrect-auction-bidding)                               | Reorg attacks can lead to user fund trapping and incorrect auction bidding                                  |
| [QA-05](#qa-05-fix-typos-_multiple-instances_)                                                                           | Fix typos _multiple instances_                                                                              |
| [QA-06](<#qa-06-consider-integrating-gnosistrade.cansettle()-into-gnosistrade.settle()-or-a-similar-functionality>)      | Consider integrating `GnosisTrade.canSettle()` into `GnosisTrade.settle()` or a similar functionality       |
| [QA-07](#qa-07-somewhat-broken-logic-in-regards-to-changing-the-delay-for-unstaking)                                     | Somewhat broken logic in regards to changing the delay for unstaking                                        |
| [QA-08](#qa-08-consider-removing-storage-gaps-in-favour-of-using-diamondstorage)                                         | Consider removing storage gaps in favour of using `DiamondStorage`                                          |
| [QA-09](#qa-09-invariant-of-having-unique-erc20-tokens-when-forwarding-revenue-can-be-broken)                            | Invariant of having unique erc20 tokens when forwarding revenue can be broken                               |
| [QA-10](#qa-10-an-admin-cant-set-the-most-accurate-value-for-a-_long-freeze_)                                            | An admin can't set the most accurate value for a _long freeze_                                              |
| [QA-11](<#qa-11-main#poke()-is-meant-for-testing-and-should-be-removed-before-final-production>)                         | `Main#poke()` is meant for testing and should be removed before final production                            |
| [QA-12](#qa-12-incorrect-logical-operator-in-auction-length-validation-allows-setting-zero-auction-length)               | Incorrect logical operator in auction length validation allows setting `zero auction length`                |
| [QA-13](#qa-13-remove-misleading-comments-from-production)                                                               | Remove misleading comments from production                                                                  |
| [QA-14](#qa-14-precision-issue-in-baskethandler.quote-function)                                                          | Precision issue in BasketHandler.quote function                                                             |
| [QA-15](<#qa-15-dutchtrade-inconsistent-time-check-in-cansettle()-&-settle()>)                                           | `DutchTrade`: Inconsistent time check in `canSettle()` & `settle()`                                         |
| [QA-16](#qa-16-wrong-implementation-of-the-storage-gap-in-auth.sol)                                                      | Wrong implementation of the storage gap in `Auth.sol`                                                       |
| [QA-17](#qa-17-trade-slippage-can-never-be-set-to-the-accepted-max)                                                      | Trade slippage can never be set to the accepted max                                                         |
| [QA-18](#qa-18-setters-dont-have-equality-checkers)                                                                      | Setters don't have equality checkers                                                                        |
| [QA-19](#qa-19-fix_max_int-is-wrongly-set)                                                                               | `FIX_MAX_INT` is wrongly set                                                                                |
| [QA-20](<#qa-20-permitlib#requiresignature()-could-fail-even-for-valid-signatures-from-counterfactual-wallets>)          | `PermitLib#requireSignature()` could fail even for valid signatures from counterfactual wallets             |
| [QA-21](<#qa-21-auth#setlongfreeze()-should-include-more-checks>)                                                        | `Auth#setLongFreeze()` should include more checks                                                           |
| [QA-22](#qa-22-remove-unused-imports-in-main.sol)                                                                        | Remove unused imports in `Main.sol`                                                                         |
| [QA-23](<#qa-23-consider-using-_disableinitializers()-for-upgradeable-contracts>)                                        | Consider using `_disableInitializers()` for upgradeable contracts                                           |
| [QA-24](#qa-24-import-declarations-should-import-specific-identifiers-rather-than-the-whole-file)                        | Import declarations should import specific identifiers, rather than the whole file                          |
| [QA-25](#qa-25-do-not-import-deprecated-files)                                                                           | Do not import deprecated files                                                                              |
| [QA-26](<#qa-26-attach-an-overflow-protection-in-fixed#plus()>)                                                          | Attach an overflow protection in `Fixed#plus()`                                                             |
| [QA-27](<#qa-27-links-in-core-documentation-should-not-lead-to-crashing-pages>)                                                          | Links in core documentation should not lead to crashing pages                                                             |

## QA-01 `GnosisTrade#init()`'s `minBuyAmountPerOrder` calculation should be based on asset value not token fraction

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/GnosisTrade.sol#L125-L128

```solidity
        uint256 minBuyAmtPerOrder = Math.max(
            minBuyAmount / MAX_ORDERS,
            DEFAULT_MIN_BID.shiftl_toUint(int8(buy.decimals()))
        );
```

Note that `DEFAULT_MIN_BID` is defined as:

```solidity
uint192 public constant DEFAULT_MIN_BID = FIX_ONE / 100; // {tok} 0.01 token units
```

This current implementation can cause problems for high-value tokens like WBTC. For example:

1. One hundredth of one WBTC is worth `~$700` at current prices.
2. If the assets being sold are worth only $100, no one will buy them for $700 (7x the price).
3. This mismatch can lead to failed trades or inefficient use of assets.

### Impact

> Borderline medium/low

This can:

- Prevent necessary trades from occurring.
- Lead to suboptimal pricing in trades.
- Cause inefficiencies in the protocol's asset management.

### Recommended Mitigation Steps

Consider revamping the whole logic, i.e calculate the `minBuyAmountPerOrder` based on the value of the asset being sold, rather than a fixed fraction of a token, i.e

1. Use the asset's price feed to determine its value in a stable unit (e.g., USD).
2. Set a minimum value threshold in USD (e.g., $10).
3. Calculate the `minBuyAmountPerOrder` based on this value threshold.

This approach ensures that the `minBuyAmountPerOrder` is always based on a reasonable value threshold, regardless of the token's individual price. For assets without a price feed, we can then fall back to the current method as a safeguard.

### Additional Note

As hinted under _Impact_, this is a borderline `low/medium`, considering how pricey WBTC is/can get, leaving it in the hands of the judge to upgrade if they see fit.

## QA-02 BasketHandler.warmupPeriod can be changed during warm-up period disrupting system readiness

### Proof of Concept

The `BasketHandler` contract uses a `warmupPeriod` to ensure a delay after a basket status change before the system is considered ready. However, the current implementation allows the `warmupPeriod` to be changed at any time, even during an ongoing warm-up period.

The `isReady()` function, which is used by other components to check if it's safe to interact with the basket handler, relies on this `warmupPeriod`:

See https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BasketHandler.sol#L331-L335

```solidity
function isReady() external view returns (bool) {
  return
    status() == CollateralStatus.SOUND && (block.timestamp >= lastStatusTimestamp + warmupPeriod);
}

```

The `warmupPeriod` can be changed by the owner at any time https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BasketHandler.sol#L633-L639

```solidity
function setWarmupPeriod(uint48 val) public {
  requireGovernanceOnly();
  require(val >= MIN_WARMUP_PERIOD && val <= MAX_WARMUP_PERIOD, 'invalid warmupPeriod');
  emit WarmupPeriodSet(warmupPeriod, val);
  warmupPeriod = val;
}

```

### Impact

> _QA considering the function is governance backed._
> If the `warmupPeriod` is decreased during an ongoing warm-up, the system might become ready earlier than intended, potentially before all necessary checks and balances have been completed. Conversely, if the `warmupPeriod` is increased during a warm-up, it could unexpectedly extend the time before the system is ready, leading to longer than necessary downtime.

### Recommended Mitigation Steps

Consider revamping the logic not to allow setting `warmupPeriod` during an ongoing warm-up.

## QA-03 Consider `init` ing GnosisTrade before transfers are done

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Broker.sol#L243-L266

```solidity
function newBatchAuction(TradeRequest memory req, address caller) private returns (ITrade) {
  require(!batchTradeDisabled, 'batch auctions disabled');
  require(batchAuctionLength != 0, 'batch auctions not enabled');
  GnosisTrade trade = GnosisTrade(address(batchTradeImplementation).clone());
  trades[address(trade)] = true;

  // Apply Gnosis EasyAuction-specific resizing of req, if needed: Ensure that
  // max(sellAmount, minBuyAmount) <= maxTokensAllowed, while maintaining their proportion
  uint256 maxQty = (req.minBuyAmount > req.sellAmount) ? req.minBuyAmount : req.sellAmount;

  if (maxQty > GNOSIS_MAX_TOKENS) {
    req.sellAmount = mulDiv256(req.sellAmount, GNOSIS_MAX_TOKENS, maxQty, CEIL);
    req.minBuyAmount = mulDiv256(req.minBuyAmount, GNOSIS_MAX_TOKENS, maxQty, FLOOR);
  }

  // == Interactions ==
  IERC20Upgradeable(address(req.sell.erc20())).safeTransferFrom(
    caller,
    address(trade),
    req.sellAmount
  );
  trade.init(this, caller, gnosis, batchAuctionLength, req); //@audit
  return trade;
}

```

Evidently, the initialization of the GnosisTrade contract happens after the transfer of tokens to it.

Now whereas SafeERC20's `safeTransferFrom` function will protect against failed transfers, it **does not** mitigate against unsafe external calls. In the event a _faux ERC20_ is added as an asset, the initialization of the GnosisTrade contract could happen earlier than expected, via re-entrancy, and subsequently cause the intended initialization to revert.

### Impact

This bug window would effectively lead to the DOS of trade requests which may prevent the RToken system from recollateralizing.

### Recommended Mitigation Steps

The initialization of the GnosisTrade contract should be done before the transfer of the tokens to the GnosisTrade contract. Apply these changes

```diff
    function newBatchAuction(TradeRequest memory req, address caller) private returns (ITrade) {
        require(!batchTradeDisabled, "batch auctions disabled");
        require(batchAuctionLength != 0, "batch auctions not enabled");
        GnosisTrade trade = GnosisTrade(address(batchTradeImplementation).clone());
        trades[address(trade)] = true;

        // Apply Gnosis EasyAuction-specific resizing of req, if needed: Ensure that
        // max(sellAmount, minBuyAmount) <= maxTokensAllowed, while maintaining their proportion
        uint256 maxQty = (req.minBuyAmount > req.sellAmount) ? req.minBuyAmount : req.sellAmount;

        if (maxQty > GNOSIS_MAX_TOKENS) {
            req.sellAmount = mulDiv256(req.sellAmount, GNOSIS_MAX_TOKENS, maxQty, CEIL);
            req.minBuyAmount = mulDiv256(req.minBuyAmount, GNOSIS_MAX_TOKENS, maxQty, FLOOR);
        }

+        trade.init(this, caller, gnosis, batchAuctionLength, req);//@audit

        // == Interactions ==
        IERC20Upgradeable(address(req.sell.erc20())).safeTransferFrom(
            caller,
            address(trade),
            req.sellAmount
        );
-        trade.init(this, caller, gnosis, batchAuctionLength, req);//@audit
        return trade;
    }
```

This ensures that the GnosisTrade contract is properly initialized before any token transfers occur, preventing potential re-entrancy attacks and maintaining the intended order of operations.

## QA-04 Reorg attacks can lead to user fund trapping and incorrect auction bidding

### Proof of Concept

While reorg attacks are not very common on mainnet, they can still occur, as evidenced by the [7-block reorg on the Ethereum Beacon chain](https://decrypt.co/101390/ethereum-beacon-chain-blockchain-reorg) before the merge. The risk is even higher on L2 networks, which the protocol is to deploy on per the readMe:

https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/README.md#L140-L141

```markdown
| Chains the protocol will be deployed on | Ethereum,Arbitrum,Base,Optimism |
```

Now the current implementation is vulnerable to reorg attacks in several scenarios:

1. RToken Deployment:
   A user might mint right after an RToken is deployed. In a reorg, a different RToken could be deployed to the same address, potentially trapping the user's funds as the deployer becomes the owner.

2. Dutch Auction:
   A reorg could switch the addresses of two auctions, causing a user to bid on the wrong auction. This is particularly relevant for the `DutchTrade` contract:

   ```solidity
   contract DutchTrade is ITrade, Versioned {
     // ...
     function bid() external returns (uint256 amountIn) {
       // ...
     }
     // ...
   }

   ```

3. Gnosis Auctions:
   Similar to Dutch Auctions, this can cause users to bid on the wrong auction.

### Impact

_Borderline low/medium_

1. Users could have their funds trapped in an unintended RToken contract.
2. Users might place bids on incorrect auctions, potentially leading to financial losses.
3. These issues could erode user trust in the protocol and potentially lead to reduced participation.

While the likelihood of these scenarios is low on mainnet, the potential impact is significant, especially since the protocol expands to L2 networks where reorgs are more common.

### Recommended Mitigation Steps

Use `create2` for contract deployments:
Deploy all contracts using `create2` with a salt that's unique to the features of the contract. This ensures that even in the case of a reorg, the contract wouldn't be deployed to the same address as before.

Example for RToken deployment:

```solidity
bytes32 salt = keccak256(abi.encodePacked(rTokenName, rTokenSymbol, owner));
address rTokenAddress = Create2.deploy(0, salt, rTokenBytecode);
```

## QA-05 Fix typos _multiple instances_

### Proof of Concept

> NB: Fixes denoted with "\*\* \*\*"

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Fixed.sol#L22

```diff
- //   Every function should revert iff its result is out of bounds.
+ //   Every function should revert **if** its result is out of bounds.
```

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/GnosisTrade.sol#L45

```diff
-    // from this trade's acution will all eventually go to origin.
+    // from this trade's **auction** will all eventually go to origin.
```

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/GnosisTrade.sol#L54-L55

```diff
-    // tokens. If we actually *get* a worse clearing that worstCasePrice, we consider it an error in
+    // tokens. If we actually *get* a worse clearing **than** worstCasePrice, we consider it an error in

```

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Furnace.sol#L61

```diff
-    //   lastPayoutBal' = rToken.balanceOf'(this) (balance now == at end of pay leriod)
+    //   lastPayoutBal' = rToken.balanceOf'(this) (balance now == at end of pay **period**)
```

### Impact

QA

### Recommended Mitigation Steps

```diff
+Apply the fixes
```

]## QA-04 Div before multiply in `Distributor.distribute()` would cause for lesser tokens to be distributed

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Distributor.sol#L120-L200

```solidity
function distribute(IERC20 erc20, uint256 amount) external {
  // Intentionally do not check notTradingPausedOrFrozen, since handled by caller

  address caller = _msgSender();
  require(caller == rsrTrader || caller == rTokenTrader, 'RevenueTraders only');
  require(erc20 == rsr || erc20 == rToken, 'RSR or RToken');
  bool isRSR = erc20 == rsr; // if false: isRToken

  uint256 tokensPerShare;
  uint256 totalShares;
  {
    RevenueTotals memory revTotals = totals();
    totalShares = isRSR ? revTotals.rsrTotal : revTotals.rTokenTotal;
    if (totalShares != 0) tokensPerShare = amount / totalShares; //@audit
    require(tokensPerShare != 0, 'nothing to distribute');
  }
  // snip
  for (uint256 i = 0; i < destinations.length(); ++i) {
    address addrTo = destinations.at(i);

    uint256 numberOfShares = isRSR ? distribution[addrTo].rsrDist : distribution[addrTo].rTokenDist;
    if (numberOfShares == 0) continue;
    uint256 transferAmt = tokensPerShare * numberOfShares; //@audit
    paidOutShares += numberOfShares;

    if (addrTo == FURNACE) {
      addrTo = address(furnace);
      if (transferAmt != 0) accountRewards = true;
    } else if (addrTo == ST_RSR) {
      addrTo = address(stRSR);
      if (transferAmt != 0) accountRewards = true;
    }

    transfers[numTransfers] = Transfer({ addrTo: addrTo, amount: transferAmt });
    ++numTransfers;
  }
  emit RevenueDistributed(erc20, caller, amount);

  //snip
}

```

This function is used to distribute revenue in rsr or rtoken, per the distribution table, issue however is that there's a div before multiplication during the distribution as shown by the @audit tag that could cause the amount distributed to be lower than it should.

### Impact

low, amount being distributed would be unfairly cut off since this would mean that value up to the amount of total shares (1e7 wei) can be locked in the sender without being able to distribute it.

### Recommended Mitigation Steps

Consider avoiding such precision loss.

## QA-06 Consider integrating `GnosisTrade.canSettle()` into `GnosisTrade.settle()` or a similar functionality

### Proof of Concept

The `settle()` function in the `GnosisTrade` contract is intended to be called after the auction has ended. However, the current implementation lacks a crucial check to ensure that the auction's end time has indeed passed.

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/GnosisTrade.sol#L175-L190

```solidity
    function settle()
        external
        stateTransition(TradeStatus.OPEN, TradeStatus.CLOSED)
        returns (uint256 soldAmt, uint256 boughtAmt)
    {
        require(msg.sender == origin, "only origin can settle");

        // Optionally process settlement of the auction in Gnosis
        if (!isAuctionCleared()) {
            // By design, we don't rely on this return value at all, just the
            // "cleared" state of the auction, and the token balances this contract owns.
            // slither-disable-next-line unused-return
            gnosis.settleAuction(auctionId);
            assert(isAuctionCleared());

//...snip
        }

```

The function comment [here](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/GnosisTrade.sol#L165) requires end time to has passed.

```solidity
    //   now >= endTime
```

However, the actual implementation does not include this check.

### Impact

QA, since this would mean a premature attempt at settling auctions

### Recommended Mitigation Steps

The `canSettle()` function already exists that ensures ` now >= endTime` consider looking for a way to integrate this to `settle()` or a similar functionality like what's in [DutchTrade#settle()](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/DutchTrade.sol#L280-L281)

```solidity
    function settle()
{
  ..snip
        require(bidder != address(0) || block.timestamp > endTime, "auction not over");
  ..snip
  }

```

## QA-07 Somewhat broken logic in regards to changing the delay for unstaking

### Proof of Concept

#### Build up

To unstake, there is a need to create the unstaking draft, see https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L259-L281

```solidity
function unstake(uint256 stakeAmount) external {
  _requireNotTradingPausedOrFrozen();
  _notZero(stakeAmount);

  address account = _msgSender();
  require(stakes[era][account] >= stakeAmount, 'insufficient balance');

  _payoutRewards();

  _burn(account, stakeAmount);

  uint256 newStakeRSR = (FIX_ONE_256 * totalStakes + (stakeRate - 1)) / stakeRate;
  uint256 rsrAmount = stakeRSR - newStakeRSR;
  stakeRSR = newStakeRSR;

  //@audit draft is being created
  (uint256 index, uint64 availableAt) = pushDraft(account, rsrAmount);
  emit UnstakingStarted(index, draftEra, account, rsrAmount, stakeAmount, availableAt);
}

```

Now take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L640-L664

```solidity
function pushDraft(address account, uint256 rsrAmount)
  internal
  returns (uint256 index, uint64 availableAt)
{
  // draftAmount: how many drafts to create and assign to the user
  // pick draftAmount as big as we can such that (newTotalDrafts <= newDraftRSR * draftRate)
  draftRSR += rsrAmount;
  // newTotalDrafts: {qDrafts} = D18{qDrafts/qRSR} * {qRSR} / D18
  uint256 newTotalDrafts = (draftRate * draftRSR) / FIX_ONE;
  uint256 draftAmount = newTotalDrafts - totalDrafts;
  totalDrafts = newTotalDrafts;

  // Push drafts into account's draft queue
  CumulativeDraft[] storage queue = draftQueues[draftEra][account];
  index = queue.length;

  uint192 oldDrafts = index != 0 ? queue[index - 1].drafts : 0;
  uint64 lastAvailableAt = index != 0 ? queue[index - 1].availableAt : 0;
  availableAt = uint64(block.timestamp) + unstakingDelay;
  //@audit
  if (lastAvailableAt > availableAt) {
    availableAt = lastAvailableAt;
  }

  queue.push(CumulativeDraft(uint176(oldDrafts + draftAmount), availableAt));
}

```

Now this function includes a few checks, one of which is to check if the account had an drafts in it's queue previously and in the case where it does, `lastAvailableAt` is being checked for the last queue.

Issue with this implementation is the fact that, in the case where `lastAvailableAt > availableAt` the `availableAt` for the current draft would be unfairly set to `lastAvailableAt`.

This approach then causes for users to be unfairly locked off from their assets, cause from here: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L40-L41

```solidity
    uint48 private constant MIN_UNSTAKING_DELAY = 60 * 2; // {s} 2 minutes
    uint48 private constant MAX_UNSTAKING_DELAY = 60 * 60 * 24 * 365; // {s} 1 year
```

We can see that the `unstakingDelay` can only be [set](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L970-L976) within, this range, and the [setUnstakingDelay()](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L970-L976) exists so the governance can update this value as/when they see fit.

Now consider this scenario:

- `unstakingDelay` is currently `3 months`.
- User pushes an unstake draft `A`.
- Two weeks later `unstakingDelay` gets reduced to `3 weeks`.
- User pushes another unstake draft `B`.

Now what happens is that instead of the user having access to their tokens from draft `B` available after `21` days, they'd have to wait a whooping `70` which is more than `x3` the current wait time.

### Coded POC

This can be shown by running the current test at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/test/ZZStRSR.test.ts#L1247-L1297

> _NB: There is a need to glance through the earlier blocks in ` describe('Withdrawals/Unstaking', () => {` to get a glimpse of the current state before this test._

```typescript
it('Should handle changes in stakingWithdrawalDelay correctly', async function () {
  // Get current balance for user 2
  const prevAddr2Balance = await rsr.balanceOf(addr2.address)

  // Create first withdrawal for user 2
  await stRSR.connect(addr2).unstake(amount2)

  // Reduce staking withdrawal delay significantly - Also need to update rewardPeriod
  const newUnstakingDelay: BigNumber = bn(172800) // 2 days

  await expect(stRSR.connect(owner).setUnstakingDelay(newUnstakingDelay))
    .to.emit(stRSR, 'UnstakingDelaySet')
    .withArgs(config.unstakingDelay, newUnstakingDelay)

  // Perform another withdrawal for user 2, with shorter period
  await stRSR.connect(addr2).unstake(amount3)

  // Move forward time past this second stake
  // Should not be processed, only after the first pending stake is done
  await advanceToTimestamp(Number(newUnstakingDelay.add(await getLatestBlockTimestamp())))

  // Check withdrawals - Nothing available yet
  expect(await stRSR.endIdForWithdraw(addr2.address)).to.equal(0)

  // Nothing completed still
  expect(await stRSR.totalSupply()).to.equal(0)
  expect(await rsr.balanceOf(addr2.address)).to.equal(prevAddr2Balance)
  // All staked funds withdrawn upfront
  expect(await stRSR.balanceOf(addr2.address)).to.equal(0)

  // Move forward past first stakingWithdrawalDelay
  await advanceToTimestamp(Number(await getLatestBlockTimestamp()) + stkWithdrawalDelay)

  //  Check withdrawals - We can withdraw both stakes
  expect(await stRSR.endIdForWithdraw(addr2.address)).to.equal(2)

  // Withdraw with correct index
  await stRSR.connect(addr2).withdraw(addr2.address, 2)

  // Withdrawals were completed
  expect(await stRSR.totalSupply()).to.equal(0)
  expect(await rsr.balanceOf(stRSR.address)).to.equal(amount1) // Still pending the withdraw for user1
  expect(await rsr.balanceOf(addr2.address)).to.equal(prevAddr2Balance.add(amount2).add(amount3))
  // All staked funds withdrawn upfront
  expect(await stRSR.balanceOf(addr2.address)).to.equal(0)

  // Exchange rate remains steady
  expect(await stRSR.exchangeRate()).to.equal(fp('1'))
})
```

#### Main issue

One can conclude that this test case: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/test/ZZStRSR.test.ts#L1247-L1297 is the intended functionality, which in short is to showcase that in a case where a user tries unstaking after a previous unstake they can't have a shorter time if they have one waiting for a long time on the queue.

But if that's the case, then a user can **completely sidestep** this via cancelling their unstake here: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L346 `cancelUnstake()` and then placing in another unstake request to sidestep the implementation.

### Impact

> NB: I had originally assessed this as _medium_, but on discussing with the sponsors they indicated that this is the intended functionality, i.e allowing the unstaker to sidestep the longer delay by cancelling, so attaching this report as _QA_.

### Recommended Mitigation Steps

A common approach to withdrawal/unstaking delays is to have them on a case by case basis, where consequent/previous withdrawals/unstakes should not affect each other and that should be the case here.

## QA-08 Consider removing storage gaps in favour of using `DiamondStorage`

### Proof of Concept

Looking at various in-scope contracts in the RToken system we see that they are deployed as ERC-1967 proxies. These contracts typically include storage gaps, such as:

```solidity
uint256[50] private __gap;
```

Now storage gaps for upgradeable contracts can be problematic, adding significant mental overhead to developers in addition to cluttering the contract code/storage layout. This technique requires a lot of attention to be paid as forgetting a gap or adding variables without adjustment can result in inadvertently modifying storage between upgrades.

### Impact

QA, but would be key to note that this approach increases the risk of storage collision during upgrades, which could lead to data corruption or unexpected behavior in the upgraded contracts.

### Recommended Mitigation Steps

A better alternative would be the use of EIP-2535 [DiamondStorage](https://medium.com/cartesi/upgrading-smart-contract-code-and-storage-layout-with-eip-2535-diamonds-31e0aff5d255) which provides a more robust framework for adding storage variables to upgradeable contracts. This approach offers several benefits:

1. It provides a more flexible and organized way to manage storage in upgradeable contracts.
2. It reduces the risk of storage collisions during upgrades.
3. It simplifies the process of adding new storage variables in future upgrades.

Additionally, DiamondStorage is [planned to be adopted by OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/2964#issuecomment-1116358040) in a future major release, which would provide better integration with widely-used contract libraries.

By adopting DiamondStorage, the development team can improve the maintainability and upgradability of the RToken system while reducing the risks associated with storage management in upgradeable contracts.

> NB: This seem to have been adopted by OZ in their v5.0

## QA-09 Invariant of having unique erc20 tokens when forwarding revenue can be broken

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BackingManager.sol#L179-L265

```solidity
function forwardRevenue(IERC20[] calldata erc20s) external nonReentrant {
  requireNotTradingPausedOrFrozen();
  //@audit
  require(ArrayLib.allUnique(erc20s), 'duplicate tokens');

  assetRegistry.refresh();

  BasketRange memory basketsHeld = basketHandler.basketsHeldBy(address(this));

  require(tradesOpen == 0, 'trade open');
  require(basketHandler.isReady(), 'basket not ready');
  require(block.timestamp >= basketHandler.timestamp() + tradingDelay, 'trading delayed');
  require(basketsHeld.bottom >= rToken.basketsNeeded(), 'undercollateralized');

  // snip
}

```

This function is used to forward revenue to `RevenueTraders` and is expected to revert if not fully collaterized.

Now among the checks it expects all the erc20s to be unique, and it uses the ArrayLib to help with these, i.e https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Array.sol#L9-L17

```solidity
function allUnique(IERC20[] memory arr) internal pure returns (bool) {
  uint256 arrLen = arr.length;
  for (uint256 i = 1; i < arrLen; ++i) {
    for (uint256 j = 0; j < i; ++j) {
      if (arr[i] == arr[j]) return false;
    }
  }
  return true;
}

```

Issue however is that from the readMe, we can see that the protocol expects to integrate any sort of ERC20, and hinting which they wouldn't support [here](), however upgradeable tokens are supported and to this end allows one to be able to sidestep all ` require(ArrayLib.allUnique(erc20s), "duplicate tokens");` both in `BackingManager#forwardRevenue()` & in [BasketHandler#requireValidCollArray()](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L861-L874) which is directly used when [setting the prime baskets](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L291) or [backup config](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L344), since any token can be upgraded to be a token with multiple addresses.

### Impact

Not having duplicate tokens in the array can be easily sidestepped when these sort of tokens are integrated, now in the case of [BasketHandler#requireValidCollArray()](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L861-L874) which is directly used when [setting the prime baskets](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L291) or [backup config](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/BasketHandler.sol#L344) we can consider this an Admin responsibility not to duplicate said token in the array by providing the same token with it's different address twice since the functionality is backed by an admin protection, however in the case of `BackingManager#forwardRevenue()` anyone can process this and forward more revenue than is expected for a particular token, breaking core functionality.

### Recommended Mitigation Steps

Consider clearly indicating that these tokens are not supported, attach this to the docs:
https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/README.md#L141-L148

```diff

### ERC20 token behaviors in scope

| Question                                                                                                                                                   | Answer |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| [Missing return values](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#missing-return-values)                                                      |   In scope  |
| [Fee on transfer](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#fee-on-transfer)                                                                  |  Out of scope  |
| [Balance changes outside of transfers](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#balance-modifications-outside-of-transfers-rebasingairdrops) | Out of scope    |
-| [Upgradeability](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#upgradable-tokens)                                                                 |   In scope  |
+| [Upgradeability](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#upgradable-tokens)                                                                 |    Out of scope  |
..snip

```

## QA-10 An admin can't set the most accurate value for a _long freeze_

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L204-L209

```solidity
/// @custom:governance
function setLongFreeze(uint48 longFreeze_) public onlyRole(OWNER) {
  require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE, 'long freeze out of range');
  emit LongFreezeDurationSet(longFreeze, longFreeze_);
  longFreeze = longFreeze_;
}

```

This function is used to set the long freeze value, issue however is that this function checks the value of the new long freeze against `MAX_LONG_FREEZE`,i.e https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L10-L11

```solidity
uint48 constant MAX_LONG_FREEZE = 31536000; // 1 year

```

Issue however is that the value for this was set without _leap_ years in mind, and as such the timeline is considerably _0.25_ days lagging, since this value represents 365 days in seconds (365 _ 24 _ 60 \* 60 = 31,536,000 seconds).

### Impact

QA, since the difference is quite negligible.

> \_NB: A somewhat similar argument can be made for the snippes below:

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BackingManager.sol#L33-L34

```solidity
    uint48 public constant MAX_TRADING_DELAY = 60 * 60 * 24 * 365; // {s} 1 year

```

- https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BasketHandler.sol#L33-L34

```solidity
    uint48 public constant MAX_WARMUP_PERIOD = 60 * 60 * 24 * 365; // {s} 1 year

```

### Recommended Mitigation Steps

Consider applying either of these changes:
https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L10-L11

1.

```diff
- uint48 constant MAX_LONG_FREEZE = 31536000; // 1 year
+ uint48 constant MAX_LONG_FREEZE = 31557600; // 1 year (365.25 days)

```

2.

```diff
- uint48 constant MAX_LONG_FREEZE = 31536000; // 1 year
+ uint48 constant MAX_LONG_FREEZE = 31536000; // 1 year (NB: No inclusion is made of leap years)

```

## QA-11 `Main#poke()` is meant for testing and should be removed before final production

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Main.sol#L51-L58

```solidity
function poke() external {
  // == Refresher ==
  assetRegistry.refresh(); // runs furnace.melt()

  // == CE block ==
  stRSR.payoutRewards();
}

```

This function is used as a refresher and per the walkthrough video for Reserve 1.2 _CC [10:30](https://youtu.be/341MhkOWsJE?t=631)_. we can see that the poke function was for testing and to prove equivalence between P1 and P0 and as such is not needed in main prod.

### Impact

QA

### Recommended Mitigation Steps

Consider removing this function from final prod.

```diff
-    function poke() external {
-        // == Refresher ==
-        assetRegistry.refresh(); // runs furnace.melt()
-
-        // == CE block ==
-        stRSR.payoutRewards();
-    }
```

## QA-12 Incorrect logical operator in auction length validation allows setting `zero auction length`

### Proof of Concept

In the `Broker.sol` contract, the `setBatchAuctionLength()` and `setDutchAuctionLength()` functions contain a logical error in their validation checks. The use of the `||` (OR) operator instead of `&&` (AND) allows for the auction length to be set to zero, which can cause subsequent auction creation functions to revert.

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Broker.sol#L197-L228

```solidity
function setBatchAuctionLength(uint48 newAuctionLength) public governance {
  require(
    newAuctionLength == 0 ||
      (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
    'invalid batchAuctionLength'
  );
  emit BatchAuctionLengthSet(batchAuctionLength, newAuctionLength);
  batchAuctionLength = newAuctionLength;
}

function setDutchAuctionLength(uint48 newAuctionLength) public governance {
  require(
    newAuctionLength == 0 ||
      (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
    'invalid dutchAuctionLength'
  );
  emit DutchAuctionLengthSet(dutchAuctionLength, newAuctionLength);
  dutchAuctionLength = newAuctionLength;
}

```

Evidently, the same issue exists in the `setDutchAuctionLength()` function:

### Impact

This logical error allows the auction length to be set to zero, which can have the following consequences:

1. The `newBatchAuction()` function will revert when trying to create a new auction with zero length.
2. The `newDutchAuction()` function will also revert when attempting to create a new auction with zero length.
3. This could potentially lead to a denial of service for auction creation if the auction length is mistakenly set to zero.

> NB: Submitted as QA as the window is admin "closed".

### Recommended Mitigation Steps

Replace the `||` (OR) operator with `&&` (AND) in both `setBatchAuctionLength()` and `setDutchAuctionLength()` functions:

```diff
    function setBatchAuctionLength(uint48 newAuctionLength) public governance {
        require(
-            newAuctionLength == 0 ||
+            newAuctionLength == 0 &&
                (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
            "invalid batchAuctionLength"
        );
        emit BatchAuctionLengthSet(batchAuctionLength, newAuctionLength);
        batchAuctionLength = newAuctionLength;
}
    function setDutchAuctionLength(uint48 newAuctionLength) public governance {
        require(
-            newAuctionLength == 0 ||
+            newAuctionLength == 0 &&
                (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
            "invalid dutchAuctionLength"
        );
        emit DutchAuctionLengthSet(dutchAuctionLength, newAuctionLength);
        dutchAuctionLength = newAuctionLength;
    }

```

Alternatively, consider simplifying the check to:

```solidity
require(
    newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH,
    "invalid auctionLength"
);
```

## QA-13 Remove misleading comments from production

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Fixed.sol#L40-L50

```solidity
// If a particular uint192 is represented by the uint192 n, then the uint192 represents the
// value n/FIX_SCALE.
uint64 constant FIX_SCALE = 1e18;

// FIX_SCALE Squared:
uint128 constant FIX_SCALE_SQ = 1e36;

// The largest integer that can be converted to uint192 .
// This is a bit bigger than 3.1e39//@audit false
uint192 constant FIX_MAX_INT = type(uint192).max / FIX_SCALE;

```

This is from the native `Fixed.sol`, issue here is that there is wrong assumption on the value of ` type(uint192).max / FIX_SCALE` where we have the latter to be `1e18`, it's hinted that the value is a bit bigger than `3.1e39` however running the POC below we see that the value of `FIX_MAX_INT` is actually over double `3.1e39`_(6.2e39)_

Coded POC:

Run the python script below

```py
from decimal import Decimal, getcontext

# Set the precision for the Decimal calculations
getcontext().prec = 100  # Adjust precision as needed

# Calculate the maximum value of uint192
uint192_max = Decimal(2**192 - 1)

# Perform the division
result = uint192_max / Decimal('1e18')

# Check if result is greater than (3.1e390 * 2)
if result > Decimal('6.2e39'):
    print("Result is greater than 6.2e39")
else:
    print("Result is not greater than 6.2e39")

print(result)

```

we get the result:

```rust
Result is greater than 6.2e39
6277101735386680763835789423207666416102.355444464034512895
```

### Impact

QA, confusing code

### Recommended Mitigation Steps

Consider applying these changes to https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Fixed.sol#L40-L50

```diff
// If a particular uint192 is represented by the uint192 n, then the uint192 represents the
// value n/FIX_SCALE.
uint64 constant FIX_SCALE = 1e18;

// FIX_SCALE Squared:
uint128 constant FIX_SCALE_SQ = 1e36;

// The largest integer that can be converted to uint192 .
-// This is a bit bigger than 3.1e39//
+// This is a bit bigger than 6.2e39//
uint192 constant FIX_MAX_INT = type(uint192).max / FIX_SCALE;

```

## QA-14 Precision issue in BasketHandler.quote function

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/BasketHandler.sol#L482-L512

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
    uint192 q = _quantity(basket.erc20s[i], coll, rounding).safeMul(amount, rounding); //@audit

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

In the `BasketHandler` contract, there's a precision issue in the `quote()` function. Unlike the `quoteCustomRedemption()` function which uses `safeMulDivFloor()` for calculating quantities, the `quote()` function uses `safeMul()` after the division involved in the `_quantity()` function call.

### Impact

QA, but would be key to note that the slight inaccuracies in the calculated quantities of tokens for basket composition.

### Recommended Mitigation Steps

Consider implementing changes that have the multiplication before division.

## QA-15 `DutchTrade`: Inconsistent time check in `canSettle()` & `settle()`

### Proof of Concept

In the `DutchTrade` contract, there's an inconsistency in how the auction end time is checked between the `canSettle()` and `settle()` functions.

See https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/DutchTrade.sol#L305-L307

```solidity
function canSettle() external view returns (bool) {
  return status == TradeStatus.OPEN && (bidder != address(0) || block.timestamp > endTime);
}

```

The condition uses a strict inequality (`>`) to check if the current time has passed the `endTime`.

However, in the `settle()` function: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/plugins/trading/DutchTrade.sol#L273-L293

```solidity
function settle()
  external
  stateTransition(TradeStatus.OPEN, TradeStatus.CLOSED)
  returns (uint256 soldAmt, uint256 boughtAmt)
{
  require(msg.sender == address(origin), 'only origin can settle');
  require(bidder != address(0) || block.timestamp >= endTime, 'auction not over');

  // ... snip
}

```

The condition uses a greater than or equal to (`>=`) comparison.

### Impact

This inconsistency could lead to a brief window where `canSettle()` returns `false`, but `settle()` can be successfully called, i.e confusion for developers or integrators who might expect these functions to be perfectly aligned.

> _NB: While the practical impact is minimal (affecting only the exact second of `endTime`), it represents an inconsistency in the contract logic that could lead to unexpected behavior or confusion._

### Recommended Mitigation Steps

To resolve this inconsistency, align the time checks in both functions. Given that `settle()` is the actual function executing the settlement, it's generally safer to use its more inclusive check. Therefore, update the `canSettle()` function to match, so apply these changes

```diff
    function canSettle() external view returns (bool) {
-        return status == TradeStatus.OPEN && (bidder != address(0) || block.timestamp > endTime);
+        return status == TradeStatus.OPEN && (bidder != address(0) || block.timestamp >= endTime);
    }
```

## QA-16 Wrong implementation of the storage gap in `Auth.sol`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L219-L225

```solidity


    uint256[48] private __gap;
```

This is the empty reserved space that's put in place to allow future versions to add new variables without shifting down storage in the inheritance chain.

Per [documentation](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L223) in scope, this has been inherited from Openzeppelin, i.e https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps.

Issue however is that going to the documentation under Openzeppelin and how this approach has been commonly used in the web3 space, we are to have the storage _left_ plus the storage _already used_ = _50_, however this has been wrongly implemented here, considering all previously declares used slots are not being taken into account for the value of _gap_ used.

### Impact

QA

> NB: A similar case can be made for this instance:
> https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/RToken.sol#L537-L545

```solidity

    /**
     * @dev This empty reserved space is put in place to allow future versions to add new
     * variables without shifting down storage in the inheritance chain.
     * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
     *
     * RToken uses 56 slots, not 50.
     */
    uint256[42] private __gap;
```

### Recommended Mitigation Steps

Consider correctly following the guidelines set by Openzeppelin [here](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps) as is currently correctly done in [ComponentRegistry.sol#L118](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/ComponentRegistry.sol#L118).

## QA-17 Trade slippage can never be set to the accepted max

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/mixins/Trading.sol#L145-L151

```solidity
function setMaxTradeSlippage(uint192 val) public governance {
  require(val < MAX_TRADE_SLIPPAGE, 'invalid maxTradeSlippage');
  emit MaxTradeSlippageSet(maxTradeSlippage, val);
  maxTradeSlippage = val;
}

```

This function is used to set the max trade slippage value, issue however is that the check is not inclusive which would then make an attempt of setting `maxTradeSlippage` = `MAX_TRADE_SLIPPAGE` would now revert.

### Impact

Low, inability to set the trade slippage to the accepted max.

### Recommended Mitigation Steps

Consider applying these changes:

```diff

    function setMaxTradeSlippage(uint192 val) public governance {
-        require(val < MAX_TRADE_SLIPPAGE, "invalid maxTradeSlippage");
+        require(val <= MAX_TRADE_SLIPPAGE, "invalid maxTradeSlippage");
        emit MaxTradeSlippageSet(maxTradeSlippage, val);
        maxTradeSlippage = val;
    }

```

Like is done here: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Broker.sol#L197-L205

```solidity
function setBatchAuctionLength(uint48 newAuctionLength) public governance {
  require(
    newAuctionLength == 0 ||
      (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
    'invalid batchAuctionLength'
  );
  emit BatchAuctionLengthSet(batchAuctionLength, newAuctionLength);
  batchAuctionLength = newAuctionLength;
}

```

## QA-18 Setters don't have equality checkers

### Proof of Concept

Multiple instances in scope for example take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Broker.sol#L177-L228

```solidity
/// @custom:governance
function setGnosis(IGnosis newGnosis) public governance {
  require(address(newGnosis) != address(0), 'invalid Gnosis address');

  emit GnosisSet(gnosis, newGnosis);
  gnosis = newGnosis;
}

/// @custom:main
function setBatchTradeImplementation(ITrade newTradeImplementation) public onlyMain {
  require(
    address(newTradeImplementation) != address(0),
    'invalid batchTradeImplementation address'
  );

  emit BatchTradeImplementationSet(batchTradeImplementation, newTradeImplementation);
  batchTradeImplementation = newTradeImplementation;
}

/// @custom:governance
function setBatchAuctionLength(uint48 newAuctionLength) public governance {
  require(
    newAuctionLength == 0 ||
      (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
    'invalid batchAuctionLength'
  );
  emit BatchAuctionLengthSet(batchAuctionLength, newAuctionLength);
  batchAuctionLength = newAuctionLength;
}

/// @custom:main
function setDutchTradeImplementation(ITrade newTradeImplementation) public onlyMain {
  require(
    address(newTradeImplementation) != address(0),
    'invalid dutchTradeImplementation address'
  );

  emit DutchTradeImplementationSet(dutchTradeImplementation, newTradeImplementation);
  dutchTradeImplementation = newTradeImplementation;
}

/// @custom:governance
function setDutchAuctionLength(uint48 newAuctionLength) public governance {
  require(
    newAuctionLength == 0 ||
      (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
    'invalid dutchAuctionLength'
  );
  emit DutchAuctionLengthSet(dutchAuctionLength, newAuctionLength);
  dutchAuctionLength = newAuctionLength;
}

```

All these functions are used as setters, however there are no checks that the current value being set != the previously stored value.

### Impact

QA

### Recommended Mitigation Steps

Consider applying equality checkers.

## QA-19 `FIX_MAX_INT` is wrongly set

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Fixed.sol#L40-L50

```solidity
// If a particular uint192 is represented by the uint192 n, then the uint192 represents the
// value n/FIX_SCALE.
uint64 constant FIX_SCALE = 1e18;

// FIX_SCALE Squared:
uint128 constant FIX_SCALE_SQ = 1e36;

// The largest integer that can be converted to uint192 .
// This is a bit bigger than 3.1e39//@audit false
uint192 constant FIX_MAX_INT = type(uint192).max / FIX_SCALE;

```

This is from the native `Fixed.sol`, issue here is that Fix max int is wrongly set, see this Coded POC:

> Run the python script below

```py
from decimal import Decimal, getcontext

# Set the precision for the Decimal calculations
getcontext().prec = 100  # Adjust precision as needed

# Calculate the maximum value of uint192
uint192_max = Decimal(2**192 - 1)

# Perform the division
result = uint192_max / Decimal('1e18')

print(result)


```

we get the result: `6277101735386680763835789423207666416102.355444464034512895`

Which means that per solidity's rounding down spec, `FIX_MAX_INT` is `(1e18 * 355444464034512895)` < the real value of type`(uint192).max / FIX_SCALE`.

### Impact

QA, cause this var is not been used as a limiting check in scope.

### Recommended Mitigation Steps

Consider correctly attaching the possible max.

## QA-20 `PermitLib#requireSignature()` could fail even for valid signatures from counterfactual wallets

### Proof of Concept

[EIP 1271](https://eips.ethereum.org/EIPS/eip-1271) is being implemented in protocol, albeit via the `SignatureCheckerUpgradeable` inherited from openzeppelin which allows contracts to sign messages and works great in tandem with EIP 4337 (account abstraction).

For ERC-4337, the account is not deployed until the first UserOp is mined, previous to this, the account exists "counterfactually" — it has an address even though it's not really deployed. Usually, this is great since we can use the counterfactual address to receive assets without deploying the account first. Now, not being able to sign messages from counterfactual contracts/accounts has always been a limitation of [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) since one can't call the `_isValidSignature()` function on them.

Here is the current method of validation: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Permit.sol#L18-L22

```solidity
        if (AddressUpgradeable.isContract(owner)) {
            require(
                IERC1271Upgradeable(owner).isValidSignature(hash, abi.encodePacked(r, s, v)) ==
                    0x1626ba7e,
                "ERC1271: Unauthorized"
            );
```

Which calls the below via `SignatureChecker`

```solidity
function isValidSignatureNow(
  address signer,
  bytes32 hash,
  bytes memory signature
) internal view returns (bool) {
  (address recovered, ECDSA.RecoverError error, ) = ECDSA.tryRecover(hash, signature);
  return
    (error == ECDSA.RecoverError.NoError && recovered == signer) ||
    isValidERC1271SignatureNow(signer, hash, signature);
}

function isValidERC1271SignatureNow(
  address signer,
  bytes32 hash,
  bytes memory signature
) internal view returns (bool) {
  (bool success, bytes memory result) = signer.staticcall(
    abi.encodeCall(IERC1271.isValidSignature, (hash, signature))
  );
  return (success &&
    result.length >= 32 &&
    abi.decode(result, (bytes32)) == bytes32(IERC1271.isValidSignature.selector));
}

```

Which practically means that Reserve will fail to validate signatures for users of notorious wallets/projects, including Safe, Sequence, and Argent supporting ERC1271, but also ERC4337 wallets, even though a clear intention has been made to support signatures by EIP1271 compliant wallets, as confirmed by using the Eip-1271 method of validating signatures.

Crux of the issue is the fact that protocol is taking responsibility to check the validity of signatures, but partially failing to trigger signature validation signatures for a group of wallets from a provider since _(the validation will succeed if the ERC4337 wallet is deployed)_ and given that the protocol decided to support contract-based wallets (that support ERC4337) and implement ERC1271, one could assume that they "inherit" the possibility from the wallet providers to support ERC4337 too.

### Impact

This vulnerability easily impacts Reserve's ability to validate signatures for counterfactual ERC-4337 accounts, limiting the usability for users of certain wallets that rely on AA, leading to the limitation of functionalities in the protocol, since all operations that need the signatures attached to the typehashes wouldn't work for some set of users, i.e the availability of these functions is impacted.

Considering this has been clearly documented: https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Permit.sol#L7

```solidity
/// Internal library for verifying metatx sigs for EOAs and smart contract wallets
```

This then means that some of the protocol would be unable to validate real signatures even from users who are expected to integrate with the protocol since the revert from `PermitLib#requireSignature()` would bubble back up to all instances where it's been used across protocol, for e.g in [StRSR.sol#L937](https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/StRSR.sol#L937).

### Recommended Mitigation Steps

[EIP-6492](https://eips.ethereum.org/EIPS/eip-6492) solves this issue. The author of the EIP already implemented ERC-6492 in a universal library which verifies 6492, 712, and 1271 sigs with this pull request.

ERC6492 assumes that the signing contract will normally be a contract wallet, but it could be any contract that implements [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) and is deployed counterfactually.

- If the contract is deployed, produce a normal [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) signature
- If the contract is not deployed yet, wrap the signature as follows: `concat(abi.encode((create2Factory, factoryCalldata, originalERC1271Signature), (address, bytes, bytes)), magicBytes)`

Reserve could adopt the [reference-implementation](https://eips.ethereum.org/EIPS/eip-6492#reference-implementation) as stated in the EIP and delegate the responsibility of supporting counterfactual signatures to the wallets themselves, and this works because, as defined in the EIP, the wallet should return the magic value in both cases.

## QA-21 `Auth#setLongFreeze()` should include more checks

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/mixins/Auth.sol#L205-L210

```solidity
function setLongFreeze(uint48 longFreeze_) public onlyRole(OWNER) {
  require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE, 'long freeze out of range');
  emit LongFreezeDurationSet(longFreeze, longFreeze_);
  longFreeze = longFreeze_;
}

```

This function is used to set the time for the `longFreeze` issue however is that there is no check to ensure that `setLongFreeze()` is not set to be less than `shortFreeze`, which defeats the purpose of having both.

### Impact

QA, _this can validly be seen as admin error._

### Recommended Mitigation Steps

Consider adding a check so that `longFreeze` is always larger than `shortFreeze` or that it is larger than `MAX_SHORT_FREEZE`.

```diff
    function setLongFreeze(uint48 longFreeze_) public onlyRole(OWNER) {
-        require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE, "long freeze out of range");
+        require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE && longFreeze_ > MAX_SHORT_FREEZE , "long freeze out of range");
        emit LongFreezeDurationSet(longFreeze, longFreeze_);
        longFreeze = longFreeze_;
    }

```

## QA-22 Remove unused imports in `Main.sol`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Main.sol#L4-L16

```solidity
import '@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol';
import '@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol';
import '@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol';
import '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import '../interfaces/IMain.sol';
import '../mixins/ComponentRegistry.sol';
import '../mixins/Auth.sol';
import '../mixins/Versioned.sol';
import '../registry/VersionRegistry.sol';
import '../registry/AssetPluginRegistry.sol';
import '../registry/DAOFeeRegistry.sol';
import '../interfaces/IBroker.sol';

```

All of these imports are used asides `OwnableUpgradeable` & `IBroker` which can be removed.

### Impact

QA, NC

### Recommended Mitigation Steps

Consider applying these changes, especially for the `OwnableUpgradeable`

```diff
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
- import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "../interfaces/IMain.sol";
import "../mixins/ComponentRegistry.sol";
import "../mixins/Auth.sol";
import "../mixins/Versioned.sol";
import "../registry/VersionRegistry.sol";
import "../registry/AssetPluginRegistry.sol";
import "../registry/DAOFeeRegistry.sol";
- import "../interfaces/IBroker.sol";

```

## QA-23 Consider using `_disableInitializers()` for upgradeable contracts

### Proof of Concept

This is rampant accross contracts in scope but take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Main.sol#L28-L31

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
// solhint-disable-next-line no-empty-blocks
constructor() initializer {}

```

Instead of using an empty constructor with the `initializer` modifier, OpenZeppelin recommendation is to use `_disableInitializers()` inside the constructor.

This is because with this, you now have the reinitializers support.

And by putting this in the constructor, we prevent initialization of the implementation contract itself, as extra protection to prevent an attacker from initializing it.

See this [post](https://forum.openzeppelin.com/t/what-does-disableinitializers-function-mean/28730) for more information.

Additionally you can see this [post](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable) too on `Initializable`, where this has been stated.

### ⚠️ Caution

> Avoid leaving a contract uninitialized.
> An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the `_disableInitializers` function in the constructor to automatically lock it when it is deployed.

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
  _disableInitializers();
}

```

### Impact

QA, contracts are left uninitialized.

### Recommended Mitigation Steps

Consider using `_disableInitializers()` for upgradeable contracts as suggested by OZ

## QA-24 Import declarations should import specific identifiers, rather than the whole file

### Proof of Concept

Multiple instances in scope, for example take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p1/Main.sol#L4-L16

```solidity
import '@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol';
import '@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol';
import '@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol';
import '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import '../interfaces/IMain.sol';
import '../mixins/ComponentRegistry.sol';
import '../mixins/Auth.sol';
import '../mixins/Versioned.sol';
import '../registry/VersionRegistry.sol';
import '../registry/AssetPluginRegistry.sol';
import '../registry/DAOFeeRegistry.sol';
import '../interfaces/IBroker.sol';

```

Evidently, the imports being done is not name specific, but this is not the best implementation cause this could lead to polluting the symbol namespace.

### Impact

QA, albeit this could lead to the potential pollution of the symbol namespace and a slower compilation speed.

### Recommended Mitigation Steps

Consider using import declarations of the form `import {<identifier_name>} from "some/file.sol"` which avoids polluting the symbol namespace making flattened files smaller, and speeds up compilation (but does not save any gas)

## QA-25 Do not import deprecated files

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/p0/StRSR.sol#L5

```solidity
import '@openzeppelin/contracts-upgradeable/utils/cryptography/draft-EIP712Upgradeable.sol';

```

However, EIP712 is no longer a draft & this the file is deprecated.

### Impact

QA, NC

### Recommended Mitigation Steps

Consider fixing this

## QA-26 Attach an overflow protection in `Fixed#plus()`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-07-reserve/blob/825e5a98a8c94a7b4458b3b359a507edb9e662ba/contracts/libraries/Fixed.sol#L220-L225

```solidity
///
/// @return x + y
// as-ints: x + y
function plus(uint192 x, uint192 y) internal pure returns (uint192) {
  return x + y;
}

```

This function is used to add a `uint192` to another `uint192`, issue however is that there is no check that the result of the addition is < `uint192`, which would lead to reverts when that's the case.
Run this test POC on remix

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

contract OverflowPOC {
  // Simulating the FixLib.plus function
  function plus(uint192 x, uint192 y) public pure returns (uint192) {
    return x + y;
  }

  // Function to get max uint192 value
  function getMaxUint192() public pure returns (uint192) {
    return type(uint192).max;
  }

  // Function to test overflow
  function testOverflow() public pure returns (uint192) {
    uint192 x = type(uint192).max;
    uint192 y = 1;
    return plus(x, y); // This will revert
  }
}

```

NB: _This function is heavily used across Reserve Core_

### Impact

QA

### Recommended Mitigation Steps

Consider reverting with a descriptive error message when the value is > `uint192`.

## QA-27 Links in core documentation should not lead to crashing pages

### Proof of Concept

We've been hinted the official page for the protocol's docs here: https://reserve.org/protocol/, by the C4 team via the readMe for the audit.

However on going to this link, under smart contracts: https://reserve.org/protocol/smart_contracts/, on clicking the _developer documentation_, the page crashes, i.e:

![Image](https://ibb.co/Q6gPF9X)
![Image](https://i.ibb.co/JQPYj0R)

### Impact

QA

### Recommended Mitigation Steps

Consider fixing valid links in docs or removing it as a whole.
