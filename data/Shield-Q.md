## 1. `shifLeft == 39` condition in the `Fixed.shiftl_toFix` function will overflow the `FIX_MAX` value 

In the `Fixed.shiftl_toFix` function the following logic implementation is given:

```solidity
    if (40 <= shiftLeft) revert UIntOutOfBounds(); // 10**56 < FIX_MAX < 10**57

    shiftLeft += 18; //@audit-info - account for the fixed arithmetic decimal amount

    uint256 coeff = 10**abs(shiftLeft);
```

As per the natspec it is clear `10**56 < FIX_MAX < 10**57`. The `40 <= shiftLeft` allows the `shifLeft == 39` which addup to `shiftLeft = 57` after the addition by `+18`. 

This makes the `coeff = 10**57` which is `> FIX_MAX` and this should not be an allowed state.

As a result the `x * coeff` will overflow since it is ` > FIX_MAX`.

```solidity
    uint256 shifted = (shiftLeft >= 0) ? x * coeff : _divrnd(x, coeff, rounding);
```

Hence recommended to update the `40 <= shiftLeft` check as follows:

```solidity
    if (39 <= shiftLeft) revert UIntOutOfBounds();
```

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/libraries/Fixed.sol#L103-L108

## 2. The `DAOFeeRegistry.setRTokenFeeNumerator` can be called with `feeNumerator_ = 0` thus effectively resetting the rTokenFee parameters with wrong state 

In the `DAOFeeRegistry.setRTokenFeeNumerator` function the `feeNumerator_` can be `0` and it is not explicitly checked. 

The issue here is if the passed in `feeNumerator_` is `0` then the following state would occur.

```solidity
        rTokenFeeNumerator[rToken] =  0;
        rTokenFeeSet[rToken] = true;
```

But this is not the intended behaviour since in the `getFeeDetails.resetRTokenFee` it is clear that when a reset occurs the desired state is the following:

```solidity
        rTokenFeeNumerator[rToken] = 0;
        rTokenFeeSet[rToken] = false;
```

When the `getFeeDetails.getFeeDetails` is called the wrong state will be returned which will effect the `rsrTotal` calculation thus making the revenue distribution erroneous.

Hence it is recommended to revert the `DAOFeeRegistry.setRTokenFeeNumerator` transaction if the `feeNumerator_ == 0`.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/registry/DAOFeeRegistry.sol#L66-L81

## 3. In the `Auth.setLongFreeze` function there is no check to ensure that `longFreeze_ > MAX_SHORT_FREEZE`

The `Auth.setShortFreeze` and `Auth.setLongFreeze` functions are used to set the `shortFreeze` and `longFreeze` values respectively. 

But the issue is when the `longFreeze` is set there is no check in place to ensure that `longFreeze_ > MAX_SHORT_FREEZE`. 

```solidity
require(longFreeze_ != 0 && longFreeze_ <= MAX_LONG_FREEZE, "long freeze out of range");
```

Hence a `longFreeze` can be set to a value less than `MAX_SHORT_FREEZE` value which is not the intended behavior of this functionality which has seperate bounds for the `shortFreeze` and `longFreeze`.

Hence it is recommended to add a check to ensure that `longFreeze_ > MAX_SHORT_FREEZE` when the `longFreeze` is set in the `setLongFreeze` function.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/mixins/Auth.sol#L198-L209

## 4. Not enough input validation checks for the parameters of the `TradePrices` struct, in the `DutchTrade.init` function 

The `DutchTrade.init` function uses the `TradePrices` struct to calculate the `worstPrice` and `bestPrice` for the auction. The following input validations are inplace for the `TradePrices struct parameters`.

```solidity
require(prices.sellLow != 0 && prices.sellHigh < FIX_MAX / 1000, "bad sell pricing"); 
require(prices.buyLow != 0 && prices.buyHigh < FIX_MAX / 1000, "bad buy pricing");
```

But above checks are not enough since the `TradePrices` struct declaration states the neccesity of further input validation.

```solidity
struct TradePrices {
    uint192 sellLow; // {UoA/sellTok} can be 0
    uint192 sellHigh; // {UoA/sellTok} should not be 0
    uint192 buyLow; // {UoA/buyTok} should not be 0
    uint192 buyHigh; // {UoA/buyTok} should not be 0 or FIX_MAX
}
```

Hence it is required to perform `!= 0` check on the `buyHigh and sellHigh` parameters as well as stated in the `TradePrices struct` declaration. Further the `buyHigh != 0` check is important since during the `_worstPrice` calculation division by `buyHigh` happens.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/interfaces/IBroker.sol#L16-L21
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L168-L169

## 5. The `_startTime` calculation in the `DutchTrade.init` function is not correct

The `DutchTrade.init` function is used to initialize the `proxy trade contract`. In this function execution the `startTime` and `endTime` are updated as follows:

```solidity
uint48 _startTime = uint48(block.timestamp) + 1; // cannot fulfill in current block
startTime = _startTime; // gas-saver
endTime = _startTime + auctionLength;
```

The `_startTime` is set as the `uint48(block.timestamp) + 1` to ensure that the trade can not be settled within the same block as it was initiated. 

As per the natspec comment given for the `startTime` the following is mentioned.

```solidity
uint48 public startTime; // {s} when the dutch auction begins (one block after init()) lossy!
```

Hence the `dutch auction` is expected to begin from the `next block` onwards. But the `initialize` function starts the `auction from the next seconds onwards`.

Since Dutch auction reduces the `bid price` from high to low price, this discrepancy in the `startTime` of the auction could lead to premature start of the auction by `11 seconds (assuming 12 seconds block duation)`. Hence by the time the auction starts the auction price would already have been reduced.

Hence it is recommended to set the `startTime` as `uint48(block.timestamp) + blockDuration` instead of `uint48(block.timestamp) + 1` to ensure that the dutch auction begins from one block after init(), where the `blockDuration` is the block duration of the respective block chain. There needs to be additional logic in the protocol to retrieve the `block duration` of the respective block chain.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L180
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L110

## 6. In the `DutchTrade.bidWithCallback` the bidder is allowed to send in more `buy token amount` during the `callback functionc call` thus incurring a loss to himself

In the `DutchTrade.bid` function the `bidder` is sending only the `amountIn` (buy token amount) and then this `buy token amount` is transferred to the `origin contract (BackingManager)`.

But in the `DutchTrade.bidWithCallback` the bidder is allowed to send in more `buy token amount` during the `callback functionc call` as shown below:

```solidity
        IDutchTradeCallee(bidder).dutchTradeCallback(address(buy), amountIn, data);
require(
            amountIn <= buy.balanceOf(address(this)) - balanceBefore,
            "insufficient buy tokens"
        );
```

Due to this the `excessive funds` sent in will also be transferred to the `origin` contract and will be a loss to the `bidder` since he is overpaying for the `sell token amount`. This could happen by mistake during bidding process.

Hence it is recommended to let the `bidder` pass in only the `buy token amount` in the `DutchTrade.bidWithCallback` function as well.

The `require` statement can be modified as shown below:

```solidity
require(
            amountIn == buy.balanceOf(address(this)) - balanceBefore,
            "insufficient buy tokens"
        );
```

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L258-L262

## 7. A user can distribute the `full buyToken balance of the RevenueTrader contract` to the `destinations`, by calling the `_distributeTokenToBuy` function multiple times

The `Distributor.distribute` function is used to `distribute` either the `RToken` or the `rsrToken` to the `respective destinations`. 

The amount to transfer to each `destination address` is calculated as follows:

```solidity
if (totalShares != 0) tokensPerShare = amount / totalShares; 
```

```solidity
uint256 transferAmt = tokensPerShare * numberOfShares;
```

As it is evident from above the `tokensPerShare` is susceptible to rounding down thus when multiplied by the `numberOfShares` will not always transfer the full `buyToken balance of the RevenueTrader contract` to the `destinations`. 

The protocol says this is the intended behaviour even though it is loss of funds to the `destinations`, as per the following natspec comment:

```solidity
// This rounds "early", and that's deliberate!
``` 

But a user can always get the `remaining balance` distributed as follows:

    1. User can do a pre calculation of the remaining balance after calling the `distribute` function.

    2. User can determine how many times to call the `distribute` function to get as much balance of the `revenueTrader` contract as possible. Let's assume this number to be 3.

    3. User calls the `RevenueTrader.manageTokens` with the `erc20s` array being filled with `rsrToken` three times (3 duplicate entries).

    4. Since there is no check on the `unique elements of the array` the  following code snippet will call the `_distributeTokenToBuy` function 3 times.

    ```solidity
    for (uint256 i = 0; i < len; ++i) {
    if (erc20s[i] == IERC20(address(rToken))) involvesRToken = true;
    if (erc20s[i] == tokenToBuy) {
                    _distributeTokenToBuy();
    if (len == 1) return; // return early if tokenToBuy is only entry
                }
    ```

    5. This will ensure the `distribute` function is called 3 times thus distributing the remaining balance among the `destination addresses`.

It is recommended to check for the duplicate entries of the `erc20s` array when passed to the `RevenueTrader.manageTokens` to eliminate the above issue. 

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/RevenueTrader.sol#L127-L133
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Distributor.sol#L133
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Distributor.sol#L152
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Distributor.sol#L137

## 8. Not enough trade parameter validation checks in the `RevenueTrader.manageTokens` function

The `RevenueTrader.manageTokens` function is used to convert the given set of `erc20 tokens` into the `buyToken`.

During the execution flow of the function the following operations are performed to ensure that the price of the `buyToken` is valid.

```solidity
(uint192 buyLow, uint192 buyHigh) = assetToBuy.price(); // {UoA/tok}
require(buyHigh != 0 && buyHigh != FIX_MAX, "buy asset price unknown");
```

As it is shown above the `buyLow` values is not checked in the `require` statement. The `buyLow` value in the tradePrices can not be zero and it is stated in the natspec comments of the `IBroker.TradePrices` struct as shown below:

```solidity
struct TradePrices {
    uint192 sellLow; // {UoA/sellTok} can be 0
    uint192 sellHigh; // {UoA/sellTok} should not be 0
    uint192 buyLow; // {UoA/buyTok} should not be 0
    uint192 buyHigh; // {UoA/buyTok} should not be 0 or FIX_MAX
}
```

Hence it is recomended to update the `require statement` as shown below:

```solidity
require(buyLow != 0 && buyHigh != 0 && buyHigh != FIX_MAX, "buy asset price unknown");
```

Similarly when the price of the `assetToSell` is retrieved there is no check inplace to validate the `sellLow` and `sellHigh` prices. It is required that `sellHigh` must not be zero and this check should be implemented after the `assetToSell.price()` call.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/RevenueTrader.sol#L149-L150
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/interfaces/IBroker.sol#L16-L21

## 9. The `RevenueTrader.manageTokens` function should check the input `erc20 tokens array` for duplicate entries

The `RevenueTrader.manageTokens` function is used to convert the given set of `erc20 tokens` to the `tokenToBuy token` to be distributed among the `destinations`.

Anyone can call the `manageTokens` function with the required set of `ERC20 token set` to be managed. 

The issue here is that there is no check to ensure `passed in erc20 token array has all the unique erc20 entries`. If there are duplicate entries in the array the transaction will revert at the end of the `manageTokens` function execution thus wasting significant amount gas (considering the complexity of the function).

For example the duplicate entries would revert at the following line of code in the `manageTokens` function

```solidity
require(address(trades[erc20]) == address(0), "trade open");
```

Hence it is recommended to call the `ArrayLib.allUnique` function by passing in the `erc20 token array` at the start of the `RevenueTrader.manageTokens` function to ensure there are no duplicate entries

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/RevenueTrader.sol#L157
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/RevenueTrader.sol#L109

## 10. Disagreement between the `natspec` and the logic implementation in the `BackingManager.rebalance` function

In the `BackingManager.rebalance` function the following logic is implemented.

```solidity
// First dissolve any held RToken balance (above Distributor-dust)
// gas-optimization: 1 whole RToken must be worth 100 trillion dollars for this to skip $1

uint256 balance = rToken.balanceOf(address(this)); //@audit-info - MAX_DISTRIBUTION = 1e4; and MAX_DESTINATIONS = 100;
if (balance >= MAX_DISTRIBUTION * MAX_DESTINATIONS) rToken.dissolve(balance);
```

The `natspec` states that the `dissolve any held RToken balance (above Distributor-dust)`. But in the implementation the entire `balance is desolved`. Hence there is a discrepancy between the `natspec` and the logic implementation.

Hence it is recommended to update the `natspec` to reflect the logic implementation.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BackingManager.sol#L128-L132

## 11. The declared assumption is not checked within the `BackingManager.forwardRevenue` function scope
 
In the `BackingManager.forwardRevenue` function the following assumption is given:

```solidity
   *   - Neither RToken nor RSR are in the basket
```

The above assumption is in place for the `erc20s used for revenue forwarding`. But the above assumption is not explicitly checked in the `BackingManager.forwardRevenue` function where as the all the other declared assumptions are checked within the function scope.

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BackingManager.sol#L197

## 12. Discrepency in code & comments

### Lines of Code
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L37 

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/trading/DutchTrade.sol#L165 

### Description
In `DutchTrade.sol` it is mentioned that `30-minutes is the recommended length of auction for a chain with 12-second blocktimes`

but in `init` method this check
```
assert(address(sell_) != address(0) && address(buy_) != address(0) && auctionLength >= 60);
```
allows the auction length to be a min of 60 secs which contradicts the comment


### Recommended Mitigation Steps
```
assert(address(sell_) != address(0) && address(buy_) != address(0) && auctionLength >= 60);
``` 
should be updated to
```
assert(address(sell_) != address(0) && address(buy_) != address(0) && auctionLength >= 1700 && auctionLength <= 1900);
```
so the min auction length is always close to 30 mins

## 13. missing check in `setBackupConfig` affects 100 % utilisation of backup collateral assets


### Lines of Code
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BasketHandler.sol#L284

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/mixins/BasketLib.sol#L252

### Description
In `setBackupConfig` we set the `BackupConfig` struct which has 2 params 
```
struct BackupConfig {
    uint256 max; // Maximum number of backup collateral erc20s to use in a basket
    IERC20[] erc20s; // Ordered list of backup collateral ERC20s
}
```

so `max` should always be `=> erc20s.length`  but there is no such check in `setBackupConfig`

now this can also affect the `nextBasket` backup weight logic as there is a for loop with multiple conditions to handle backup weights

```
            for (uint256 j = 0; j < backup.erc20s.length && size < backup.max; ++j) {
                if (goodCollateral(_targetName, backup.erc20s[j], assetRegistry)) size++;
            }
```

now let's assume that `size > backup.erc20s.length` & all backup collaterals are good collaterals so therefore once the loop ends size will be `max - 1` but ideally it should be `backup.erc20s.length` and since the weights for backup tokens are divided equally therefore some backup assets can remain un-utilised


### Recommended Mitigation Steps
Add the following check in `setBackupConfig`

```
require(erc20s.length <= max, "Invalid backup config");
```

## 14. Discrepancy in code & comments

### Lines of Code
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BasketHandler.sol#L214

https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BasketHandler.sol#L237

### Description
In `_setPrimeBasket` there is a comment 
`/// @param disableTargetAmountCheck If true, skips the `requireConstantConfigTargets()` check`

but actually even if `disableTargetAmountCheck` is true & `reweightable` is false then too `requireConstantConfigTargets` is called


### Recommended Mitigation Steps
Update the comment to mention both scenario's where `requireConstantConfigTargets` is called/skipped