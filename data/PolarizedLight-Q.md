
## Team Polarized Light - QA Report - Reserve Audit on Code4rena

# [Low-Severity-Findings]

## [Low-1] Gas grief possible on unsafe external calls used in `main.sol::function _upgradeProxy`  

There is the use of a low-level `call` in the `_upgradeProxy` function. While trusting the proxy reduces the risk of intentional malicious behavior, it does not eliminate the potential for issues arising from gas usage, reentrancy, or unexpected behavior.

```solidity
function _upgradeProxy(address proxy, address implementation) internal { // 
        (bool success, ) = proxy.call(

            abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector, implementation)

        );
         require(success, "upgrade failed");
    }
```

Mitigations: 

By limiting gas, performing thorough checks, and ensuring proper error handling, you can mitigate these risks and enhance the robustness of the upgrade process.

Here are some recommended changes to be made to this function:

1. Limit Gas Forwarded - Specify the gas limited forwarded to the external call to ensure the call does not consume excessive gas.

```solidty
(bool success, ) = proxy.call{gas: 50000}(

    abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector, implementation)
);
```

2. Utilize OpenZeppelin's Address library - This library provides safer methods for making external calls. This library includes proper error handling and can limit the amount of gas forwarded.

```solidity
import "@openzeppelin/contracts/utils/Address.sol";
```

```solidity
Address.functionCall(proxy, abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector, implementation), "upgrade failed");
```

## [Low-2] `isContract` is not a reliable way to determine if an address is a EOA or contract.    

Within the `Libraries::Permit.sol::Library PermitLib`,`If(AddressUpgradeable.isContract(owner))` is being used, which can cause potential issues.

`isContract` is no longer considered reliable due to:

1. Proxy contracts
2. Contract creation during transaction execution
3. Potential for contracts to be misclassified as EOAs or vice versa

OpenZeppelin has deprecated `isContract` in their Address library due to these limitations.

The security implications of continuing to use `isContract` could lead to:

1. Bypassing intended signature verification methods
2. Potential vulnerabilities in systems relying on this distinction

While the use of `isContract` in this context is a common pattern and was considered best practice in the past, it's now recognized as potentially problematic. 

Mitigations:

Remove the reliance on `isContract` - The primary recommendation is to remove the dependency of `isContract` entirely. Instead considering treating all addresses as potential contracts and attempt ERC1271 signature verification first.

```solidity
library PermitLib {
    function requireSignature(
        address owner,
        bytes32 hash,
        uint8 v,
        bytes32 r,
        bytes32 s
        
    ) internal view {
        bytes memory signature = abi.encodePacked(r, s, v);
        // Try ERC1271 verification first

        try IERC1271Upgradeable(owner).isValidSignature(hash, signature) returns (bytes4 magicValue) {
            require(magicValue == 0x1626ba7e, "ERC1271: Unauthorized");
            return;
        } catch {

            // If ERC1271 fails, fall back to EOA signature verification
            require(
                SignatureCheckerUpgradeable.isValidSignatureNow(
                    owner,
                    hash,
                    signature
                ),
                "Invalid signature"

            );
        }
    }
}

```

Another step would be to implement the use of OpenZeppelin's ECDSA library - OpenZeppelin's ECDSA library provides a `recover` function that works for both EOAs and ERC1271 contracts:

```solidity
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
library PermitLib {
    function requireSignature(
        address owner,
        bytes32 hash,
        uint8 v,
        bytes32 r,
        bytes32 s

    ) internal view {
        bytes memory signature = abi.encodePacked(r, s, v);
        require(
            ECDSA.recover(hash, signature) == owner,
            "Invalid signature"
        );
    }
}

```

## [Low-3] Potential Off-by-One Error in Trading Delay Implementation within `contracts/p1/BackingManager.sol`

The `rebalance()` withing `BackingManager.sol#L107-L122` and `forwardRevenue()` within `BackingManager.sol#L178-L188` functions use `>=` when comparing `block.timestamp` with the trading delay:

In the `rebalance()` function:

```
require(block.timestamp >= basketHandler.timestamp() + tradingDelay, "trading delayed");
```

In the forwardRevenue() function:

```
require(block.timestamp >= basketHandler.timestamp() + tradingDelay, "trading delayed");
```

This allows actions to occur exactly when the trading delay ends, which may not align with the intended behavior of strictly enforcing the delay.

Impact:

Actions could be executed up to one block earlier than intended if the block timestamp exactly matches the end of the trading delay. G

Recommendation:

Change the comparison operator from `>=` to `>` to ensure actions occur strictly after the trading delay:

```solidity
require(block.timestamp > basketHandler.timestamp() + tradingDelay, "trading delayed");
```

## [Low-4] Consider adding a time delay to upgrade implementation functions   

The MainP1 contract contains two critical upgrade functions, `upgradeMainTo` and `upgradeRTokenTo`. These functions allow for immediate upgrades of the main contract and various component contracts without any time delay. While access is restricted to authorized roles (OWNER), this still presents a single point of failure and doesn't provide stakeholders with a window to review or the ability to react to these proposed upgrades.

Impact:
Malicious or erroneous upgrades could be executed instantly, potentially compromising user funds or system integrity. Users and stakeholders also have no time to react to or contest proposed upgrades.

If an owner's credentials are compromised, an attacker could immediately upgrade to a malicious implementation.

Recommendation:

Implement a timelock mechanism for upgrade functions. This adds a mandatory delay between proposing and executing an upgrade, enhancing security and transparency.

1. Basic Timelock

```solidity
struct PendingUpgrade {
    bytes32 versionHash;
    uint256 executeTime;
}

PendingUpgrade public pendingMainUpgrade;
uint256 public constant UPGRADE_DELAY = 2 days;

function proposeMainUpgrade(bytes32 versionHash) external onlyRole(OWNER) {
    pendingMainUpgrade = PendingUpgrade(versionHash, block.timestamp + UPGRADE_DELAY);
    emit UpgradeProposed(versionHash, pendingMainUpgrade.executeTime);
}

function executeMainUpgrade() external onlyRole(OWNER) {
    require(block.timestamp >= pendingMainUpgrade.executeTime, "Timelock not expired");
    require(pendingMainUpgrade.versionHash != bytes32(0), "No upgrade pending");
    // Existing upgrade logic here
    upgradeMainTo(pendingMainUpgrade.versionHash);
    delete pendingMainUpgrade;
}
```

2. Cancelable Timelock:

Add a function to cancel proposed upgrades
```solidity
function cancelMainUpgrade() external onlyRole(OWNER) {
    emit UpgradeCancelled(pendingMainUpgrade.versionHash);
    delete pendingMainUpgrade;
}
```

3. Multi-signature Timelock:

Require multiple authorized signers to approve the upgrade within the timelock period:
```solidity
mapping(address => bool) public hasApproved;
uint256 public constant REQUIRED_APPROVALS = 3;
uint256 public approvalCount;

function approveUpgrade() external onlyRole(APPROVER) {
    require(!hasApproved[msg.sender], "Already approved");
    hasApproved[msg.sender] = true;
    approvalCount++;
}

function executeMainUpgrade() external onlyRole(OWNER) {
    require(block.timestamp >= pendingMainUpgrade.executeTime, "Timelock not expired");
    require(approvalCount >= REQUIRED_APPROVALS, "Insufficient approvals");
    // Existing upgrade logic here
    // Reset state
    delete pendingMainUpgrade;
    approvalCount = 0;
}

```

4. Governance-based Timelock:
Integrate with a governance system where token holders can vote on proposed upgrades during the timelock period.

By implementing one of these options, the system would significantly enhance its security posture and provide greater transparency and control over the upgrade process.

## [Low-5] Try catch statement without human readable error within `Distributor.sol#L63-L66`

The DistributorP1 contract uses try-catch statements without catching human-readable errors. This practice prevents the contract from capturing and handling specific, descriptive error messages that could provide valuable information about failures in critical operations for example.

```solidity
function setDistribution(address dest, RevenueShare calldata share) external governance {

    try main.rsrTrader().distributeTokenToBuy() {} catch {}
    try main.rTokenTrader().distributeTokenToBuy() {} catch {}
    _setDistribution(dest, share);
    RevenueTotals memory revTotals = totals();
    _ensureSufficientTotal(revTotals.rTokenTotal, revTotals.rsrTotal);
}

function setDistributions(address[] calldata dests, RevenueShare[] calldata shares)
    external
    governance

{

    require(dests.length == shares.length, "array length mismatch");
    try main.rsrTrader().distributeTokenToBuy() {} catch {}
    try main.rTokenTrader().distributeTokenToBuy() {} catch {}

    for (uint256 i = 0; i < dests.length; ++i) {
        _setDistribution(dests[i], shares[i]);

    }

    RevenueTotals memory revTotals = totals();
    _ensureSufficientTotal(revTotals.rTokenTotal, revTotals.rsrTotal);

}

```

Impact:

1. Loss of specific error information:
    - Critical details about failed `distributeTokenToBuy()` calls are lost.
    - Administrators lack visibility into the root causes of distribution failures.
2. Reduced debugging capability:
    - Troubleshooting becomes more time-consuming and complex.
    - Identifying patterns in distribution failures is hindered.
3. Missed opportunity for targeted error handling:
    - Contract cannot differentiate between various error types (e.g., out-of-gas vs. contract reverts).
    - Unable to implement specific recovery mechanisms for different error scenarios.
4. Potential state inconsistencies:
    - If distribution attempts fail silently, subsequent operations may occur in an unexpected system state.
    - This could lead to incorrect distribution settings or unintended economic consequences.
5. Decreased protocol reliability:
    - Silent failures may mask recurring issues, reducing overall system dependability.
    - Users and integrators may experience unexplained behavior, potentially eroding trust in the protocol.
6. Impaired monitoring and alerting:
    - Off-chain monitoring systems cannot effectively track and alert on specific error conditions.
    - This delays awareness and response to critical issues.

Why It Needs to Change

Improved error diagnostics: Capturing human-readable errors would allow for more precise identification of issues in the distribution process.

Enhanced debugging and maintenance: Specific error messages would make it easier for developers to troubleshoot and fix problems.

Potential for more nuanced error handling: With human-readable errors, the contract could implement more sophisticated error handling strategies based on the specific type of error encountered.

Recommended Mitigations:

Implement error catching for human-readable errors:

```diff
function setDistribution(address dest, RevenueShare calldata share) external governance {
-   try main.rsrTrader().distributeTokenToBuy() {} catch {}
-   try main.rTokenTrader().distributeTokenToBuy() {} catch {}
+   _tryDistributeTokenToBuy(main.rsrTrader());
+   _tryDistributeTokenToBuy(main.rTokenTrader());
    _setDistribution(dest, share);
    // ... rest of the function ...
  }

  function setDistributions(address[] calldata dests, RevenueShare[] calldata shares)
    external
    governance
  {
    require(dests.length == shares.length, "array length mismatch");
-   try main.rsrTrader().distributeTokenToBuy() {} catch {}
-   try main.rTokenTrader().distributeTokenToBuy() {} catch {}
+   _tryDistributeTokenToBuy(main.rsrTrader());
+   _tryDistributeTokenToBuy(main.rTokenTrader());
    // ... rest of the function ...
  }

+ function _tryDistributeTokenToBuy(IRevenueTrader trader) internal {
+   try trader.distributeTokenToBuy() {
+     // Operation successful
+   } catch Error(string memory reason) {
+     emit DistributeTokenToBuyError(address(trader), reason);
+   } catch (bytes memory lowLevelData) {
+     emit DistributeTokenToBuyErrorBytes(address(trader), lowLevelData);
+   }
+ }

+ event DistributeTokenToBuyError(address trader, string reason);
+ event DistributeTokenToBuyErrorBytes(address trader, bytes data);
```

Consider adding specific error handling logic based on the caught error messages.

By implementing these mitigations, the contract will be able to capture and handle human-readable errors, significantly improving its error diagnostics and potential for targeted error handling.

## [Low-6] The use of deprecated AccessControl functions in `contracts/registry/RoleRegistry.sol#L15-L15 

The `RoleRegistry.sol` contract (line 15) uses the deprecated `_setupRole` function from OpenZeppelin's AccessControl. This outdated implementation poses potential compatibility risks and goes against best practices. Using deprecated functions may lead to future issues and is not recommended for maintaining a secure and up-to-date smart contract system.

The current recommended way to set up roles in OpenZeppelin's AccessControl is to use `_grantRole` instead of `_setupRole`.

Potential impact: 

While the contract will still function as intended with `setupRole`, it may cause issues if attempts are made to upgrade to newer versions of OpenZeppelin. 

```diff
constructor() {
+        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);  
-        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
```

This change will maintain the same functionality while using the current recommended method.

## [Low-7]  `contracts/p1/Main.sol#L157-L159` susceptible to .selector-related optimizer bug due to the use of version Solidity 0.8.19.

The smart contract `contracts/p1/Main.sol#L157-L159` uses Solidity version 0.8.19, which is known to have a .selector-related optimizer bug. This bug can cause incorrect code generation in specific scenarios where .selector is used with function calls instead of direct contract names for selector lookup.

In the `_upgradeProxy` function, there's an instance of .selector usage that matches the pattern susceptible to this bug:

```solidity
abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector, implementation)
```

Impact:

The impact of this could potentially lead to, Incorrect execution of the upgrade process if the .selector is not properly recognized. Possible failure of contract upgrades, which could temporarily break contract functionality. In worst-case scenarios, it might lead to unexpected behavior in the upgrade mechanism, potentially compromising the contract's security model.

While the specific usage in this contract (internal function, infrequent operation) limits the exposure, it still represents a security risk that should be addressed.

Recommended Mitigation

Update the Solidity version to 0.8.21 or later, which fixes this optimizer bug. This is the most straightforward and recommended solution. If immediate updating is not possible, consider refactoring the code to avoid using .selector in this pattern.

By addressing this issue, it would improve the contract's security and ensure it functions correctly in all scenarios.

## [Low-8] Functions calling contracts/addresses with transfer hooks are missing reentrancy guards

Read-only reentrancy risks exist in functions lacking reentrancy guards, even with check-effects-interaction patterns. These vulnerabilities can be exploited without altering contract state. Implementing reentrancy guards is crucial to protect against both traditional and read-only reentrancy attacks, enhancing overall protocol security.

Description:

Several critical functions in the protocol are susceptible to reentrancy attacks. These vulnerabilities could lead to unexpected behavior, loss of funds, or manipulation of the protocol's state. The affected functions are:

`withdraw` (StRSRP1 contract)

`seizeRSR` (StRSRP1 contract)

`bidWithCallback` (DutchTrade contract)

`settle` (DutchTrade contract)

`settle` (GnosisTrade contract)

Detailed Analysis:

`withdraw` function (StRSRP1 contract):

Risk: The function modifies multiple state variables, performs complex calculations, and transfers tokens. The final check for basket handler state could be exploited if the state changes during execution.

`seizeRSR` function (StRSRP1 contract):

Risk: This function modifies critical state variables like stakeRSR and draftRSR, performs complex calculations, and interacts with external contracts.

`bidWithCallback` function (DutchTrade contract):

Risk: This function interacts with external contracts through a callback, modifies state, and transfers tokens. The external call could lead to unexpected behavior.

`settle` function (DutchTrade contract):

Risk: While it has some checks, it performs multiple token transfers and interacts with external addresses, which could potentially be exploited.

`settle` function (GnosisTrade contract):

Risk: This function interacts with an external contract (Gnosis auction) and performs multiple token transfers, which could lead to unexpected states if reentered.

Recommendation:

Implement the nonreentrant modifier for all these functions to prevent reentrancy attacks. This will ensure that these functions cannot be re-entered before their execution is complete.

Here's a diff to implement the recommended mitigations:

```diff

// StRSRP1.sol

+ import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";

- contract StRSRP1 is Initializable, ComponentP1, IStRSR, EIP712Upgradeable {

+ contract StRSRP1 is Initializable, ComponentP1, IStRSR, EIP712Upgradeable, ReentrancyGuardUpgradeable {

- function withdraw(address account, uint256 endId) external {
+ function withdraw(address account, uint256 endId) external nonReentrant {

- function seizeRSR(uint256 rsrAmount) external {
+ function seizeRSR(uint256 rsrAmount) external nonReentrant {

// DutchTrade.sol

+ import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

- contract DutchTrade is ITrade, Versioned {
+ contract DutchTrade is ITrade, Versioned, ReentrancyGuard {

- function bidWithCallback(bytes calldata data) external returns (uint256 amountIn) {

+ function bidWithCallback(bytes calldata data) external nonReentrant returns (uint256 amountIn) {

- function settle()
+ function settle() nonReentrant

// GnosisTrade.sol

+ import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

- contract GnosisTrade is ITrade, Versioned {
+ contract GnosisTrade is ITrade, Versioned, ReentrancyGuard {

- function settle()
+ function settle() nonReentrant
```

By implementing these changes, the protocol significantly reduces the risk of reentrancy attacks in these critical functions. This enhances the overall security of the system and protects against potential exploitation of these vulnerabilities.


## [Low-9] Missing events in functions that are either setters, privileged or voting related

Sensitive setter functions in smart contracts often alter critical state variables. Without events emitted in these functions, external observers or dApps cannot easily track or react to these state changes. Missing events can obscure contract activity, hampering transparency and making integration more challenging. To resolve this, incorporate appropriate event emissions within these functions. Events offer an efficient way to log crucial changes, aiding in real-time

Implementing events in these functions is crucial for multiple reasons

Transparency: Events provide a clear, immutable record of important state changes within the contract.

Off-chain Tracking: They allow external systems and dApps to easily monitor and react to contract activities.

Auditability: Events facilitate easier auditing and debugging of contract behavior.

User Interface Updates: They enable real-time updates in user interfaces, improving user experience.

Integration: Events make it easier for other contracts or systems to integrate with your contract.racking and post-transaction verification.

Mitigations:

Consider implementing the following changes.

For - `setPrimeBasket`

```diff
+ event PrimeBasketSet(IERC20[] erc20s, uint192[] targetAmts);
function setPrimeBasket(IERC20[] calldata erc20s, uint192[] calldata targetAmts) external {
    // Existing function logic
+   emit PrimeBasketSet(erc20s, targetAmts);

}
```

For - `setDistribution`

```diff
+ event DistributionSet(address indexed dest, RevenueShare share);
function setDistribution(address dest, RevenueShare calldata share) external governance {
    // Existing function logic
+   emit DistributionSet(dest, share);
}
```

For -  `setDistributions`

```diff
+ event DistributionsSet(address[] dests, RevenueShare[] shares);
function setDistributions(address[] calldata dests, RevenueShare[] calldata shares)
    external
    governance
{
    // Existing function logic
+   emit DistributionsSet(dests, shares);
}
```

For - `setVersionRegistry`

```diff
+ event VersionRegistrySet(VersionRegistry indexed newRegistry);
+ 
function setVersionRegistry(VersionRegistry versionRegistry_) external onlyRole(OWNER) {
    // Existing function logic
+   emit VersionRegistrySet(versionRegistry_);
}
```

For - `setAssetPluginRegistry`

```diff
+ event AssetPluginRegistrySet(AssetPluginRegistry indexed newRegistry);

function setAssetPluginRegistry(AssetPluginRegistry registry_) external onlyRole(OWNER) {
    // Existing function logic
+   emit AssetPluginRegistrySet(registry_);
}
```

For - `setDAOFeeRegistry`

```diff
+ event DAOFeeRegistrySet(DAOFeeRegistry indexed newFeeRegistry);

function setDAOFeeRegistry(DAOFeeRegistry feeRegistry_) external onlyRole(OWNER) {
    // Existing function logic
+   emit DAOFeeRegistrySet(feeRegistry_);
}
```

For -  `setVotingDelay`

```diff
+ event VotingDelaySet(uint256 newVotingDelay, uint256 oldVotingDelay);

function setVotingDelay(uint256 newVotingDelay) public override {
+   uint256 oldVotingDelay = votingDelay();
    // Existing function logic
+   emit VotingDelaySet(newVotingDelay, oldVotingDelay);
}
```

For - `refreshBasket`

```diff
+ event BasketRefreshed(IERC20[] newBasket, uint192[] newTargetAmts);

function refreshBasket() external {
    // Existing function logic
+   IERC20[] memory newBasket = // ... get new basket
+   uint192[] memory newTargetAmts = // ... get new target amounts
+   emit BasketRefreshed(newBasket, newTargetAmts);
}
```

Implementing these events will significantly enhance the transparency, auditability, and integrability of your smart contract. Each event provides crucial information about state changes, allowing off-chain systems to track and react to on-chain activities effectively.

By implementing these changes, you'll create a more robust and user-friendly smart contract ecosystem.

## [Low-10] Functions with Read-only Reentrancy Risk 

Multiple functions in the protocol are potentially vulnerable read-only reentrancy attacks. These attacks can manipulate the state read by these functions without actually changing the contract's storage, potentially leading to unexpected behavior or exploitation.

At-Risk Functions

`issueTo` (RTokenP1.sol, line 105):

Risk: Multiple external calls and state changes without protection against read-only reentrancy.

Specific concern: The function relies on `assetRegistry.refresh()` and `basketHandler.isReady()`, which could be manipulated in a read-only reentrancy attack.

`distribute` (DistributorP1.sol, line 120):

Risk: Complex logic with multiple transfers and reliance on external state.

Specific concern: The function uses totals() to calculate distributions, which could be manipulated in a read-only reentrancy scenario.

`redeemTo` (RTokenP1.sol, line 183):

Risk: Multiple external calls and state changes without protection.

Specific concern: Similar to issueTo, it relies on `assetRegistry.refresh()` and external balance checks that could be exploited.

`withdraw` (StRSRP1.sol, line 304):

Risk: External calls and state changes with potential for manipulation.

Specific concern: The function checks `basketHandler.isReady()` and `basketHandler.fullyCollateralized()` at the end, which could be exploited in a read-only reentrancy attack.

`seizeRSR` (StRSRP1.sol, line 424):

Risk: Multiple state changes and an external transfer without full protection.

Specific concern: While it has access control, it still relies on balance checks that could be manipulated.

Mitigations:

To better protect these functions from read-only reentrancy attacks, consider the following changes:

For `issueTo` (RTokenP1.sol, line 105):
```diff
+ import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

- contract RTokenP1 is ComponentP1, ERC20PermitUpgradeable, IRToken {
+ contract RTokenP1 is ComponentP1, ERC20PermitUpgradeable, IRToken, ReentrancyGuard {

- function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen {
+ function issueTo(address recipient, uint256 amount) public notIssuancePausedOrFrozen nonReentrant {
```

For `redeemTo` (RTokenP1.sol, line 183):
```diff
- function redeemTo(address recipient, uint256 amount) public notFrozen {
+ function redeemTo(address recipient, uint256 amount) public notFrozen nonReentrant {
```

For `withdraw` (StRSRP1.sol, line 304):
```diff
- function withdraw(address account, uint256 endId) external {
+ function withdraw(address account, uint256 endId) external nonReentrant {
```

For `distribute` (DistributorP1.sol, line 120):
```diff
- contract DistributorP1 is ComponentP1, IDistributor {
+ contract DistributorP1 is ComponentP1, IDistributor, ReentrancyGuard {

- function distribute(IERC20 erc20, uint256 amount) external {
+ function distribute(IERC20 erc20, uint256 amount) external nonReentrant {
```

For `seizeRSR` (StRSRP1.sol, line 424):
```diff
- function seizeRSR(uint256 rsrAmount) external {
+ function seizeRSR(uint256 rsrAmount) external nonReentrant {
```

These changes add the ReentrancyGuard to the relevant contracts and apply the nonReentrant modifier to each of the at-risk functions. This provides a straightforward way to protect against both read-only and state-changing reentrancy attacks for these functions.

# [NonCritical-findings]

## [NonCritical-1] call abi.encodeWithSelector within `contracts/p1/Main.sol#L158-L159` is not type safe.

The current implementation doesn't pose a significant risk in this specific case because the types are known and fixed. However, Reserve is not taking advantage of the latest Solidity features for type safety within this instance. The current implementation is functionally correct but could be made more robust by adopting abi.encodeCall.

Consider adopting the suggested logic below.

```diff
(bool success, ) = proxy.call(
-    abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector, implementation)
+    abi.encodeCall(UUPSUpgradeable.upgradeTo, (implementation))
);
```

By implementing this change, the contract will benefit from increased type safety without altering its functionality. This improvement aligns with best practices in Solidity development and reduces the potential for future errors if the function signature changes.

## [NonCritical-2]  Lack of Zero Address Check for Recipient in `contracts/p1/RToken.sol#L183-L183` `redeemTo` Function

The `redeemTo` function in the RTokenP1 contract does not explicitly check if the recipient address is the zero address (0x0). While there are some safeguards in place, this specific check is missing and could potentially lead to unintended behavior.

The `_burn` function, which is called during the redemption process, includes a check to prevent burning from the zero address. However, the lack of a specific check for the recipient in `redeemTo` could still lead to edge cases where RTokens are burned but the underlying assets are sent to the zero address, potentially resulting in their loss.

Affected Code:

The redeemTo function in the RTokenP1 contract:

```solidity
function redeemTo(address recipient, uint256 amount) public notFrozen {
```

Recommendation:

Add an explicit check at the beginning of the `redeemTo` function to revert the transaction if the recipient is the zero address:

Here's a diff of the recommended change:

```diff
function redeemTo(address recipient, uint256 amount) public notFrozen {
+   require(recipient != address(0), "RToken: cannot redeem to zero address");
    // == Refresh ==
    assetRegistry.refresh();
    // ... rest of the function
}
```

While the existing safeguards mitigate most risks, implementing this additional check would eliminate any remaining edge cases and align the code with best practices for handling address inputs in critical functions. This change enhances the contract's robustness and improves its overall security posture.

## [NonCritical-3] Floating Pragma Usage in `contracts/libraries/Fixed.sol#L4-L4`

The FixedPoint library, which provides critical fixed-point arithmetic functionality, uses a floating pragma:

```solidity
pragma solidity ^0.8.19;
```

This allows the contract to be compiled with any Solidity version from 0.8.19 onwards

Impact

The continued use of floating pragma can potentially lead to:

1. Version Inconsistency: Different developers or deployments might use different compiler versions, potentially leading to subtle inconsistencies in contract behavior.
2. Future Compatibility Issues: As new Solidity versions are released, there's a risk of unintentionally introducing bugs or changed behavior without explicit opt-in.
3. Auditing Complications: Floating pragmas can complicate the auditing process by introducing uncertainty about the exact compiler version used.
4. Reduced Reproducibility: It becomes more challenging to ensure consistent and reproducible builds across different environments and times.
5. Security Vulnerabilities: In rare cases, compiling with a newer version might introduce or reintroduce security vulnerabilities that were not present in the originally intended version.

Recommendation: 

Consider using a fixed pragma version to ensure consistent compilation and behavior

```diff
- pragma solidity ^0.8.19;
+ pragma solidity 0.8.19;
```

By adopting a fixed pragma, the project ensures consistent compilation results across different environments and reduces the risk of introducing unintended behavior in future compiler versions.

## [NonCritical-4] Missing events in sensitive functions   

Several important functions within the smart contracts lack event emissions. Emitting events for significant state changes is a crucial practice in smart contract development, even for functions only callable by privileged roles. They help improve the following.

- Transparency: On-chain visibility of state changes
- Off-chain tracking: Easier monitoring without constant polling
- Auditability: Immutable logs for reconstructing contract history
- Debugging: Clear signals about function calls
- User interfaces: Real-time updates for better UX

Below are the functions recommended to have Event Emissions implemented.

`setPrimeBasket` in BasketHandler.sol:

```solidity
function setPrimeBasket(IERC20[] calldata erc20s, uint192[] calldata targetAmts) external { //
    }
```

contracts/p1/Distributor.sol#L61-L61 - function `setDistribution`

```solidity
function setDistribution(address dest, RevenueShare calldata share) external governance {
```

contracts/p1/Distributor.sol#L81-L81 - function `setDistributions`

```solidity
function setDistributions(address[] calldata dests, RevenueShare[] calldata shares) 
```

contracts/p1/Main.sol#L61-L61 - function `setVersionRegistry`

```solidity
function setVersionRegistry(VersionRegistry versionRegistry_) external onlyRole(OWNER) { 
```

contracts/p1/Main.sol#L70-L70 - function `setAssetPluginRegistry`

```solidity
function setAssetPluginRegistry(AssetPluginRegistry registry_) external onlyRole(OWNER) { 
```

contracts/p1/Main.sol#L79-L79 - function `setDAOFeeRegistry`

```solidity
function setDAOFeeRegistry(DAOFeeRegistry feeRegistry_) external onlyRole(OWNER) { 
```

contracts/p1/mixins/BasketLib.sol#L77-L77 - function `setFrom`

```solidity
function setFrom(Basket storage self, Basket storage other) internal { 
```

contracts/plugins/governance/Governance.sol#L62-L62 - function `setVotingDelay`

```solidity
function setVotingDelay(uint256 newVotingDelay) public override { 
```

Recommendations
  
Add event emissions to these functions. For example:

```diff
+ event PrimeBasketSet(IERC20[] erc20s, uint192[] targetAmts);

function setPrimeBasket(IERC20[] calldata erc20s, uint192[] calldata targetAmts) external {
    // Existing function body
+   emit PrimeBasketSet(erc20s, targetAmts);
}
```

Implementing event emissions for these sensitive functions will significantly improve the contract's transparency, auditability, and ease of integration with external systems. This change aligns with industry best practices and enhances the overall quality of the smart contract system.

## [NonCritical-5] Missing Event Emissions for Non-Immutable State Variables Set in Constructors

Several contracts set non-immutable state variables in their constructors without emitting corresponding events. This practice reduces transparency and makes it harder to track the initial state of contracts

Below are the functions recommended to have Event Emissions implemented.

contracts/p1/Deployer.sol#L42-L69

```solidity

constructor(
    IERC20Metadata rsr_,
    IGnosis gnosis_,
    IAsset rsrAsset_,
    Implementations memory implementations_
) {
    // Validation checks
    rsr = rsr_;
    gnosis = gnosis_;
    rsrAsset = rsrAsset_;
    _implementations = implementations_;
}
```

contracts/registry/AssetPluginRegistry.sol#L26-L28

```solidity
constructor(address _versionRegistry) {
    versionRegistry = VersionRegistry(_versionRegistry);
    roleRegistry = versionRegistry.roleRegistry();
}
```

contracts/plugins/trading/GnosisTrade.sol#L66-L67

```solidity
constructor() {
    status = TradeStatus.CLOSED;
}
```

contracts/registry/DAOFeeRegistry.sol#L35-L41

```solidity
constructor(RoleRegistry _roleRegistry, address _feeRecipient) {
    if (address(_roleRegistry) == address(0)) {
        revert DAOFeeRegistry__InvalidRoleRegistry();
    }

    roleRegistry = _roleRegistry;
    feeRecipient = _feeRecipient;
}
```

contracts/registry/VersionRegistry.sol#L27-L32

```solidity
constructor(RoleRegistry _roleRegistry) {
    if (address(_roleRegistry) == address(0)) {
        revert VersionRegistry__ZeroAddress();
    }

    roleRegistry = _roleRegistry;
}leRegistry = _roleRegistry; 
    }
```

Recommendation:

Emit events for each non-immutable state variable set in the constructor. For example, in Deployer.sol:

```diff
+ event RsrSet(IERC20Metadata indexed rsr);
+ event GnosisSet(IGnosis indexed gnosis);
+ event RsrAssetSet(IAsset indexed rsrAsset);
+ event ImplementationsSet(Implementations implementations);

constructor(
    IERC20Metadata rsr_,
    IGnosis gnosis_,
    IAsset rsrAsset_,
    Implementations memory implementations_
) {
    // Validation checks
    rsr = rsr_;
+   emit RsrSet(rsr_);
    gnosis = gnosis_;
+   emit GnosisSet(gnosis_);
    rsrAsset = rsrAsset_;
+   emit RsrAssetSet(rsrAsset_);
    _implementations = implementations_;
+   emit ImplementationsSet(implementations_);
}
```

## [NonCritical-6] Variable Names Conflicting with Function Names

It's generally a good practice to avoid naming variables the same as defined functions within a project. This practice can lead to confusion for both developers working on the code and users of the project. 

Below are the functions recommended to have Event Emissions implemented.

contracts/p1/BasketHandler.sol#L315-L315

```solidity
uint256 size = basket.erc20s.length;
```

contracts/plugins/governance/Governance.sol#L187-L187

```solidity
uint256 pastEra = IStRSRVotes(address(token)).getPastEra(startTimepoint);
```

To improve these examples, you could rename the variables as follows:

```diff
- uint256 size = basket.erc20s.length;
+ uint256 basketSize = basket.erc20s.length;

- uint256 pastEra = IStRSRVotes(address(token)).getPastEra(startTimepoint);
+ uint256 fetchedPastEra = IStRSRVotes(address(token)).getPastEra(startTimepoint);
```

Adopting clear and distinct naming conventions for variables and functions enhances code readability, reduces the potential for errors, and improves overall maintainability. These changes, while subtle, contribute to a more robust and developer-friendly codebase.

## [NonCritical-7] Function call in event emit found within multiple areas.

Calculating values before event emissions, rather than calling functions directly within emit statements, significantly improves code readability, efficiency, and maintainability. This practice is crucial for developing robust and secure smart contracts.

In the `seizeRSR` function:

```solidity
emit ExchangeRateSet(initRate, exchangeRate()); 
```

In the` _payoutRewards` function:

```solidity
emit ExchangeRateSet(initRate, exchangeRate()); 
```

In the `deployment` function

```solidity
emit RTokenCreated(main, components.rToken, components.stRSR, owner, version()); 
```

Recommendation: 

A better approach would be to calculate the values before the emit statement and then pass these pre-calculated values to the event. For example:

For the `seizeRSR` and `_payoutRewards` functions:

```diff
- emit ExchangeRateSet(initRate, exchangeRate());
+ uint192 newExchangeRate = exchangeRate();
+ emit ExchangeRateSet(initRate, newExchangeRate);
```

For the `deployment` function:

```diff
- emit RTokenCreated(main, components.rToken, components.stRSR, owner, version());
+ string memory currentVersion = version();
+ emit RTokenCreated(main, components.rToken, components.stRSR, owner, currentVersion);
```

While the `version()` call in the `RTokenCreated` event is a simple getter function and less problematic, applying this principle consistently across the codebase improves overall code quality and maintainability.

  

  
  


  
  
