# Liquidity Generation Event (LGE) Hook

A novel token launchpad mechanism built on Uniswap v4 using hooks for fair token distribution and automatic liquidity creation.

## 🚀 Overview

The LGE Hook is a specialized Uniswap v4 hook that enables:
- Fair token launches with linear distribution
- Automatic liquidity pool creation
- On-chain price discovery through dynamic tick placement
- Protection against failed launches with full refund mechanism

## 📊 Core Mechanics

### Token Release Schedule

| Parameter | Value |
|-----------|-------|
| **Total Supply** | 17,745,440,000 tokens |
| **Release Period** | 5,000 blocks (~1 hour on Ethereum) |
| **Release Rate** | 3,549,088 tokens/block |
| **Trading Lock** | 6,000 blocks |

### How It Works

```mermaid
graph LR
    A[Deploy LGE] --> B[Tokens Stream Over Time]
    B --> C[Users Send ETH]
    C --> D{Success?}
    D -->|100% Claimed| E[Users Get LP Tokens]
    D -->|<100% Claimed| F[Full ETH Refund]
```

1. **Launch** - Deploy contract with fixed token supply
2. **Contribute** - Users send ETH to claim streaming tokens
   - 50% ETH → Pairs with tokens for LP
   - 50% ETH → Commitment fee (retained on success)
3. **Settlement**
   - ✅ Success: Users receive LP tokens
   - ❌ Failure: Full ETH refund

## 💡 Dynamic Pricing Model

The system uses a bonding curve to map claim size to Uniswap tick placement:

![LGE Bonding Curve](LGE-curve-image.jpg)

| Claim Size | Tick Range | ETH Cost |
|------------|------------|----------|
| Small (0-4.4%) | Very negative | High (expensive) |
| Medium (4.4-15.6%) | Near zero | Balanced |
| Large (15.6-100%) | Positive | Lower (cheaper) |

## 🔧 Implementation

### Tick Calculation

```solidity
function tickForClaim(uint256 tokensClaimed) public pure returns (int256 Tick) {
    // Constants
    uint256 TOTAL_SUPPLY = 17_745_440_000;
    uint256 LOW_THRESHOLD = 774_544_000;      // ~4.4% of supply
    uint256 HIGH_THRESHOLD = 2_774_544_000;   // ~15.6% of supply
    int256 TMAX = 887_272;                    // Maximum tick
    int256 BASE_TICK = 212_985;               // Base tick (parameterizable)

    if (tokensClaimed < LOW_THRESHOLD) {
        // Smooth curve: -TMAX to BASE_TICK
        int256 x = int256(tokensClaimed * 1e6 / LOW_THRESHOLD);
        int256 oneMinusX = int256(1e6) - x;
        
        Tick = (-TMAX * oneMinusX * oneMinusX / 1e12)
             + (BASE_TICK * x * x / 1e12);
    } 
    else if (tokensClaimed <= HIGH_THRESHOLD) {
        // Plateau around BASE_TICK
        int256 y = int256((tokensClaimed - LOW_THRESHOLD) * 1e6 / 
                         (HIGH_THRESHOLD - LOW_THRESHOLD));
        
        Tick = BASE_TICK * (900_000 + (y / 10)) / 1_000_000;
    } 
    else {
        // Rising curve: BASE_TICK to TMAX
        int256 z = int256((tokensClaimed - HIGH_THRESHOLD) * 1e6 / 
                         (TOTAL_SUPPLY - HIGH_THRESHOLD));
        
        Tick = BASE_TICK + (TMAX - BASE_TICK) * z * z / 1e12;
    }

    // Snap to tick spacing
    Tick = (Tick / 200) * 200;
}
```

### ETH Requirement Calculation

```solidity
// 1. Calculate claim percentage
p = (tokensClaimed * 100) / 100_000_000_000

// 2. Get tick from bonding curve
tick = tickForClaim(tokensClaimed)

// 3. Calculate base ETH requirement
price = 1.0001^tick
ethRequired = tokensClaimed / price

// 4. Apply 2x multiplier (prevents LP discount)
userMustSend = ethRequired * 2
```

## 📈 Bonding Curve Properties

```
Tick
  ^
  |     _______________/
  |    /
  |   /
  |  /
  | /
  |/_________________
  +-------------------> % Claimed
  0%   4.4%  15.6%  100%
```

- **Phase 1 (0-4.4%)**: Steep rise from -887,272 to 212,985
- **Phase 2 (4.4-15.6%)**: Plateau around 212,985
- **Phase 3 (15.6-100%)**: Gradual rise to 887,272

## ⚙️ Technical Details

### Uniswap Integration
- Uses `IPoolManager.mint()` for liquidity creation
- Implements `ModifyLiquidityParams` with ±25,000 tick range
- Tick spacing: 200 (Uniswap v4 compatible)

### Key Features
- ✅ Linear token release over blocks
- ✅ Automatic LP creation
- ✅ On-chain price discovery
- ✅ Refund mechanism for failed launches
- ✅ No front-running (fixed release schedule)
- ✅ Fair distribution (bonding curve incentives)

## 🛠️ Configuration

```solidity
// Parameterizable values
BASE_TICK = 212_985;  // Can be derived from target market cap
TICK_SPACING = 200;   // Uniswap v4 requirement
LIQUIDITY_RANGE = 25_000;  // ± from median tick
```

## 📚 Resources

- [Uniswap v4 Hooks Documentation](https://docs.uniswap.org/contracts/v4/overview)
- [Tick Math Reference](https://docs.uniswap.org/contracts/v4/reference/core/libraries/TickMath)
- Similar projects: Clanker

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📄 License

[Insert License Here]

---

**Note**: This is an experimental protocol. Use at your own risk. Always conduct thorough testing before mainnet deployment.
