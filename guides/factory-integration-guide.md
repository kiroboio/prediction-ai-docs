# PredictionMarketFactory Integration Guide

> **Target Audience:** AI agents and DeFi developers creating prediction markets

> **AI-Optimized Version:** If you have access to the `IPredictionMarketFactory` ABI, see `factory-integration-guide-ai.md` for a 70% shorter guide focused on business logic and non-obvious concepts only.

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Architecture](#architecture)
4. [API Reference](#api-reference)
5. [Market Creation](#market-creation)
6. [Creator Management](#creator-management)
7. [Market Discovery](#market-discovery)
8. [Fee Management](#fee-management)
9. [Module Management](#module-management)
10. [Challenge Period & Invalid Statements](#challenge-period--invalid-statements)
11. [Best Practices](#best-practices)
12. [Code Example](#code-example-ai-powered-market-creator)
13. [Network Configurations](#network-configurations)

---

## Overview

The `PredictionMarketFactory` is the central contract for deploying and managing UMA-based prediction markets. It uses the EIP-1167 minimal proxy pattern (clones) to achieve ~90% gas savings on market deployment.

### Key Features

- **Gas-Efficient Deployment**: Clone pattern reduces deployment costs
- **Creator Tracking**: Comprehensive statistics and performance metrics
- **Rate Limiting**: Prevents spam (1-minute cooldown, 100 markets/day max)
- **Flexible Fees**: Default and custom fee structures per creator
- **Market Discovery**: Filter by category, tags, settlement time, creator
- **Batch Operations**: Create and query multiple markets efficiently
- **Module Validation**: Ensures module compatibility and activation

### Contract Addresses

**Mainnet Networks:**
- **Ethereum Mainnet** (Chain ID: 1): TBD
- **Base Mainnet** (Chain ID: 8453): TBD

**Testnet Networks:**
- **Ethereum Sepolia** (Chain ID: 11155111): TBD
- **Base Sepolia** (Chain ID: 84532): TBD

---

## Quick Start

### 1. Install Dependencies

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./interfaces/IPredictionMarketFactory.sol";
import "./interfaces/IPredictionMarket.sol";
```

### 2. Create Your First Market

```solidity
// Connect to factory
IPredictionMarketFactory factory = IPredictionMarketFactory(FACTORY_ADDRESS);

// Define market parameters
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "Bitcoin price was above $100,000",  // Single statement - system generates TRUE/FALSE variants
    dataSource: "CoinGecko BTC/USD price feed",     // Optional: verification source
    category: "crypto",
    tags: ["bitcoin", "price", "2025"],
    stakeToken: USDC_ADDRESS, // 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 on Ethereum
    stakingDeadline: block.timestamp + 90 days,
    settlementTime: 1735689600, // Dec 31, 2025 00:00:00 UTC
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0, // 0 = use default 48h, or specify custom (24h - 7 days)
    marketCreator: address(0), // address(0) = use msg.sender as creator
    platformOperator: address(0), // address(0) = no platform operator, 100% fees to creator
    creatorFeeToCreatorBps: 0, // 0 = use default (180 bps = 1.8% to creator)
    creatorFeeToPlatformBps: 0  // 0 = use default (0 bps, no platform split)
});

// Create market (pay creation fee if required)
address newMarket = factory.createMarket{value: factory.creationFee()}(params);

// The factory automatically generates:
// TRUE statement:  "As of timestamp 1735689600 according to CoinGecko BTC/USD price feed, the following is TRUE: Bitcoin price was above $100,000"
// FALSE statement: "As of timestamp 1735689600 according to CoinGecko BTC/USD price feed, the following is FALSE: Bitcoin price was above $100,000"
```

### 3. Query Market Information

```solidity
// Get all active markets
address[] memory activeMarkets = factory.getActiveMarketsPaginated(0, 50);

// Get markets by category
address[] memory cryptoMarkets = factory.getMarketsByCategory("crypto");

// Get creator statistics
IPredictionMarketFactory.CreatorProfile memory profile = factory.getCreatorProfile(msg.sender);
```

---

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────┐
│         PredictionMarketFactory (Singleton)         │
├─────────────────────────────────────────────────────┤
│ • Market deployment via EIP-1167 clones             │
│ • Creator registry and statistics                   │
│ • Module validation and compatibility               │
│ • Fee structure management                          │
│ • Market discovery and filtering                    │
└────────────┬────────────────────────────────────────┘
             │ creates (clone)
             │
             ├──→ Market Instance 1 ──→ Modules (AMM, Staking, Oracle)
             ├──→ Market Instance 2 ──→ Modules (AMM, Staking, Oracle)
             └──→ Market Instance N ──→ Modules (AMM, Staking, Oracle)
```

### Module Registry Integration

The factory validates all modules against the `ModuleRegistry`:

1. **Module must exist** in registry
2. **Module must be ACTIVE** status
3. **Modules must be compatible** with each other
4. **Module types must match** (AMM, Staking, Oracle)

### Clone Pattern

Markets are deployed as minimal proxies pointing to a master implementation:

- **Implementation**: Single master contract deployed once
- **Clones**: Lightweight proxies (45 bytes) delegating to implementation
- **Gas Savings**: ~90% reduction in deployment costs
- **Initialization**: Each clone is initialized with unique parameters

---

## API Reference

### Constants

```solidity
uint256 public constant MIN_STAKING_PERIOD = 1 hours;
uint256 public constant MIN_SETTLEMENT_DELAY = 30 minutes;
uint256 public constant MAX_TOTAL_FEE_BPS = 1000; // 10%
uint256 public constant BPS_DENOMINATOR = 10000; // 100%
uint256 public constant CREATION_COOLDOWN = 1 minutes;
uint256 public constant MAX_DAILY_MARKETS = 100;
```

### Data Structures

#### MarketParams

Complete parameters for market creation.

```solidity
struct MarketParams {
    string statement;            // Base statement (e.g., "ETH price was above $4000")
    string dataSource;           // Optional verification source (e.g., "CoinGecko ETH/USD")
    string category;             // Market category (e.g., "crypto", "sports", "politics")
    string[] tags;               // Array of tags for discovery (e.g., ["ethereum", "price"])
    address stakeToken;          // ERC20 token for staking (typically USDC)
    uint256 stakingDeadline;     // Timestamp when staking period ends
    uint256 settlementTime;      // Timestamp when oracle can be queried
    bytes32 ammModuleId;         // AMM module identifier (e.g., CONSTANT_PRODUCT_AMM_V1)
    bytes32 stakingModuleId;     // Staking module identifier (e.g., STANDARD_STAKING_V1)
    bytes32 oracleModuleId;      // Oracle module identifier (e.g., UMA_ORACLE_V3_MODULE)
    MarketType marketType;       // BINARY (only type currently supported)
    bytes moduleParams;          // Additional parameters for modules (optional)
    uint256 settlementActivationWindow; // Duration after settlement time before auto-INVALID (24h - 7 days, 0 = use default 48h)
    address marketCreator;       // Optional: override msg.sender as market creator (address(0) = use msg.sender)
    address platformOperator;    // Optional: platform operator for fee split (address(0) = no platform, 100% to creator)
    uint256 creatorFeeToCreatorBps;    // Optional: custom fee to market creator (0 = use default 180 bps)
    uint256 creatorFeeToPlatformBps;   // Optional: custom fee to platform operator (0 = use default 0 bps)
}
```

**Automatic Statement Generation:**
The factory automatically generates TRUE and FALSE variants from your base statement:
- **TRUE**: `"As of timestamp {settlementTime} according to {dataSource}, the following is TRUE: {statement}"`
- **FALSE**: `"As of timestamp {settlementTime} according to {dataSource}, the following is FALSE: {statement}"`

This ensures consistent formatting and reduces errors from manually crafting both statements.

**Multi-Tier Creator Fee System:**
The system supports splitting the 1.8% creator fee between two parties:
- **Market Creator** (`marketCreator`): The person who defined the market (defaults to msg.sender)
- **Platform Operator** (`platformOperator`): Optional UI provider who facilitated market creation

Set `platformOperator = address(0)` for independent creators (100% fees to creator).
Set custom split with `creatorFeeToCreatorBps` and `creatorFeeToPlatformBps` (must sum to 180 or use 0 for defaults).

**Validation Rules:**
- `statement` must be non-empty
- `category` must be non-empty
- `stakeToken` must be a valid ERC20 contract
- `stakingDeadline` >= `block.timestamp + MIN_STAKING_PERIOD` (1 hour)
- `settlementTime` >= `stakingDeadline + MIN_SETTLEMENT_DELAY` (30 minutes)
- All module IDs must be active in ModuleRegistry
- Modules must be compatible with each other
- If custom creator fees provided: `creatorFeeToCreatorBps + creatorFeeToPlatformBps` must equal 180 (1.8%)
- Fees are controlled by factory (platform manager sets defaults via 48h timelock)

#### CreatorProfile

Creator statistics and metadata.

```solidity
struct CreatorProfile {
    address creator;             // Creator address
    uint256 totalMarketsCreated; // Total markets created all-time
    uint256 activeMarkets;       // Currently active (stakeable) markets
    uint256 settledMarkets;      // Markets that have been settled
    uint256 totalVolumeGenerated;// Total volume across all markets
    uint256 totalFeesEarned;     // Total creator fees earned
    uint256 createdAt;           // Timestamp of first market creation
    uint256 lastActivityAt;      // Timestamp of last activity
    bool isVerified;             // Verification status (set by admin)
    uint256 tier;                // Creator tier (0=normal, 1=silver, 2=gold)
    string metadataURI;          // IPFS URI for creator metadata
}
```

**Creator Tiers:**
- **Tier 0 (Normal)**: Default tier for all creators
- **Tier 1 (Silver)**: Enhanced visibility, potential fee discounts
- **Tier 2 (Gold)**: Premium tier with maximum benefits

#### MarketSummary

Lightweight market information for batch queries.

```solidity
struct MarketSummary {
    address market;              // Market contract address
    string statement;            // Market statement (TRUE variant)
    string category;             // Market category
    uint256 totalVolume;         // Total staked amount (YES + NO)
    uint256 stakingDeadline;     // When staking closes
    uint256 settlementTime;      // When settlement begins
    bool isActive;               // Currently accepting stakes
    bool isSettled;              // Has been settled
}
```

---

## Market Creation

### createMarket

Creates a single prediction market.

```solidity
function createMarket(MarketParams calldata params)
    external
    payable
    returns (address market)
```

**Parameters:**
- `params` - Complete market configuration (see MarketParams structure)

**Returns:**
- `market` - Address of the newly deployed market contract

**Requirements:**
- Contract must not be paused
- Must pay `creationFee` (if set) via `msg.value`
- Must respect rate limits:
  - 1-minute cooldown between markets
  - Maximum 100 markets per 24 hours per creator
- All module IDs must be active and compatible
- Timestamps must satisfy timing constraints
- Fee structure must be valid (if using custom fees)

**Events Emitted:**
```solidity
event MarketCreated(
    address indexed market,
    address indexed creator,
    string category,
    string statementTrue,
    uint256 stakingDeadline,
    uint256 settlementTime
)

// If first market by creator:
event CreatorRegistered(address indexed creator, uint256 timestamp)
```

**Custom Errors:**
```solidity
error InvalidParameters();      // Parameters fail validation
error InvalidModule();           // Module ID not found
error ModuleNotActive();         // Module exists but not active
error IncompatibleModules();     // Modules incompatible
error InvalidTimestamps();       // Timing constraints violated
error ExceedsMaxFees();          // Fees > 10%
```

**Example:**
```solidity
// Create market with default fees
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "ETH will reach $10,000 by end of 2025",
    dataSource: "CoinGecko ETH/USD price feed",
    category: "crypto",
    tags: ["ethereum", "price"],
    stakeToken: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, // USDC on Ethereum
    stakingDeadline: block.timestamp + 60 days,
    settlementTime: 1735689600, // Dec 31, 2025
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    moduleParams: "",
    settlementActivationWindow: 0 // 0 = use default 48h
});

uint256 fee = factory.creationFee();
address market = factory.createMarket{value: fee}(params);
```

---

### batchCreateMarkets

Creates multiple markets in a single transaction.

```solidity
function batchCreateMarkets(MarketParams[] calldata paramsArray)
    external
    payable
    returns (address[] memory markets)
```

**Parameters:**
- `paramsArray` - Array of market parameters (max 10 markets)

**Returns:**
- `markets` - Array of deployed market addresses

**Requirements:**
- Array length must be 1-10
- Must pay `creationFee × array.length` via `msg.value`
- Each market must meet individual creation requirements
- Rate limits apply to total markets created

**Example:**
```solidity
IPredictionMarketFactory.MarketParams[] memory batch = new IPredictionMarketFactory.MarketParams[](3);

// Market 1: BTC price
batch[0] = IPredictionMarketFactory.MarketParams({
    statement: "BTC > $100k by end of 2025",
    dataSource: "CoinGecko BTC/USD price feed",
    category: "crypto",
    tags: ["bitcoin", "price"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 90 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    moduleParams: "",
    settlementActivationWindow: 0 // Use default 48h
});

// Market 2: ETH price
batch[1] = // ... similar structure

// Market 3: SOL price
batch[2] = // ... similar structure

uint256 totalFee = factory.creationFee() * 3;
address[] memory markets = factory.batchCreateMarkets{value: totalFee}(batch);
```

---

## Creator Management

### getCreatorMarkets

Get all markets created by a specific creator.

```solidity
function getCreatorMarkets(address creator) external view returns (address[] memory)
```

---

### getCreatorActiveMarketsPaginated

```solidity
function getCreatorActiveMarketsPaginated(
    address creator,
    uint256 offset,
    uint256 limit
) external view returns (address[] memory)
```

Returns active markets for a creator with pagination (offset/limit).

---

### getCreatorProfile

```solidity
function getCreatorProfile(address creator) external view returns (CreatorProfile memory)
```

Returns creator statistics including total markets, active markets, fees earned, tier, and verification status.

---

### claimAllCreatorFees

```solidity
function claimAllCreatorFees(address creator) external returns (uint256 totalClaimed)
```

Claims all pending creator fees across all settled markets. Callable by creator or owner. Returns total amount claimed.

---

## Market Discovery

### getActiveMarketsPaginated

```solidity
function getActiveMarketsPaginated(uint256 offset, uint256 limit) external view returns (address[] memory)
```

Returns active markets with pagination support.

---

### getMarketsByCategory

```solidity
function getMarketsByCategory(string calldata category) external view returns (address[] memory)
```

Returns all markets in a specific category (e.g., "crypto", "sports", "politics").

---

### getMarketsByTagsPaginated

```solidity
function getMarketsByTagsPaginated(
    string[] calldata tags,
    uint256 offset,
    uint256 limit
) external view returns (address[] memory)
```

Returns markets matching any of the provided tags (OR logic) with pagination.

---

### getMarketsSettlingBetweenPaginated

```solidity
function getMarketsSettlingBetweenPaginated(
    uint256 startTime,
    uint256 endTime,
    uint256 offset,
    uint256 limit
) external view returns (address[] memory)
```

Returns markets settling within a time range with pagination.

---

### getBatchMarketSummaries

```solidity
function getBatchMarketSummaries(address[] calldata markets) external view returns (MarketSummary[] memory)
```

Returns lightweight market summaries for multiple markets in a single call. Perfect for UI list views.

---

## Fee Management

The system has two layers of fee control:

1. **Platform-Controlled Fees**: Set by factory owner via 48h timelock (platform fee + total creator fee + activator fee)
2. **Creator Fee Split**: Customizable by market creators (how to split the 1.8% creator fee between creator and platform operator)

### Platform-Controlled Default Fees

These are set by the factory owner (platform manager) and apply to all new markets unless overridden:

**Mainnet Default:**
- **Platform Fee**: 1.0% (100 bps) → factory treasury
- **Total Creator Fee**: 1.8% (180 bps) → split between creator and platform operator (if any)
- **Activator Fee**: 2.0% (200 bps) → distributed among bond pool contributors
- **Total**: 4.8%

**Testnet Default:**
- **Platform Fee**: 0.5% (50 bps) → factory treasury
- **Total Creator Fee**: 1.8% (180 bps) → split between creator and platform operator (if any)
- **Activator Fee**: 2.0% (200 bps) → distributed among bond pool contributors
- **Total**: 4.3%

**Important:** Only the factory owner can change these default fee rates, and changes require a 48-hour timelock for security.

### Creator Fee Split (Per-Market Customization)

Market creators can customize how the 1.8% creator fee is split between:
- **Market Creator**: The person who conceptualized and defined the market
- **Platform Operator**: The UI provider or platform that facilitated creation

See the [Multi-Tier Creator Fee System](#multi-tier-creator-fee-system) section for detailed examples and use cases.

**Key Points:**
- Platform fee (1.0%) is NOT customizable by creators
- Activator fee (2.0%) is NOT customizable by creators
- Only the creator fee split (1.8%) can be customized per market
- Custom split must sum to exactly 180 bps (1.8%)

### getDefaultFees

```solidity
function getDefaultFees() external view returns (IPredictionMarket.FeeInfo memory)
```

Returns current platform-controlled default fee structure (platform, creator total, activator).

---

### getCreatorFees

```solidity
function getCreatorFees(address creator) external view returns (IPredictionMarket.FeeInfo memory)
```

Returns custom fees for a creator if the factory owner has set creator-specific overrides, otherwise returns default fees.

**Note:** This function returns platform-level fee overrides set by the factory owner, NOT the per-market creator fee splits. Creator fee splits are set when creating each individual market via `MarketParams`.

---

## Module Management

### isModuleActive

```solidity
function isModuleActive(bytes32 moduleId) external view returns (bool)
```

Returns true if module is registered and active.

---

### getModuleAddress

```solidity
function getModuleAddress(bytes32 moduleId) external view returns (address)
```

Returns implementation address for a module.

---

## Multi-Tier Creator Fee System

The factory supports a flexible multi-tier creator fee system that enables fee splitting between market creators and platform operators. This allows platforms to monetize their infrastructure while rewarding content creators.

### Overview

The system recognizes two distinct roles:

1. **Market Creator** (`marketCreator`): The person who defined the market statement, parameters, and conceptualized the prediction
2. **Platform Operator** (`platformOperator`): The UI provider or platform that facilitated market creation and provides infrastructure

The total creator fee is **1.8%** (180 basis points), which can be split between these two parties using custom basis point allocations.

### How It Works

When creating a market via `createMarket()`, you can specify:

```solidity
struct MarketParams {
    // ... other fields ...
    address marketCreator;               // Defaults to msg.sender if address(0)
    address platformOperator;            // No platform operator if address(0)
    uint256 creatorFeeToCreatorBps;      // Defaults to 180 (1.8%) if 0
    uint256 creatorFeeToPlatformBps;     // Defaults to 0 (0%) if 0
}
```

**Default Behavior:**
- If `marketCreator = address(0)`, the factory uses `msg.sender` as the market creator
- If `platformOperator = address(0)`, there's no platform split (100% fees to creator)
- If both fee fields are `0` **with** a `platformOperator`, the factory uses default 50/50 split (90 bps each)
- If both fee fields are `0` **with** a `platformOperator` set, but you want to eliminate fees, use special case below

**Custom Fee Splits:**
- To split fees, set both `creatorFeeToCreatorBps` and `creatorFeeToPlatformBps`
- The sum must equal exactly **180** (1.8% total) **OR** both be 0 to eliminate creator fees
- **Special case: 0+0 with platformOperator** eliminates the 1.8% creator fee entirely (reduces total fee from 5.0% to 3.2%)
- **Supported splits**: 180+0, 0+180, or any custom split that sums to 180

### Common Scenarios

#### Scenario 1: Independent Creator (Default)

A user creates a market directly without going through a platform.

```solidity
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "Bitcoin price was above $100,000",
    dataSource: "CoinGecko BTC/USD price feed",
    category: "crypto",
    tags: ["bitcoin", "price"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 90 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0,
    marketCreator: address(0),        // Defaults to msg.sender
    platformOperator: address(0),     // No platform operator
    creatorFeeToCreatorBps: 0,        // Defaults to 180 (1.8%)
    creatorFeeToPlatformBps: 0        // Defaults to 0 (0%)
});

address market = factory.createMarket{value: factory.creationFee()}(params);
```

**Result:**
- Market Creator: `msg.sender`
- Creator Fee: 1.8% (180 bps) → 100% to creator
- Platform Fee: 0%

---

#### Scenario 2: 50/50 Partnership

A platform and creator agree to split fees equally.

```solidity
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "Ethereum reaches $10,000 by end of 2025",
    dataSource: "CoinGecko ETH/USD price feed",
    category: "crypto",
    tags: ["ethereum", "price"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 60 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0,
    marketCreator: CONTENT_CREATOR_ADDRESS, // Explicit creator address
    platformOperator: PLATFORM_ADDRESS,     // Platform infrastructure provider
    creatorFeeToCreatorBps: 90,             // 0.9% to creator
    creatorFeeToPlatformBps: 90             // 0.9% to platform (90 + 90 = 180)
});

address market = factory.createMarket{value: factory.creationFee()}(params);
```

**Result:**
- Market Creator: `CONTENT_CREATOR_ADDRESS`
- Platform Operator: `PLATFORM_ADDRESS`
- Creator Fee: 0.9% (90 bps) → content creator
- Platform Fee: 0.9% (90 bps) → platform operator
- **Total: 1.8% (180 bps)**

---

#### Scenario 3: Platform-Heavy Split (70/30)

A platform provides significant infrastructure and takes a larger share.

```solidity
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "S&P 500 closes above 6000 on Dec 31, 2025",
    dataSource: "Yahoo Finance S&P 500 closing price",
    category: "finance",
    tags: ["stocks", "sp500", "markets"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 180 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0,
    marketCreator: COMMUNITY_MEMBER_ADDRESS,
    platformOperator: MAJOR_PLATFORM_ADDRESS,
    creatorFeeToCreatorBps: 54,             // 0.54% to creator (30%)
    creatorFeeToPlatformBps: 126            // 1.26% to platform (70%) (54 + 126 = 180)
});

address market = factory.createMarket{value: factory.creationFee()}(params);
```

**Result:**
- Creator receives: 30% of 1.8% = **0.54%** (54 bps)
- Platform receives: 70% of 1.8% = **1.26%** (126 bps)
- **Total: 1.8% (180 bps)**

---

#### Scenario 4: Creator-Heavy Split (70/30)

A platform provides light infrastructure; creator takes majority.

```solidity
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "Lakers win NBA Championship in 2025",
    dataSource: "Official NBA Championship results",
    category: "sports",
    tags: ["basketball", "nba", "lakers"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 120 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0,
    marketCreator: EXPERT_ANALYST_ADDRESS,
    platformOperator: LIGHTWEIGHT_PLATFORM_ADDRESS,
    creatorFeeToCreatorBps: 126,            // 1.26% to creator (70%)
    creatorFeeToPlatformBps: 54             // 0.54% to platform (30%) (126 + 54 = 180)
});

address market = factory.createMarket{value: factory.creationFee()}(params);
```

**Result:**
- Creator receives: 70% of 1.8% = **1.26%** (126 bps)
- Platform receives: 30% of 1.8% = **0.54%** (54 bps)
- **Total: 1.8% (180 bps)**

---

#### Scenario 5: External Payment Arrangement (0+0)

An external contract or platform handles creator/operator payments off-chain, eliminating on-chain creator fees entirely.

```solidity
IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
    statement: "Tesla stock closes above $500 on Dec 31, 2025",
    dataSource: "NASDAQ Tesla closing price",
    category: "stocks",
    tags: ["tesla", "stocks", "tsla"],
    stakeToken: USDC_ADDRESS,
    stakingDeadline: block.timestamp + 150 days,
    settlementTime: 1735689600,
    ammModuleId: keccak256("CONSTANT_PRODUCT_AMM_V1"),
    stakingModuleId: keccak256("STANDARD_STAKING_V1"),
    oracleModuleId: keccak256("UMA_ORACLE_V3_MODULE"),
    marketType: IPredictionMarketFactory.MarketType.BINARY,
    moduleParams: "",
    settlementActivationWindow: 0,
    marketCreator: ACTUAL_CREATOR_ADDRESS,
    platformOperator: EXTERNAL_CONTRACT_ADDRESS,  // External contract managing payments
    creatorFeeToCreatorBps: 0,                    // No on-chain creator fee
    creatorFeeToPlatformBps: 0                    // No on-chain platform fee
});

address market = factory.createMarket{value: factory.creationFee()}(params);
```

**Result:**
- Creator Fee: **ELIMINATED** (0% on-chain)
- Platform Fee: 1.2% (120 bps) → factory treasury
- Activator Fee: 2.0% (200 bps) → bond pool contributors
- **Total Fee: 3.2%** (instead of 5.0%)

**Use Case:**
- External contract pays creator and platform operator via separate mechanism (off-chain, different token, subscription model, etc.)
- Users benefit from **1.8% lower fees** (3.2% vs 5.0%)
- Useful for platforms with alternative monetization strategies

**Important Notes:**
- The 1.8% is truly eliminated from on-chain fees (not just redistributed)
- External contract responsible for paying creator and platform operator through other means
- Factory treasury still receives 1.2% platform fee
- Bond pool contributors still receive 2.0% activator fee

---

### Validation Rules

The factory enforces flexible validation on creator fee configurations:

1. **Sum Requirement**: `creatorFeeToCreatorBps + creatorFeeToPlatformBps` must:
   - Equal **180** (1.8%) for standard splits, OR
   - Both be **0** to eliminate creator fees entirely (reduces total fee to 3.2%)
   - Supports **180+0** (100% to creator), **0+180** (100% to platform), **0+0** (external payment)

2. **Platform Operator Requirement**: If `creatorFeeToPlatformBps > 0`, then `platformOperator` must be non-zero address

3. **External Payment Mode (0+0)**: When both fees are 0 with a `platformOperator`:
   - Creator fee is eliminated from on-chain fees
   - Total fee reduced from 5.0% to 3.2%
   - External contract responsible for off-chain payment arrangements

**Example Validation:**

```solidity
// ✅ VALID: Default (no custom fees, no platform operator)
marketCreator: address(0),
platformOperator: address(0),
creatorFeeToCreatorBps: 0,
creatorFeeToPlatformBps: 0
// Result: 180 bps to msg.sender, total fee = 5.0%

// ✅ VALID: External payment (0+0 with platform operator)
marketCreator: CREATOR_ADDR,
platformOperator: PLATFORM_ADDR,
creatorFeeToCreatorBps: 0,
creatorFeeToPlatformBps: 0
// Result: Creator fee ELIMINATED, total fee = 3.2%

// ✅ VALID: 100% to creator (180+0)
marketCreator: CREATOR_ADDR,
platformOperator: PLATFORM_ADDR,
creatorFeeToCreatorBps: 180,
creatorFeeToPlatformBps: 0
// Result: 180 bps to creator, 0 to platform, total fee = 5.0%

// ✅ VALID: 100% to platform (0+180)
marketCreator: CREATOR_ADDR,
platformOperator: PLATFORM_ADDR,
creatorFeeToCreatorBps: 0,
creatorFeeToPlatformBps: 180
// Result: 0 to creator, 180 bps to platform, total fee = 5.0%

// ✅ VALID: 50/50 split
marketCreator: CREATOR_ADDR,
platformOperator: PLATFORM_ADDR,
creatorFeeToCreatorBps: 90,
creatorFeeToPlatformBps: 90
// Result: 90 bps to creator, 90 bps to platform, total fee = 5.0%

// ❌ INVALID: Sum doesn't equal 180 (and not 0+0)
creatorFeeToCreatorBps: 100,
creatorFeeToPlatformBps: 100
// Error: "Custom fee split must equal total creator fee"

// ❌ INVALID: Platform fee without platform operator
platformOperator: address(0),
creatorFeeToCreatorBps: 0,
creatorFeeToPlatformBps: 180
// Error: Cannot have platform fees without platform operator
```

---

### Fee Claiming

Both market creators and platform operators can claim their fees:

```solidity
// Creator claims fees from all their markets
factory.claimAllCreatorFees(CREATOR_ADDRESS);

// Platform operator claims fees from all markets they're operator for
factory.claimAllCreatorFees(PLATFORM_ADDRESS);
```

The factory tracks fees separately for each role, ensuring proper accounting even when the same market has both a creator and platform operator.

---

### Use Cases

**1. Decentralized Platforms (No Platform Operator)**
- Community-driven markets where creators keep 100% of fees
- Set `platformOperator = address(0)`

**2. Curated Platforms (Platform-Heavy)**
- Professional platforms providing infrastructure, data feeds, UX
- Typical split: 70% platform / 30% creator (126/54 bps)

**3. Creator-First Platforms (Creator-Heavy)**
- Platforms focusing on empowering content creators
- Typical split: 70% creator / 30% platform (126/54 bps)

**4. Partnership Models (Equal Split)**
- Joint ventures between platforms and expert analysts
- Typical split: 50% / 50% (90/90 bps)

---

## Challenge Period & Invalid Statements

Two mechanisms protect against invalid statements:

1. **Settlement Activation Window** (24h-7d, default 48h): Auto-INVALID if no one initiates settlement
2. **Challenge Period** (7 days): Auto-INVALID if first assertion fails and no second assertion made

### Mechanism 1: Settlement Activation Window

**When:** No one initiates settlement after settlement time passes.

**Configuration:**
- Range: 24 hours to 7 days (set at market creation)
- Default: 48 hours (use `settlementActivationWindow = 0`)

**How it works:**
1. Settlement time passes
2. Wait for configured window (24h-7d)
3. If no settlement initiated, anyone can call `finalizeAsInvalidNoSettlement()`
4. Market marked INVALID, users get proportional refunds

---

### Mechanism 2: Challenge Period

**When:** First settlement assertion fails (disputed by UMA oracle).

**How it works:**
1. First assertion is disputed and fails
2. 7-day challenge period begins
3. Anyone can make a second assertion with a bond
4. If no second assertion made within 7 days, anyone can call `finalizeAsInvalid()`
5. Market marked INVALID, users get full refunds

**Economic Incentive:** If statement is valid, making second assertion is "easy money". If invalid/nonsensical, no one will risk their bond.

---

### Market Outcomes

- **YES** (1): Statement was true
- **NO** (2): Statement was false
- **INVALID** (3): Statement nonsensical/unverifiable (full refunds)
- **PENDING** (0): Not yet settled

---

## Best Practices

### Market Creation
- Use clear, unambiguous statements
- Set reasonable staking periods (≥7 days recommended)
- Choose settlement times after events resolve
- Use widely accepted tokens (USDC recommended)
- Categorize and tag appropriately
- Respect rate limits (1-minute cooldown, 100 markets/day max)

### Fee Configuration
- Keep total fees ≤5% for better UX
- Default fees: 3.5% mainnet, 2.5% testnet
- All fees controlled by platform manager via 48h timelock
- Creators cannot set custom fees (security by design)

### Gas Optimization
- Use `batchCreateMarkets` for multiple markets
- Use paginated queries for large datasets
- Cache module IDs instead of recalculating

### Security
- Validate all inputs before market creation
- Verify module IDs are active
- Ensure timestamps are in the future
- Use try/catch for error handling

---

## Code Example: AI-Powered Market Creator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./interfaces/IPredictionMarketFactory.sol";

contract AIMarketCreator {
    IPredictionMarketFactory public immutable factory;
    address public immutable usdcToken;

    // Standard module IDs
    bytes32 public constant AMM_MODULE = keccak256("CONSTANT_PRODUCT_AMM_V1");
    bytes32 public constant STAKING_MODULE = keccak256("STANDARD_STAKING_V1");
    bytes32 public constant ORACLE_MODULE = keccak256("UMA_ORACLE_V3_MODULE");

    event AIMarketCreated(
        address indexed market,
        string category,
        uint256 stakingPeriod,
        uint256 settlementTime
    );

    constructor(address _factory, address _usdc) {
        factory = IPredictionMarketFactory(_factory);
        usdcToken = _usdc;
    }

    /**
     * @notice Creates a market with AI-generated parameters
     * @param statement The base market statement
     * @param dataSource Optional data source for verification
     * @param category Market category
     * @param tags Array of tags
     * @param stakingPeriodDays Staking period in days
     * @param settlementTimestamp When to settle the market
     * @param settlementWindow Custom settlement activation window (0 = use default 48h)
     */
    function createAIMarket(
        string calldata statement,
        string calldata dataSource,
        string calldata category,
        string[] calldata tags,
        uint256 stakingPeriodDays,
        uint256 settlementTimestamp,
        uint256 settlementWindow
    ) external payable returns (address market) {
        // Validate inputs
        require(bytes(statement).length > 0, "Empty statement");
        require(stakingPeriodDays >= 1, "Staking period too short");
        require(settlementTimestamp > block.timestamp, "Settlement in past");

        // Calculate staking deadline
        uint256 stakingDeadline = block.timestamp + (stakingPeriodDays * 1 days);
        require(settlementTimestamp >= stakingDeadline + 30 minutes, "Invalid timing");

        // Build parameters
        IPredictionMarketFactory.MarketParams memory params = IPredictionMarketFactory.MarketParams({
            statement: statement,
            dataSource: dataSource,
            category: category,
            tags: tags,
            stakeToken: usdcToken,
            stakingDeadline: stakingDeadline,
            settlementTime: settlementTimestamp,
            ammModuleId: AMM_MODULE,
            stakingModuleId: STAKING_MODULE,
            oracleModuleId: ORACLE_MODULE,
            moduleParams: "",
            settlementActivationWindow: settlementWindow
        });

        // Create market
        market = factory.createMarket{value: msg.value}(params);

        emit AIMarketCreated(market, category, stakingPeriodDays, settlementTimestamp);
    }

    /**
     * @notice Get creator statistics
     */
    function getMyStats() external view returns (
        uint256 totalMarkets,
        uint256 activeMarkets,
        uint256 totalFees
    ) {
        IPredictionMarketFactory.CreatorProfile memory profile = factory.getCreatorProfile(address(this));
        return (
            profile.totalMarketsCreated,
            profile.activeMarkets,
            profile.totalFeesEarned
        );
    }
}
```

---

## Network Configurations

| Network | Chain ID | USDC Address | UMA OOv3 | UMA Finder | Fees |
|---------|----------|--------------|----------|------------|------|
| **Ethereum Mainnet** | 1 | `0xA0b86991...06eB48` | `0xfb55F43f...ebDC0dE` | `0x40f941E4...7D77c3` | 3.5% |
| **Base Mainnet** | 8453 | `0x833589fC...bdA02913` | `0x2aBf1Bd7...5Eec75AF500c` | `0x7E6d9618...97e2512` | 3.5% |
| **Sepolia Testnet** | 11155111 | `0x94a9D9AC...85d5E4C8` | `0xFd9e2642...F31924944` | `0xf4C48eDA...815F8f13` | 2.5% |
| **Base Sepolia** | 84532 | `0x036CbD53...1eC2318f3dCF7e` | `0x0F7fC5E6...7b57388deE` | `0xfF4Ec014...456F7dB` | 2.5% |

**Factory & Module Registry**: TBD (deployment pending)

---

## Resources

- **UMA Docs**: https://docs.uma.xyz
- **Contracts**: `src/PredictionMarketFactory.sol`, `src/PredictionMarket.sol`, `src/ModuleRegistry.sol`
