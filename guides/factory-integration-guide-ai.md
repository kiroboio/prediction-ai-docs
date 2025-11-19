# PredictionMarketFactory Integration Guide (AI-Optimized)

> **Target Audience:** AI agents with access to both PredictionMarketFactory and PredictionMarket ABIs
> **Assumption:** You have complete ABI knowledge - function signatures, structs, events, and errors
> **Focus:** Business logic, edge cases, and concepts NOT obvious from ABIs alone

---

## Table of Contents

1. [Overview & Key Concepts](#overview--key-concepts)
2. [Multi-Tier Creator Fee System](#multi-tier-creator-fee-system) ⭐
3. [Settlement Protection Mechanisms](#settlement-protection-mechanisms) ⭐
4. [Market Creation Flow](#market-creation-flow)
5. [Timing Constraints](#timing-constraints)
6. [Network Configurations](#network-configurations)
7. [Best Practices & Gotchas](#best-practices--gotchas)
8. [Cross-Reference: User Participation](#cross-reference-user-participation)

---

## Overview & Key Concepts

**Factory Role:** Deploy markets via EIP-1167 clones (~90% gas savings)

### Key Features NOT Obvious from ABI

1. **Automatic Statement Generation**
   - You provide single `statement` parameter
   - Factory generates TRUE and FALSE variants automatically
   - Format: `"As of timestamp {settlementTime} according to {dataSource}, the following is {TRUE/FALSE}: {statement}"`

2. **Two-Layer Fee Control**
   - **Layer 1 (Platform Defaults)**: Set by owner via 48h timelock (`proposeSetDefaultFees`)
   - **Layer 2 (Creator Splits)**: Per-market customization of 1.8% creator fee distribution

3. **Rate Limiting**
   - 1-minute cooldown between markets per creator
   - 100 markets maximum per 24 hours per creator
   - Applied to `msg.sender`, not `params.marketCreator`

4. **Module Validation Requirements**
   - Modules must exist in ModuleRegistry
   - Modules must have ACTIVE status (not just registered)
   - Modules must be compatible with each other
   - Correct module types (AMM, Staking, Oracle)

### When to Use Factory vs Market Contract

- **Factory**: Market creation, discovery, batch operations, creator management
- **Market Contract**: User staking, settlement, claims → See [market-dapp-guide-ai.md](market-dapp-guide-ai.md)

---

## Multi-Tier Creator Fee System

### Overview

The **1.8% creator fee** (180 basis points) can be split between two parties:
- **Market Creator** (`marketCreator`): Person who conceptualized the market
- **Platform Operator** (`platformOperator`): UI/platform provider who facilitated creation

### Three Operating Modes

#### Mode 1: Independent Creator (Default)

```solidity
params.platformOperator = address(0);
params.creatorFeeToCreatorBps = 0;      // Defaults to 180
params.creatorFeeToPlatformBps = 0;     // Defaults to 0
```

**Result:**
- Creator receives: 1.8% (180 bps)
- Platform receives: 0%
- **Total market fee: 5.0%** (1.2% platform + 1.8% creator + 2.0% activator)

---

#### Mode 2: Fee Split (Custom Distribution)

```solidity
params.platformOperator = PLATFORM_ADDRESS;
params.creatorFeeToCreatorBps = 90;     // 0.9% to creator
params.creatorFeeToPlatformBps = 90;    // 0.9% to platform
```

**Result:**
- Creator receives: 0.9% (90 bps)
- Platform receives: 0.9% (90 bps)
- **Total market fee: 5.0%** (1.2% platform + 1.8% creator split + 2.0% activator)

**Supported Splits:**
- `180+0`: 100% to creator
- `0+180`: 100% to platform operator
- `90+90`: 50/50 split
- `126+54`: 70/30 platform-heavy
- `54+126`: 70/30 creator-heavy
- Any custom split that sums to exactly **180**

---

#### Mode 3: External Payment (0+0) ⚠️ SPECIAL CASE

```solidity
params.platformOperator = EXTERNAL_CONTRACT_ADDRESS;
params.creatorFeeToCreatorBps = 0;
params.creatorFeeToPlatformBps = 0;
```

**Result:**
- Creator fee: **ELIMINATED** (0%)
- **Total market fee: 3.2%** (1.2% platform + 0% creator + 2.0% activator)
- 1.8% discount vs standard markets (5.0% → 3.2%)

**Use Case:**
- External contract pays creator/platform via alternative mechanism
- Subscription models, different tokens, off-chain payments
- Users benefit from lower fees
- **Warning:** No on-chain enforcement of external payments

---

### Validation Rules

1. **Sum Requirement:**
   - `creatorFeeToCreatorBps + creatorFeeToPlatformBps` must equal **180** OR both be **0**
   - Reverts if sum is anything other than 0 or 180

2. **Platform Operator Requirement:**
   - If `creatorFeeToPlatformBps > 0`, then `platformOperator != address(0)`
   - Cannot allocate fees to non-existent platform

3. **External Payment Mode:**
   - When both fees are 0 with `platformOperator != address(0)`:
     - `totalCreatorFeeBps` set to 0
     - `totalFeeBps` recalculated as `platformFeeBps + activatorFeeBps` (320 instead of 500)

4. **Platform-Controlled Fees NOT Customizable:**
   - Platform fee (1.2%) cannot be changed per market
   - Activator fee (2.0%) cannot be changed per market
   - Only the creator fee split (1.8%) is customizable

### Fee Claiming

Both `marketCreator` and `platformOperator` call the same function:
```solidity
factory.claimAllCreatorFees(address);  // Works for BOTH roles
```

Factory automatically distributes accumulated fees to both parties based on their configured splits.

---

## Settlement Protection Mechanisms

Two independent mechanisms prevent invalid/nonsensical statements from settling incorrectly:

### Mechanism 1: Settlement Activation Window

**Problem:** Market reaches settlement time but no one initiates settlement (statement may be unresolvable/nonsensical)

**Solution:**
- Configurable window: 24 hours to 7 days
- Default: 48 hours (set via `settlementActivationWindow = 0`)
- If no settlement initiated within window → Anyone can call `finalizeAsInvalidNoSettlement()`

**Parameters:**
```solidity
params.settlementActivationWindow = 0;           // Use default 48 hours
params.settlementActivationWindow = 86400;       // 24 hours
params.settlementActivationWindow = 604800;      // 7 days
```

**Result:** Market marked INVALID, proportional refunds to all participants

**When to Use:**
- Simple/obvious markets: 24 hours (participants will quickly settle)
- Complex markets: 7 days (need more time to verify)

---

### Mechanism 2: Challenge Period (7 Days)

**Problem:** First settlement assertion is disputed and fails via UMA oracle

**Solution:**
- 7-day challenge period begins automatically after failed assertion
- Anyone can make second assertion with bond during this period
- If no second assertion made within 7 days → Anyone can call `finalizeAsInvalid()`

**Economic Incentive:**
- If statement is **valid/resolvable**: Someone will make second assertion for "easy money" (correct assertion = bond returned + fees)
- If statement is **invalid/nonsensical**: No one will risk their bond on unresolvable question

**Result:** Market marked INVALID, full refunds to all participants

**Key Difference from Mechanism 1:**
- Mechanism 1: **Before any assertions** (settlement window timeout)
- Mechanism 2: **After first assertion fails** (challenge period for second attempt)

---

### Market Outcomes

| Outcome | Value | Description | Payout |
|---------|-------|-------------|--------|
| PENDING | 0 | Not yet settled | None |
| YES | 1 | Statement was TRUE | YES stakers win |
| NO | 2 | Statement was FALSE | NO stakers win |
| INVALID | 3 | Nonsensical/unverifiable | Proportional refunds |

---

## Market Creation Flow

### 1. Statement Auto-Generation

**Input (you provide):**
```solidity
params.statement = "Bitcoin price was above $100,000";
params.dataSource = "CoinGecko BTC/USD price feed";
params.settlementTime = 1735689600;  // Dec 31, 2025
```

**Output (factory generates):**
```
TRUE:  "As of timestamp 1735689600 according to CoinGecko BTC/USD price feed,
        the following is TRUE: Bitcoin price was above $100,000"

FALSE: "As of timestamp 1735689600 according to CoinGecko BTC/USD price feed,
        the following is FALSE: Bitcoin price was above $100,000"
```

**Benefit:** Consistent formatting, reduces errors from manually crafting both statements

---

### 2. Rate Limiting Behavior

**Applies to:** `msg.sender` (caller of `createMarket`)

**NOT applied to:** `params.marketCreator`

**Implications:**
- External contracts creating markets on behalf of users have separate rate limits
- Each external contract can create up to 100 markets/day
- Potential bypass: Deploy multiple contracts (partially mitigated by `creationFee`)

**Limits:**
- **Cooldown**: 1 minute between markets
- **Daily Cap**: 100 markets per 24 hours
- **Tracking**: Uses gas-optimized packed struct (uint64 timestamps, uint32 counter)

---

### 3. Module Validation

Modules undergo strict validation:

1. **Existence Check:**
   ```solidity
   require(moduleRegistry.moduleExists(params.ammModuleId), "Module not found");
   ```

2. **Status Check:**
   ```solidity
   require(moduleRegistry.isModuleActive(params.ammModuleId), "Module not active");
   ```

3. **Type Check:**
   ```solidity
   require(moduleRegistry.getModuleType(params.ammModuleId) == ModuleType.AMM, "Wrong type");
   ```

4. **Compatibility Check:**
   ```solidity
   require(moduleRegistry.areCompatible(ammId, stakingId, oracleId), "Incompatible");
   ```

**Security Note:** Module registry is add-only - modules cannot be replaced after registration. Ensures market integrity.

---

## Timing Constraints

### Critical Relationships

```
block.timestamp < stakingDeadline < settlementTime < settlementTime + activationWindow
                   ↑                ↑
                   ≥ 1 hour         ≥ 30 minutes apart
```

**Validation Rules:**
```solidity
// Minimum staking period: 1 hour
require(params.stakingDeadline > block.timestamp + MIN_STAKING_PERIOD);

// Minimum settlement delay: 30 minutes after staking ends
require(params.settlementTime >= params.stakingDeadline + MIN_SETTLEMENT_DELAY);

// Settlement activation window: 24h - 7d (0 = use default 48h)
if (params.settlementActivationWindow != 0) {
    require(params.settlementActivationWindow >= 24 hours);
    require(params.settlementActivationWindow <= 7 days);
}
```

### Timestamp Buffer (Multi-Chain Safety)

**Problem:** Different chains have different block times and timestamp manipulation risks

**Solution:**
- All timing checks use `block.timestamp` with **2-hour TIMESTAMP_BUFFER**
- Works across chains:
  - Ethereum: 12s blocks
  - Base: 2s blocks
  - Arbitrum: ~0.25s blocks

**Protection:**
- Validators enforce strict timestamp rules (must be within reasonable range)
- 2-hour buffer provides ample protection against minor manipulation
- No reliance on `block.number` (varies too much across chains)

### Settlement Timing Phases

1. **Staking Period:** Until `stakingDeadline + TIMESTAMP_BUFFER (2h)`
2. **Settlement Eligibility:** After `settlementTime + TIMESTAMP_BUFFER (2h)`
3. **Activation Window:** `settlementActivationWindow` duration (default 48h)
4. **Auto-INVALID:** If no settlement after activation window expires

---

## Network Configurations

### Verified Addresses

All addresses verified from official sources:
- **UMA**: [UMA Protocol GitHub](https://github.com/UMAprotocol/protocol)
- **USDC**: [Circle Developer Docs](https://developers.circle.com/stablecoins/usdc-contract-addresses)

| Network | Chain ID | USDC Address | UMA OOv3 Address | UMA Finder Address |
|---------|----------|--------------|------------------|-------------------|
| **Ethereum Mainnet** | 1 | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | `0xfb55F43fB9F48F63f9269DB7Dde3BbBe1ebDC0dE` | `0x40f941E48A552bF496B154Af6bf55725f18D77c3` |
| **Base Mainnet** | 8453 | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | `0x2aBf1Bd76655de80eDb3086114315Eec75AF500c` | `0x7E6d9618Ba8a87421609352d6e711958A97e2512` |
| **Sepolia Testnet** | 11155111 | `0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8` | `0xFd9e2642a170aDD10F53Ee14a93FcF2F31924944` | `0xf4C48eDAd256326086AEfbd1A53e1896815F8f13` |
| **Base Sepolia** | 84532 | `0x036CbD53842c5426634e7929541eC2318f3dCF7e` | `0x0F7fC5E6482f096380db6158f978167b57388deE` | `0xfF4Ec014E3CBE8f64a95bb022F1623C6e456F7dB` |

### Default Fee Structure

**Mainnet Networks (Ethereum, Base):**
- Platform Fee: 1.2% (120 bps) → factory treasury
- Creator Fee: 1.8% (180 bps) → creator/platform split (customizable per market)
- Activator Fee: 2.0% (200 bps) → bond pool contributors
- **Total: 5.0%** (or 3.2% with 0+0 mode)

**Testnet Networks (Sepolia, Base Sepolia):**
- Platform Fee: 0.5% (50 bps)
- Creator Fee: 1.8% (180 bps)
- Activator Fee: 2.0% (200 bps)
- **Total: 4.3%** (or 2.5% with 0+0 mode)

**Fee Control:**
- Only factory owner can change platform/activator fees
- Requires 48-hour timelock for security
- Creator fee split is customizable per market (no timelock)

---

## Best Practices & Gotchas

### Market Creation Best Practices

1. **Clear Statements:**
   - Avoid subjective language ("best", "better", "significant")
   - Use objective, measurable criteria ("price > $100,000", "won championship")
   - Include exact timestamp in statement for clarity

2. **Reasonable Staking Periods:**
   - Minimum: 7 days recommended (1 hour absolute minimum)
   - Longer periods = more time for market discovery
   - Consider event timeline (don't end staking before event starts)

3. **Settlement Timing:**
   - Set `settlementTime` AFTER event resolution, not before
   - Example: Sports game ends 8 PM → set settlement for next day
   - Allows time for official results confirmation

4. **Token Selection:**
   - Use widely accepted tokens (USDC recommended)
   - Verify token address for network
   - Check token has sufficient liquidity

5. **Module Selection:**
   - Standard modules for most use cases:
     - AMM: `keccak256("CONSTANT_PRODUCT_AMM_V1")`
     - Staking: `keccak256("STANDARD_STAKING_V1")`
     - Oracle: `keccak256("UMA_ORACLE_V3_MODULE")`
   - Verify modules are active: `isModuleActive(moduleId)`

### Fee Configuration Best Practices

1. **Keep Total Fees ≤5%:**
   - Higher fees reduce user participation
   - Standard 5.0% is competitive with other prediction markets
   - 0+0 mode (3.2%) attracts users but requires external payment mechanism

2. **Understand Fee Control:**
   - Platform fee: Controlled by owner (48h timelock)
   - Creator fee total: Controlled by owner (48h timelock)
   - Creator fee split: Customizable per market (no restrictions)

3. **0+0 Mode Caution:**
   - Only use if you have reliable external payment mechanism
   - No on-chain enforcement of payments
   - Risk: Creator/platform may not receive payment
   - Benefit: 1.8% lower fees attracts users

### Gas Optimization

1. **Batch Operations:**
   ```solidity
   factory.batchCreateMarkets(paramsArray);  // Max 10 markets
   ```

2. **Paginated Queries:**
   ```solidity
   factory.getActiveMarketsPaginated(offset, limit);  // Large datasets
   ```

3. **Module ID Caching:**
   ```solidity
   bytes32 ammId = keccak256("CONSTANT_PRODUCT_AMM_V1");  // Cache, don't recalculate
   ```

### Security Considerations

1. **Input Validation:**
   - Validate all timestamps are future dates
   - Verify module IDs exist before calling factory
   - Check token addresses are valid contracts

2. **Error Handling:**
   ```solidity
   try factory.createMarket(params) returns (address market) {
       // Success
   } catch (bytes memory reason) {
       // Rate limit hit, invalid params, etc.
   }
   ```

3. **Module Verification:**
   ```solidity
   require(factory.isModuleActive(moduleId), "Module not active");
   ```

### Common Gotchas

1. **Zero Values Mean "Use Default":**
   - `settlementActivationWindow = 0` → Use default 48 hours (NOT "no window")
   - `marketCreator = address(0)` → Use `msg.sender` (NOT "no creator")
   - `creatorFeeToCreatorBps = 0` → Use default split (NOT "zero fee")

2. **Tag Search Uses OR Logic:**
   ```solidity
   getMarketsByTagsPaginated(["bitcoin", "ethereum"], 0, 10);
   // Returns markets matching bitcoin OR ethereum (not AND)
   ```

3. **Fee Claiming Works for Both Roles:**
   ```solidity
   // Same function for creator and platform operator:
   factory.claimAllCreatorFees(creatorAddress);      // Creator claims
   factory.claimAllCreatorFees(platformOperator);    // Platform claims
   ```

4. **Rate Limits Apply to Caller, Not Creator:**
   - Limit tracked by `msg.sender`
   - External contracts can bypass by deploying multiple instances
   - Partially mitigated by `creationFee` requirement

5. **Module Registry is Add-Only:**
   - Modules cannot be replaced after registration
   - Deprecated modules marked INACTIVE, not removed
   - Ensures existing markets remain functional

6. **Settlement Time ≠ Staking Deadline:**
   - `stakingDeadline`: When users can no longer stake
   - `settlementTime`: Earliest time settlement can begin
   - Minimum gap: 30 minutes (prevents race conditions)

---

## Cross-Reference: User Participation

This guide covers **market creation** via the factory. For **market participation** (user-facing flows), see:

→ **[market-dapp-guide-ai.md](market-dapp-guide-ai.md)**

**Covers:**
- Staking flow with slippage protection
- Bond pool settlement (crowdsourced bonds with time-weighted rewards)
- Settlement activation and oracle integration
- Reward claims and fee distribution
- Emergency withdrawals and edge cases
- AMM pricing and position tracking

---

## Document Version

- **Version:** 1.0
- **Last Updated:** November 19, 2025
- **Target:** AI agents with ABI access
- **Full Guide:** [factory-integration-guide.md](factory-integration-guide.md) (for human developers)
