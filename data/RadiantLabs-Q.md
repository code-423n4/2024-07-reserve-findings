## [L-1] Asymmetric decay of Asset saved priced distorts average price

### Links to affected code
https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/plugins/assets/Asset.sol#L150-L163

### Description

The Asset contract implements a graceful fallback for being able to provide underlying asset prices when the upstream oracle is unreachable.

This fallback uses the last known price, but artificially widens its range (`low-high`) to offset the uncertainty over the stale price.

The math of how this offset is done is implemented with this code;
```Solidity
File: Asset.sol
150:                 // Decay _high upwards to 3x savedHighPrice
151:                 // {UoA/tok} = {UoA/tok} * {1}
152:                 _high = savedHighPrice.safeMul(
153:                     FIX_ONE + MAX_HIGH_PRICE_BUFFER.muluDivu(delta - decayDelay, priceTimeout),
154:                     ROUND
155:                 ); // during overflow should not revert
156: 
157:                 // if _high is FIX_MAX, leave at UNPRICED
158:                 if (_high != FIX_MAX) {
159:                     // Decay _low downwards from savedLowPrice to 0
160:                     // {UoA/tok} = {UoA/tok} * {1}
161:                     _low = savedLowPrice.muluDivu(decayDelay + priceTimeout - delta, priceTimeout);
162:                     // during overflow should revert since a FIX_MAX _low breaks everything
163:                 }
```

We can see that when the decay starts (`delta == decayDelay`), the saved `high/low` readings are returned unchanged; then they start diverging linearly: at the end of the decay (left limit of `delta == decayDelay + priceTimeout`) `low` reaches `0` and `high` is scaled up by a multiplier of `FIX_ONE + MAX_HIGH_PRICE_BUFFER`, that is 3x its cached value.

Referring to the the below chart, because the two values - `low` (blue)  and `high` (red) - diverge with a different slope, their average (`(low + high) / 2`, green) also varies over time and increases:

![desmos-graph](https://gist.github.com/user-attachments/assets/d4a42618-e03c-4f9b-ae95-61bb77bbd224)

While this is not an issue for the Asset itself which doesn't directly combine the two values with an average, it can be for downstream contracts that may do.

Consider changing `MAX_HIGH_PRICE_BUFFER` to `FIX_ONE` instead.

----

## [L-2] "Flash" upgrade can be abused to create rigged but honest-looking RToken contracts

### Links to affected code
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Main.sol#L111-L150

### Description

The Main contract offers the possibility to upgrade Main and Component implementations, fetching the target addresses from an external `versionRegistry` provider.

Because the Deployer doesn't set `versionRegistry` but let the contract admin provide it, and this entity can't always be trusted as it's specified in the permissionless `Deployer.deploy` function, the contract upgrade system can be abused to achieve a large variety of deviations from its intended behavior by:
- setting custom implementations
- tampering with the storage of all contracts (including balances and/or settings to bypass `init` sanity checks)
- restoring the contracts in a state that looks legitimate with proper implementations and governance

All of the above can be achieved in one single transaction (hence the "Flash upgrade" term used above), possibly to be bundled with the contract creation or buried in a long list of spammy interactions to lower the chances of detection; after a moderately sophisticated attack like [the one presented in PoC](https://gist.github.com/3docSec/1cf7037b38f72719326f0f59d3f787f2) there can be no signs of past wrongdoing in the result of any of the RToken contracts' getters.

Consider having the Deployer set `Main.versionRegistry` at RToken deployment time to enforce continued use of trusted code.

----

## [L-3] Prime basket weights are not properly validated

### Links to affected code
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L264

### Description

When the Governance sets the prime basket compositions, for each of the provided collaterals, it is allowed to specify any target amount between `1` and `MAX_TARGET_AMT` (L264):
```Solidity
File: BasketHandler.sol
260:         for (uint256 i = 0; i < erc20s.length; ++i) {
261:             // This is a nice catch to have, but in general it is possible for
262:             // an ERC20 in the prime basket to have its asset unregistered.
263:             require(assetRegistry.toAsset(erc20s[i]).isCollateral(), "erc20 is not collateral");
264:             require(0 < targetAmts[i] && targetAmts[i] <= MAX_TARGET_AMT, "invalid target amount");
265: 
266:             config.erc20s.push(erc20s[i]);
267:             config.targetAmts[erc20s[i]] = targetAmts[i];
268:             names[i] = assetRegistry.toColl(erc20s[i]).targetName();
269:             config.targetNames[erc20s[i]] = names[i];
270:         }
```

This check is however insufficient because excessively low values of `targetAmt`, like `1`, would likely cause overflows in the code that translates balances to assets / baskets like the BasketHandler.basketsHeldBy function.

Consider enforcing the [range specified in the acceptable values for prime basket weights](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/docs/solidity-style.md?plain=1#L105), at the very least by requiring `targetAmts` to be `1e-6 (D18)` or more instead of the `1e-18 (D18)` or more that is currently allowed.

----

## [L-4] RToken issuance/redemption throttles can be monopolized by Bundlers and Batchers

### Links to affected code
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L121-L122
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L199-L200
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/RToken.sol#L277-L278


### Description

When the RToken issuance and redemption are close to their limits, every upcoming issuance and redemption creates opportunity for a new operation in opposite direction to happen in the form of restored amount available for redemption or issuance respectively.

This means that actors that can control the ordering of transactions, and/or when a transaction is executed, like MEV bundlers and gasless transaction batchers, will have the upper hand on using these available amounts, potentially monopolizing them for themselves.

We don't have a suggested mitigation for the contracts in the scope, but we'd rather issue a recommendation for users to use private mempools and avoid gasless transactions for operations involving RToken issuance and redemption.

----

## [L-5] Broker accepts `batchAuctionLength` and `dutchAuctionLength` to be both `0`

### Links to affected code
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Broker.sol#L199
https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/Broker.sol#L221

### Description

When `Broker` is initialized, and later re-configured via `governance` calls, it validates the `batchAuctionLength` and `dutchAuctionLength` given parameters individually:

```Solidity
File: Broker.sol
073:     function init(
---
080:     ) external initializer {
---
101:         setBatchAuctionLength(batchAuctionLength_);
102:         setDutchAuctionLength(dutchAuctionLength_);
103:     }
---
197:     function setBatchAuctionLength(uint48 newAuctionLength) public governance {
198:         require(
199:             newAuctionLength == 0 ||
200:                 (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
201:             "invalid batchAuctionLength"
202:         );
203:         emit BatchAuctionLengthSet(batchAuctionLength, newAuctionLength);
204:         batchAuctionLength = newAuctionLength;
205:     }
---
219:     function setDutchAuctionLength(uint48 newAuctionLength) public governance {
220:         require(
221:             newAuctionLength == 0 ||
222:                 (newAuctionLength >= MIN_AUCTION_LENGTH && newAuctionLength <= MAX_AUCTION_LENGTH),
223:             "invalid dutchAuctionLength"
224:         );
225:         emit DutchAuctionLengthSet(dutchAuctionLength, newAuctionLength);
226:         dutchAuctionLength = newAuctionLength;
227:     }
```

It does however not validate the situation when they are both provided as `0`. This is a situation that is not admissible because in this case, no auction can be created and an BasketManager can't open recollateralization trades.

Consider adding additional checks in `setBatchAuctionLength` and `setDutchAuctionLength` to prevent setting both to `0`.

----

## [L-6] PermitLib uses two different ERC1271 implementations for the same call

### Links to affected code

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/libraries/Permit.sol#L17

### Description

PermitLib offers the `requireSignature` function, which calls `isValidSignature` if `owner` is a contract, and uses OZ's `isValidSignatureNow` otherwise.

If we look at [the `isValidSignatureNow` implementation of the imported OZ version](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/2d081f24cac1a867f6f73d512f2022e1fa987854/contracts/utils/cryptography/SignatureCheckerUpgradeable.sol#L28), however, we can see that this, too, has a fallback call to `IERC1271.isValidSignature`.

The `if isContract` branch in `requireSignature` is therefore useless because it's redundant with the other, more standard, branch and we therefore recommend removing it.

----

## [L-7] VersionRegistry.latestVersion does not support semantic versioning

### Links to affected code

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/registry/VersionRegistry.sol#L54

### Description

From the [version history of the Reserve protocol](https://github.com/reserve-protocol/protocol/tags) it appears that the protocol is using semantic versioning or a similar alternative.

This seems a use case that does not fit well with how `VersionRegistry.latestVersion` is updated: every time a new version is registered on `VersionRegistry`,  `latestVersion` will point to that version.

In the event that the protocol releases `4.0.0` and shortly after `3.4.2`, then `latestVersion` will incorrectly point to `3.4.2`.

Consider adding a `boolean` flag to the `registerVersion` function, allowing the caller to specify whether `latestVersion` should be updated or not.

----

## [L-8] DutchTrade.bidWithCallback does not send back excess tokens

### Links to affected code

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/plugins/trading/DutchTrade.sol#L260

### Description

DutchTrade.bidWithCallback allows for bidding with a callback hook that allows the bidder to swap the bought tokens for the sold tokens.

After the callback exits, the function checks that enough tokens are provided:

```Solidity
File: DutchTrade.sol
257:         uint256 balanceBefore = buy.balanceOf(address(this)); // {qBuyTok}
258:         IDutchTradeCallee(bidder).dutchTradeCallback(address(buy), amountIn, data);
259:         require(
260:             amountIn <= buy.balanceOf(address(this)) - balanceBefore,
261:             "insufficient buy tokens"
262:         );
```

However, if extra tokens are given, the function does not return the extras to the caller. Consider adding a check to return extra tokens if any are provided

----

## [L-9] Missing check to avoid circular dependencies among RTokens

### Links to affected code

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BasketHandler.sol#L166

### Description

When BasketHandler switches basket to new reference amounts, it calls `trackStatus()`:

```Solidity
File: BasketHandler.sol
158:     function refreshBasket() external {
159:         assetRegistry.refresh();
160: 
161:         require(
162:             main.hasRole(OWNER, _msgSender()) ||
163:                 (lastStatus == CollateralStatus.DISABLED && !main.tradingPausedOrFrozen()),
164:             "basket unrefreshable"
165:         );
166:         _switchBasket();
167: 
168:         trackStatus();
169:     }
```

This call sequence would not call the scenario when there is a circular dependency between RTokens, for example: 
- RTokenA has RTokenB as collateral
- RTokenB governance is is unaware of their token being a collateral in RTokenA
- RTokenB adds RTokenA as collateral

At this point, price retrieval of either collateral would fail because of an infinite recursion.
While the RTokenB governance action can be seen as a mistake, RTokenA is affected too without any mistake made by its governance.

Consider adding a `price()` call after `trackStatus()` to trigger a failure in the above-mentioned case.

----

## [L-10] StRSRVotes.delegateBySig() misses check for delegation to happen in the era intended by the signer

### Links to affected code

https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/StRSRVotes.sol#L176

### Description

The `StRSRVotes.delegateBySig()` function allows a signer to delegate voting power to a designed delegate via a gasless call.

If we look at the logic that verifies the signature verification:

```Solidity
File: StRSRVotes.sol
166:     function delegateBySig(
167:         address delegatee,
168:         uint256 nonce,
169:         uint256 expiry,
170:         uint8 v,
171:         bytes32 r,
172:         bytes32 s
173:     ) public {
174:         require(block.timestamp <= expiry, "signature expired");
175:         address signer = ECDSAUpgradeable.recover(
176:             _hashTypedDataV4(keccak256(abi.encode(_DELEGATE_TYPEHASH, delegatee, nonce, expiry))),
177:             v,
178:             r,
179:             s
180:         );
181:         require(nonce == _useDelegationNonce(signer), "invalid nonce");
182:         _delegate(signer, delegatee);
183:     }
```

we can see that the `era` is missing from the signed payload at L176.

This means that a delegation signature can be reused across era changes, despite balances and voting power no longer apply.

Consider adding an `era` field to the signed payload for `StRSRVotes.delegateBySig()`

