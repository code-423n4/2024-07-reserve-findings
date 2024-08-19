## Summary
### Low Risk

|      | Title                                                                                                                        |
| ---- | ---------------------------------------------------------------------------------------------------------------------------- |
| L-01 | Missing ContextUpgradeable initialization in ComponentP1@__Component_init                                                    |
| L-02 | Missing ReentrancyGuardUpgradeable inititalization in TradingP1@__Trading_init                                               |
| L-03 | ComponentP1@__Component_init should be moved from BackingManagerP1@init and RevenueTraderP1@init to TradingP1@__Trading_init |
| L-04 | ComponentRegistry is extending Auth                                                                                          |
| L-05 | Unnecessary conversions to IERC20 in BackingManagerP1                                                                        |
| L-06 | Unnecessary check in DistributorP1@distribute                                                                                |
| L-07 | Distributor is not using the cached rsrTrader and rTokenTrader when setting distributions                                    |


### Non-Critical

|       | Title                                        |
| ----- | -------------------------------------------- |
| NC-01 | Wrong comment in BasketHandler@basketsHeldBy |

## Low Risks
### L-01 | Missing ContextUpgradeable initialization in ComponentP1@__Component_init

**Issue Description:**
`ComponentP1` extends `ContextUpgradeable` but doesn't initialize it.

[Component.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/mixins/Component.sol#L33-L37)

```solidity
abstract contract ComponentP1 is
    Versioned,
    Initializable,
    ContextUpgradeable, // <==== Audit
    UUPSUpgradeable,
    IComponent
{
    // ...
    function __Component_init(IMain main_) internal onlyInitializing {
        require(address(main_) != address(0), "main is zero address");
        __UUPSUpgradeable_init();
        // <==== Audit
        main = main_;
    }
    // ...
}
```

### L-02 | Missing ReentrancyGuardUpgradeable inititalization in TradingP1@__Trading_init

**Issue Description:**
`TradingP1` extends `ReentrancyGuardUpgradeable` but doesn't initialize it.

[Trading.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/mixins/Trading.sol#L48-L56)

```solidity
abstract contract TradingP1 is Multicall, ComponentP1, ReentrancyGuardUpgradeable, ITrading { // <==== Audit
    // ...
    function __Trading_init(
        IMain main_,
        uint192 maxTradeSlippage_,
        uint192 minTradeVolume_
    ) internal onlyInitializing {
	    // <==== Audit
        broker = main_.broker();
        setMaxTradeSlippage(maxTradeSlippage_);
        setMinTradeVolume(minTradeVolume_);
    }
    // ...
}
```

### L-03 | ComponentP1@__Component_init should be moved from BackingManagerP1@init and RevenueTraderP1@init to TradingP1@__Trading_init

**Issue Description:**
`ComponentP1@__Component_init` should be moved from `BackingManagerP1@init` and `RevenueTraderP1@init` to `TradingP1@__Trading_init`

[BackingManager.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BackingManager.sol#L49-L62)

```solidity
    function init(
        IMain main_,
        uint48 tradingDelay_,
        uint192 backingBuffer_,
        uint192 maxTradeSlippage_,
        uint192 minTradeVolume_
    ) external initializer {
        __Component_init(main_); // <==== Audit
        __Trading_init(main_, maxTradeSlippage_, minTradeVolume_);

        cacheComponents();
        setTradingDelay(tradingDelay_);
        setBackingBuffer(backingBuffer_);
    }
```

[RevenueTrader.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/RevenueTrader.sol#L27-L38)

```solidity
    function init(
        IMain main_,
        IERC20 tokenToBuy_,
        uint192 maxTradeSlippage_,
        uint192 minTradeVolume_
    ) external initializer {
        require(address(tokenToBuy_) != address(0), "invalid token address");
        __Component_init(main_); // <==== Audit
        __Trading_init(main_, maxTradeSlippage_, minTradeVolume_);
        tokenToBuy = tokenToBuy_;
        cacheComponents();
    }
```

[Trading.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/mixins/Trading.sol#L48-L56)

```solidity
abstract contract TradingP1 is Multicall, ComponentP1, ReentrancyGuardUpgradeable, ITrading { // <==== Audit
    // ...
    function __Trading_init(
        IMain main_,
        uint192 maxTradeSlippage_,
        uint192 minTradeVolume_
    ) internal onlyInitializing {
        // <==== Audit
        broker = main_.broker();
        setMaxTradeSlippage(maxTradeSlippage_);
        setMinTradeVolume(minTradeVolume_);
    }
    // ...
}
```

### L-04 | ComponentRegistry is extending Auth

**Issue Description:**
* `ComponentRegistry` is extending `Auth`.
* `MainP1` is extending `ComponentRegistry` but also extends `Auth`

[ComponentRegistry.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/mixins/ComponentRegistry.sol#L12)

```solidity
abstract contract ComponentRegistry is Initializable, Auth, IComponentRegistry {
	// ...
}
```

[Main.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Main.sol#L22)

```solidity
contract MainP1 is Versioned, Initializable, Auth, ComponentRegistry, UUPSUpgradeable, IMain {
	// ...
}
```
### L-05 | Unnecessary conversions to IERC20 in BackingManagerP1

**Issue Description:**
Unnecessary conversions to IERC20 in `BackingManagerP1`

[BackingManager.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BackingManager.sol)

```solidity
contract BackingManagerP1 is TradingP1, IBackingManager {
    // ...
    IERC20 private rsr; // <==== Audit
    // ...
    function grantRTokenAllowance(IERC20 erc20) external notFrozen {
        require(assetRegistry.isRegistered(erc20), "erc20 unregistered");
        // == Interaction ==
        IERC20(address(erc20)).safeApprove(address(rToken), 0); // <==== Audit
        IERC20(address(erc20)).safeApprove(address(rToken), type(uint256).max); // <==== Audit
    }
    // ...
    function forwardRevenue(IERC20[] calldata erc20s) external nonReentrant {
        // ...
        if (rsr.balanceOf(address(this)) != 0) {
            IERC20(address(rsr)).safeTransfer(address(stRSR), rsr.balanceOf(address(this))); // <==== Audit
            stRSR.payoutRewards();
        }
        // ...
    }
    // ...
}
```

### L-06 | Unnecessary check in DistributorP1@distribute

**Issue Description:**
`DistributorP1@distribute` checks for non zero `transferAmt` before setting `accountRewards` to `true`, but that check is not necessary since `tokensPerShare` and `numberOfShares` are checked before.

[Distributor.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Distributor.sol#L120-L200)

```solidity
    function distribute(IERC20 erc20, uint256 amount) external {
        // ...

        uint256 tokensPerShare;
        uint256 totalShares;
        {
            RevenueTotals memory revTotals = totals();
            totalShares = isRSR ? revTotals.rsrTotal : revTotals.rTokenTotal;
            if (totalShares != 0) tokensPerShare = amount / totalShares;
            require(tokensPerShare != 0, "nothing to distribute"); // <==== Audit
        }
        // ...

        for (uint256 i = 0; i < destinations.length(); ++i) {
            address addrTo = destinations.at(i);

            uint256 numberOfShares = isRSR
                ? distribution[addrTo].rsrDist
                : distribution[addrTo].rTokenDist;
            if (numberOfShares == 0) continue; // <==== Audit
            uint256 transferAmt = tokensPerShare * numberOfShares; // <==== Audit
            paidOutShares += numberOfShares;

            if (addrTo == FURNACE) {
                addrTo = address(furnace);
                if (transferAmt != 0) accountRewards = true; // <==== Audit
            } else if (addrTo == ST_RSR) {
                addrTo = address(stRSR);
                if (transferAmt != 0) accountRewards = true; // <==== Audit
            }
			// ...
        }
        // ...
    }
```

### L-07 | Distributor is not using the cached rsrTrader and rTokenTrader when setting distributions

**Issue Description:**
`DistributorP1` is caching `rsrTrader` and `rTokenTrader` but not using them  when setting distribution.

[Distributor.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/Distributor.sol#L61-L98)

```solidity
contract DistributorP1 is ComponentP1, IDistributor {
    // ...
    address private rTokenTrader;
    address private rsrTrader;
    
    function init(IMain main_, RevenueShare calldata dist) external initializer {
        __Component_init(main_);
        cacheComponents(); // <==== Audit
		// ...
    }
    
    function setDistribution(address dest, RevenueShare calldata share) external governance {
        try main.rsrTrader().distributeTokenToBuy() {} catch {} // <==== Audit
        try main.rTokenTrader().distributeTokenToBuy() {} catch {} // <==== Audit
		// ...
    }
    
    function setDistributions(address[] calldata dests, RevenueShare[] calldata shares)
        external
        governance
    {
        try main.rsrTrader().distributeTokenToBuy() {} catch {} // <==== Audit
        try main.rTokenTrader().distributeTokenToBuy() {} catch {} // <==== Audit
		// ...
    }

	// ...

	function cacheComponents() public {
        rsr = main.rsr();
        rToken = IERC20(address(main.rToken()));
        furnace = main.furnace();
        stRSR = main.stRSR();
        rTokenTrader = address(main.rTokenTrader()); // <==== Audit
        rsrTrader = address(main.rsrTrader()); // <==== Audit
    }
}
```

## Non-Critical
### NC-01 | Wrong comment in BasketHandler@basketsHeldBy

**Issue Description:**
Should be `Returns : (0, type(uint192).max)` instead of `Returns : (0, 0)`.

[BasketHandler.sol](https://github.com/code-423n4/2024-07-reserve/blob/main/contracts/p1/BasketHandler.sol#L603-L628)
```solidity
    /// @return baskets {BU}
    ///          .top The number of partial basket units: e.g max(coll.map((c) => c.balAsBUs())
    ///          .bottom The number of whole basket units held by the account
    /// @dev Returns (FIX_ZERO, FIX_MAX) for an empty or DISABLED basket
    // Returns:
    //    (0, 0), if (basket.erc20s is empty) or (disabled is true) or (status() is DISABLED)
    //    min(e.balanceOf(account) / quantity(e) for e in basket.erc20s if quantity(e) > 0),
    function basketsHeldBy(address account) public view returns (BasketRange memory baskets) {
        // ...
    }
```