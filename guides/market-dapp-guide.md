# PredictionMarket DApp Developer Guide

> **Target Audience:** DeFi app developers building prediction market interfaces

> **AI-Optimized Version:** If you have access to the `IPredictionMarket` ABI, see `market-dapp-guide-ai.md` for a shorter guide focused on business logic, edge cases, and non-obvious concepts only.

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Market Lifecycle](#market-lifecycle)
4. [API Reference](#api-reference)
5. [Staking & AMM](#staking--amm)
6. [Settlement & Oracle](#settlement--oracle)
7. [Rewards & Claims](#rewards--claims)
8. [Module System](#module-system)
9. [Integration Examples](#integration-examples)
10. [UI/UX Best Practices](#uiux-best-practices)
11. [Error Handling](#error-handling)
12. [Security Considerations](#security-considerations)

---

## Overview

Individual market contract for binary prediction markets with modular architecture.

**Features:** Virtual tokens, constant product AMM (x*y=k), UMA settlement, multi-tier fees, slippage protection, timestamp-based safety (2-hour buffer)

---

## Quick Start

```typescript
// 1. Connect
const market = new ethers.Contract(marketAddress, PredictionMarketABI, signer);

// 2. Get info
const info = await market.getMarketInfo();
const [yesPrice, noPrice] = await market.getCurrentPrices();
const isActive = await market.isActive();

// 3. Stake
const amount = ethers.utils.parseUnits("100", 6);
await token.approve(marketAddress, amount);
await market.stake(
    amount,
    true, // YES
    amount.mul(95).div(100), // 5% slippage
    Math.floor(Date.now() / 1000) + 300 // 5min deadline
);

// 4. Claim (after settlement)
if (info.isSettled) {
    await market.claimRewards();
}
```

---

## Market Lifecycle

### Phase 1: Staking

**Duration:** Until `stakingDeadline + 2hr TIMESTAMP_BUFFER`

**Actions:** Stake on YES/NO, AMM adjusts prices

**Multi-Chain Protection:** 2-hour timestamp buffer protects against manipulation across all chains

---

### Phase 2: Settlement

**Start:** `settlementTime + 2hr TIMESTAMP_BUFFER`

#### Activation Window (24h-7d, default 48h)
- Anyone can initiate settlement during this window
- If no one initiates → market marked INVALID via `finalizeAsInvalidNoSettlement()`
- Window length set at market creation (24h for simple markets, 7d for complex)

#### Commit-Reveal Settlement
1. **Commit**: `commitSettlement(getCommitmentHash(assertingTrue, secret))`
2. **Wait**: 10 minutes minimum
3. **Reveal**: `initiateSettlement(bond, assertingTrue, secret)` → Creates UMA assertion
4. **UMA Challenge**: 2 hours for disputes
   - If unchallenged → finalize settlement
   - If disputed and fails → 7-day challenge period

#### Challenge Period (If First Assertion Fails)
- **Duration**: 7 days
- **Options**:
  - Someone calls `initiateSecondAssertion(bond)` → 2hr UMA period → settle
  - No one asserts → call `finalizeAsInvalid()` → all get refunds

#### Bond Pool Settlement (Alternative Method)

**Crowdsourced settlement** - Instead of a single activator posting the full bond, participants can pool funds together.

**Two-Stage Contribution Period:**

1. **Stage 1: Participants Only** (0-24 hours after settlement time)
   - Only users who staked in the market can contribute
   - Time weight decays linearly: **2.0x → 1.0x** (20000 → 10000 basis points)
   - Earlier contributions get higher reward multipliers
   - Contribution caps: 30% of required bond (general), 10% (creator), 0% (platform)

2. **Stage 2: Public** (24-48 hours after settlement time)
   - Anyone can contribute
   - Time weight decays linearly: **1.0x → 0.5x** (10000 → 5000 basis points)
   - Same contribution caps apply
   - Lower rewards due to later timing

**How It Works:**

```
1. Settlement time passes (+2hr buffer)
2. Contributors call `contributeToPool(outcome, amount)` for YES or NO pool
3. First pool to reach `requiredBond` can activate
4. Anyone calls `finalizePoolSettlement(outcome)` → Creates UMA assertion
5. UMA Challenge Period (2 hours)
6. Anyone calls `finalizeSettlement()` (same as commit-reveal)
7. If pool wins:
   - Contributors get bonds back automatically
   - Contributors share activator fees proportionally (weighted by time × amount)
8. If pool loses:
   - Contributors can call `refundPoolContribution(outcome)` to reclaim funds
   - Bonds from winning pool go to oracle per UMA rules
```

**Reward Distribution (Winning Pool):**
- Shares = contribution amount × time weight
- Your reward = (your shares / total shares) × activator fees
- Higher time weight = higher share of rewards
- Example: Contribute $50 at 1.5x weight = 75 shares

**Contribution Limits:**
- **Maximum Contributors**: 1000 per pool (prevents DoS)
- **General User**: 30% of required bond
- **Market Creator**: 10% of required bond
- **Platform Treasury**: 0% (blocked from contributing)

**When to Use Bond Pool:**
- Required bond is too high for single activator
- Want to share risk across multiple participants
- Early contributor seeking time-weight rewards
- Community-driven settlement preferred

#### Finalization
- **Activator**: Can finalize after 1 hour
- **Anyone**: Can finalize after 24 hours
- Sets outcome (YES/NO/INVALID), distributes fees from losing side

---

### Phase 3: Claiming

Call `claimRewards()` after settlement.

**Outcomes:**
- **YES Wins**: YES holders split NO stakes (minus fees) proportionally
- **NO Wins**: NO holders split YES stakes (minus fees) proportionally
- **INVALID**: All get exact original stake back (`yesStakeAmount + noStakeAmount`)

**Emergency Mode** (if stuck):
- Factory can activate 30+ days after settlement time
- Users call `emergencyWithdraw()` after 7-day wait
- Get proportional refund: `(userTokens / totalTokens) × totalPool`

---

## API Reference

### Data Structures

#### MarketInfo

Complete market state and configuration.

```solidity
struct MarketInfo {
    string statementTrue;               // Statement describing TRUE outcome
    string statementFalse;              // Statement describing FALSE outcome
    address creator;                    // Market creator address
    IERC20 stakeToken;                  // Token used for staking (e.g., USDC)
    uint256 stakingDeadline;            // When staking period ends (Unix timestamp)
    uint256 settlementTime;             // When settlement can begin (Unix timestamp)
    uint256 totalYesStake;              // Total tokens staked on YES
    uint256 totalNoStake;               // Total tokens staked on NO
    uint256 totalYesTokens;             // Total virtual YES tokens issued
    uint256 totalNoTokens;              // Total virtual NO tokens issued
    IOracleModule.MarketOutcome outcome;// Final outcome (PENDING/YES/NO/INVALID)
    bool isInitialized;                 // Market has been initialized
    bool isSettled;                     // Market has been settled
    bytes32 settlementRequestId;        // Oracle request identifier
}
```

#### UserPosition

User's position in the market.

```solidity
struct UserPosition {
    uint256 yesTokens;      // Virtual YES tokens held
    uint256 noTokens;       // Virtual NO tokens held
    uint256 yesStakeAmount; // Total USDC staked on YES (for INVALID refunds)
    uint256 noStakeAmount;  // Total USDC staked on NO (for INVALID refunds)
    uint256 claimedRewards; // Amount already claimed (0 if not claimed)
    uint256 lastStakeTime;  // Timestamp of last stake
}
```

#### FeeInfo

Fee configuration for the market with **multi-tier creator fee split**.

```solidity
struct FeeInfo {
    uint64 platformFeeBps;              // Platform fee (120 = 1.2%)
    uint64 activatorFeeBps;             // Activator fee (200 = 2.0%)
    uint64 creatorFeeToCreatorBps;      // Fee to market creator (0-180 bps)
    uint64 creatorFeeToPlatformBps;     // Fee to platform operator (0-180 bps)
    uint64 totalCreatorFeeBps;          // Sum of above two (MUST = 180 bps)
    uint64 totalFeeBps;                 // Total fees (500 = 5.0%)
}
// Gas optimized: Packed into 2 storage slots (saves ~80k gas per market)
```

**Multi-Tier Creator Fee System:**
- The 1.8% creator fee can be split between **market creator** and **platform operator**
- Common splits: 100/0 (independent), 50/50 (partnership), 70/30 (platform-heavy)
- Either party can call `claimCreatorFees()` to trigger distribution to both
- Event `CreatorFeesClaimed` shows individual amounts for each party

#### MarketOutcome (Enum)

```solidity
enum MarketOutcome {
    PENDING,  // Not yet settled
    YES,      // YES outcome won
    NO,       // NO outcome won
    INVALID   // Market declared invalid
}
```

#### SettlementStage (Enum)

```solidity
enum SettlementStage {
    NOT_STARTED,        // Settlement time not reached
    PARTICIPANTS_ONLY,  // 24h window for participants (2x→1x time weight)
    PUBLIC              // 24h window for public (1x→0.5x time weight)
}
```

#### BondPool

State of the bond pooling system for crowdsourced settlement.

```solidity
struct BondPool {
    uint256 yesPool;                // Total contributions to YES bond pool
    uint256 noPool;                 // Total contributions to NO bond pool
    uint256 requiredBond;           // Required bond amount (from oracle)
    SettlementStage stage;          // Current settlement stage
    uint256 stageStartTime;         // When current stage started
    bool yesActivated;              // Whether YES pool reached required bond
    bool noActivated;               // Whether NO pool reached required bond
    uint256 yesContributorCount;    // Number of YES contributors
    uint256 noContributorCount;     // Number of NO contributors
}
```

#### BondContribution

Individual user's contribution to a bond pool.

```solidity
struct BondContribution {
    address contributor;            // Address of contributor
    uint256 amount;                 // Amount contributed
    uint256 timestamp;              // When contribution was made
    bool outcome;                   // True = YES, False = NO
    uint256 timeWeight;             // Time weight for fee distribution (in basis points)
    uint256 shares;                 // Time-weighted shares (amount × timeWeight)
}
```

---

### View Functions

```solidity
// Get all market data
function getMarketInfo() external view returns (MarketInfo memory)
function getUserPosition(address user) external view returns (UserPosition memory)

// Check status
function isActive() external view returns (bool) // True if staking open (timestamp check with 2hr buffer)
function getTotalValueLocked() external view returns (uint256)
function getMarketStats() external view returns (uint256 participants, uint256 volume, uint256 yesRatio)

// Prices and rewards
function getCurrentPrices() external view returns (uint256 yesPrice, uint256 noPrice) // In basis points
function calculatePotentialReward(address user, MarketOutcome outcome) external view returns (uint256)

// Configuration
function getFeeInfo() external view returns (FeeInfo memory)
function getModules() external view returns (address amm, address staking, address oracle)

// Bond Pool
function getBondPool() external view returns (BondPool memory)
function getUserContribution(address user, bool outcome) external view returns (BondContribution memory)
function getCurrentStage() external view returns (SettlementStage)
function calculateTimeWeight() external view returns (uint256) // Returns basis points (20000 = 2.0x)
function getContributionCap(address contributor, bool outcome) external view returns (uint256)
function canActivateFromPool(bool outcome) external view returns (bool)
```

---

### State-Changing Functions

#### stake
```solidity
function stake(uint256 amount, bool isYes, uint256 minTokensOut, uint256 deadline)
    external returns (uint256 tokensReceived)
```
- Requires: Market active, token approval, minSlippage ≥ 1%
- Gas: ~140-190k (optimized with early exits and caching)

```typescript
await token.approve(marketAddress, amount);
await market.stake(amount, true, amount.mul(95).div(100), deadline);
```

#### Settlement Functions

**Commit-Reveal Pattern:**
```solidity
// 1. Commit
function commitSettlement(bytes32 commitmentHash) external
function getCommitmentHash(bool assertingTrue, bytes32 secret) view returns (bytes32)

// 2. Reveal (after 10 minutes)
function initiateSettlement(uint256 bond, bool assertingTrue, bytes32 secret)
    external returns (bytes32 requestId)

// 3. Finalize (after UMA challenge period)
function finalizeSettlement() external
```

**Example:**
```typescript
// Commit
const secret = ethers.utils.keccak256(ethers.utils.randomBytes(32));
const hash = await market.getCommitmentHash(true, secret);
await market.commitSettlement(hash);

// Wait 10 minutes...

// Reveal
const bond = await oracle.getMinimumBond(stakeToken);
await token.approve(marketAddress, bond);
await market.initiateSettlement(bond, true, secret);

// Wait 2hr UMA period + 1hr finalization period...

// Finalize
await market.finalizeSettlement();
```

#### Challenge Period Functions

```solidity
// If no settlement within window
function finalizeAsInvalidNoSettlement() external // Anyone can call after activation window

// If first assertion fails
function initiateSecondAssertion(uint256 bond) external returns (bytes32) // Opposite assertion
function finalizeAsInvalid() external // After 7 days with no second assertion
```

#### Bond Pool Functions

```solidity
// Contribute to bond pool (crowdsourced settlement)
function contributeToPool(bool outcome, uint256 amount) external returns (uint256 timeWeight)

// Finalize settlement using pooled bonds
function finalizePoolSettlement(bool outcome) external returns (bytes32 requestId)

// Refund contribution from losing pool
function refundPoolContribution(bool outcome) external returns (uint256 amount)
```

**Example:**
```typescript
// Check bond pool state
const bondPool = await market.getBondPool();
const stage = await market.getCurrentStage();
const weight = await market.calculateTimeWeight();

console.log(`Stage: ${stage}, Weight: ${weight / 100}%`);
console.log(`YES Pool: ${bondPool.yesPool} / ${bondPool.requiredBond}`);

// Contribute to YES pool
const cap = await market.getContributionCap(userAddress, true);
const amount = ethers.utils.parseUnits("100", 6);

if (amount.lte(cap)) {
    await token.approve(marketAddress, amount);
    await market.contributeToPool(true, amount); // Returns time weight
}

// Activate if pool reached threshold
if (await market.canActivateFromPool(true)) {
    await market.finalizePoolSettlement(true);
}

// After settlement, refund from losing pool
if (info.isSettled) {
    const contribution = await market.getUserContribution(userAddress, false);
    if (contribution.amount.gt(0)) {
        await market.refundPoolContribution(false);
    }
}
```

#### Claim & Emergency Functions

```solidity
// Claim rewards after settlement
function claimRewards() external returns (uint256 amount)

// Emergency mode (factory-activated after 30 days)
function activateEmergency() external // Factory only
function emergencyWithdraw() external returns (uint256) // After 7-day wait
```

```typescript
// Claim
if (info.isSettled) await market.claimRewards();

// Emergency (if stuck)
if (emergencyMode) await market.emergencyWithdraw();
```

---

## AMM & Pricing

**Formula:** Constant Product (x*y=k), 0.3% trading fee

**Price Calculation:**
```solidity
YES price = NO pool / (YES pool + NO pool)  // Prices always sum to 10000 bps (100%)
```

**Slippage Protection:**
- Minimum: 1% (enforced, reverts if < 1%)
- Recommended: 5% (warning emitted if < 5%)

**Stake Limits:**
- Min: 1e15 tokens
- Max: 25% of TVL (after 100 token liquidity)

---

## Settlement & Oracle

**UMA Optimistic Oracle V3**: 2-hour challenge period for assertions

**Bond:** `oracle.getMinimumBond(currency)` = `finalFee / burnedBondPercentage` (typically 2x finalFee with 50% burn)

**Settlement Status:**
```typescript
const info = await market.getMarketInfo();
if (info.isSettled) return "SETTLED";
if (!info.settlementRequestId) return "NOT_INITIATED";

const [, , oracleAddr] = await market.getModules();
const [isSettled, outcome] = await oracle.checkSettlement(info.settlementRequestId);
return isSettled ? "READY_TO_FINALIZE" : "CHALLENGE_PERIOD";
```

**Events:**
- Settlement: `SettlementInitiated`, `MarketSettled`
- Bond Pool: `BondRequirementCalculated`, `BondPoolContribution`, `BondPoolStageChanged`, `BondPoolActivated`, `ActivatorFeesDistributed`, `PoolContributionRefunded`

---

## Rewards & Claims

**Calculation:**
- **YES/NO Wins**: `(userTokens / totalWinningTokens) × (losingPool - fees)`
- **INVALID**: `yesStakeAmount + noStakeAmount` (exact refund)

**Fees (from losing side):** Platform 1%, Creator 2%, Activator 0.5%

**Claim:**
```typescript
// Single market
const info = await market.getMarketInfo();
if (info.isSettled) {
    const reward = await market.calculatePotentialReward(userAddress, info.outcome);
    if (reward.gt(0)) await market.claimRewards();
}

// Batch
for (const market of markets) {
    if ((await market.getMarketInfo()).isSettled) {
        await market.claimRewards();
    }
}
```

---

## Module System

**Three Module Types:**
1. **AMM**: Price discovery (ConstantProductAMM - x*y=k, 0.3% fee)
2. **Staking**: Token issuance (StandardStaking - 1:1, 25% max, 0.001 min)
3. **Oracle**: Settlement (UMAOracleModule - 2hr challenge, dynamic bond)

**Query Modules:**
```typescript
const [ammAddr, stakingAddr, oracleAddr] = await market.getModules();
const [name, desc] = await amm.getMetadata();
const disputePeriod = await oracle.getDisputePeriod();
```

---

## Integration Examples

### Trading Interface (React)
```typescript
// Load market data and prices
const info = await market.getMarketInfo();
const [yesPrice, noPrice] = await market.getCurrentPrices();

// Stake with slippage protection
const amount = ethers.utils.parseUnits(stakeAmount, 6);
await token.approve(marketAddress, amount);
await market.stake(amount, isYes, amount.mul(95).div(100), deadline);
```

### Settlement Bot (Commit-Reveal)
```typescript
// Find settleable markets (timestamp check)
const canSettle = (
    now >= info.settlementTime + 7200 && // 2-hour buffer
    !info.isSettled
);

// Commit-reveal settlement
const secret = ethers.utils.keccak256(ethers.utils.randomBytes(32));
await market.commitSettlement(await market.getCommitmentHash(true, secret));
// Wait 10 minutes
await market.initiateSettlement(bond, true, secret);
// Wait 2hr UMA + 1hr finalization
await market.finalizeSettlement();
```

### Bond Pool Settlement Bot
```typescript
// Monitor bond pool opportunities
const info = await market.getMarketInfo();
const bondPool = await market.getBondPool();
const stage = await market.getCurrentStage();

// Check if contribution is viable
if (stage !== 0 && !info.isSettled) {  // NOT_STARTED = 0
    const timeWeight = await market.calculateTimeWeight();
    const cap = await market.getContributionCap(userAddress, true);  // YES pool

    console.log(`Stage: ${stage}, Weight: ${timeWeight / 100}%, Cap: ${cap}`);

    // Contribute if early enough (weight > 1.0x) and within cap
    if (timeWeight > 10000 && cap.gt(0)) {
        const amount = ethers.utils.parseUnits("50", 6);
        if (amount.lte(cap)) {
            await token.approve(marketAddress, amount);
            const weight = await market.contributeToPool(true, amount);
            console.log(`Contributed ${amount} with ${weight / 100}% weight`);
        }
    }

    // Activate pool if threshold reached
    if (await market.canActivateFromPool(true) && !bondPool.yesActivated) {
        await market.finalizePoolSettlement(true);
        console.log("Activated YES pool settlement");
    }
}

// After settlement, claim refunds from losing pools
if (info.isSettled) {
    const yesContribution = await market.getUserContribution(userAddress, true);
    const noContribution = await market.getUserContribution(userAddress, false);

    // Refund losing pool (if you contributed to both sides)
    if (info.outcome === 1 && noContribution.amount.gt(0)) {  // YES won, refund NO
        await market.refundPoolContribution(false);
    } else if (info.outcome === 2 && yesContribution.amount.gt(0)) {  // NO won, refund YES
        await market.refundPoolContribution(true);
    }
}
```

### Portfolio Tracker
```typescript
// Iterate markets, check positions
for (const marketAddr of allMarkets) {
    const position = await market.getUserPosition(userAddress);
    if (position.yesTokens.gt(0) || position.noTokens.gt(0)) {
        const reward = await market.calculatePotentialReward(userAddress, info.outcome);
        // Track active positions and claimable rewards
    }
}
```

---

## UI/UX Best Practices

**Market Display:** YES/NO statements, prices (%), volume, deadline, participants, fees

**Staking Flow:** Select side → Enter amount → Show price impact/slippage → Approve → Stake

**Position Display:** Tokens held, original stake, current value, P&L, potential outcomes

**Settlement States:** Staking active → Settlement ready → Initiated (challenge) → Settled → Claimed

**Error Formatting:**
```typescript
function formatError(error) {
    if (error.includes("StakingClosed")) return "Staking period ended";
    if (error.includes("SlippageExceeded")) return "Price moved too much, increase slippage";
    if (error.includes("InvalidAmount")) return "Stake must be 0.001-25% of market";
    return "Transaction failed. Please retry.";
}
```

---

## Error Handling

**Common Errors:**
- `StakingClosed`: Check `isActive()` before stakes
- `SlippageExceeded`: Increase slippage or reduce amount
- `InvalidAmount`: Must be ≥0.001 and ≤25% of market
- `TransferFailed`: Check approval and balance

**Bond Pool Errors:**
- `TooManyContributors`: Pool has reached 1000 contributors
- `PoolNotLost`: Cannot refund from winning or non-activated pool
- `NoContribution`: User has no contribution to refund
- `ContributionCapExceeded`: Amount exceeds user's contribution cap
- `NotParticipant`: Must have staked to contribute in Stage 1
- `BondPoolNotReached`: Pool hasn't reached required bond threshold
- `BondPoolAlreadyActivated`: Pool has already been activated
- `PlatformCannotContribute`: Platform treasury cannot contribute to pools

**Safe Stake Pattern:**
```typescript
// 1. Validate: isActive(), amount within limits
// 2. Calculate: minTokensOut = amount × (1 - slippage)
// 3. Approve: Check allowance, approve if needed
// 4. Stake: Set 5-10min deadline
// 5. Handle: Catch and format errors for users
```

---

## Security Considerations

**Best Practices:**
1. **Approvals**: Check allowance first, approve exact amounts (not unlimited)
2. **Deadlines**: Use 5-10min transaction deadlines
3. **Slippage**: Set 1-5% minimum (enforced: ≥1%, recommended: ≥5%)
4. **Validation**: Verify market active, amount > 0.001, balance sufficient
5. **Contract Verification**: Ensure market created by factory, isInitialized = true
6. **MEV Protection**: Use private RPC for large stakes, tighter slippage (1%) for >10k USDC

**Constants & Optimizations:**
- `MAX_CONTRIBUTORS_PER_POOL`: 1000 (prevents unbounded array DoS)
- `STAGE_1_DURATION`: 24 hours (participant-only period)
- `STAGE_2_DURATION`: 24 hours (public contribution period)
- Time Weights: Stage 1 (2.0x→1.0x), Stage 2 (1.0x→0.5x)
- **Platform Treasury Early Exit**: Saves ~5,000 gas on rejected contribution attempts
- **Cached Bond Requirement**: Calculated once, saves ~3,000 gas per subsequent contribution
- **Contributor Limit Check**: +4,500 gas overhead per contribution (prevents DoS attack)

---

## Additional Resources

- **Market Contract**: `src/PredictionMarket.sol`
- **Market Interface**: `src/interfaces/IPredictionMarket.sol`
- **Module Interfaces**: `src/interfaces/modules/`
- **UMA Documentation**: https://docs.uma.xyz
- **Factory Integration Guide**: `doc/factory-integration-guide.md`

---

## Support

For questions, issues, or feature requests:
- GitHub Issues: [Repository URL]
- Documentation: [Docs URL]
- Discord: [Community URL]
