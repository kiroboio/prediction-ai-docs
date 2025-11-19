# PredictionMarket DApp Integration Guide (AI-Optimized)

> **Audience:** AI agents building prediction market integrations
> **Assumption:** You have access to `IPredictionMarket` ABI
> **Companion:** See `factory-integration-guide-ai.md` for market creation
> **Original:** See `market-dapp-guide.md` for full documentation

---

## Purpose of This Guide

This guide covers **business logic and edge cases NOT obvious from the ABI alone**. If a concept can be inferred from function signatures, it's excluded.

### What You'll Learn
1. **Market Lifecycle** - Phase transitions and timing rules
2. **Dual Settlement System** - Commit-reveal vs Bond pools (when to use each)
3. **Bond Pool Mechanics** - Two-stage system, time weights, contribution caps
4. **Fee Distribution** - Multi-tier creator fee split (see factory guide for details)
5. **Edge Cases** - INVALID outcomes, emergency mode, challenge periods
6. **Integration Gotchas** - Slippage enforcement, timing buffers, contribution limits

---

## Market Lifecycle (Timing & State Transitions)

### Phase 1: Staking (Duration: Until `stakingDeadline + 2hr`)

**Timestamp Buffer Protection:**
- All timing checks use `block.timestamp` with **2-hour `TIMESTAMP_BUFFER`**
- Multi-chain compatible (Ethereum 12s blocks, Base 2s blocks, Arbitrum 0.25s)
- Example: `stakingDeadline = 1000000` → actually closes at `1007200`

**Key Timing Check:**
```solidity
// isActive() returns:
block.timestamp <= stakingDeadline + TIMESTAMP_BUFFER
```

**Actions:**
- Users call `stake(amount, isYes, minTokensOut, deadline)`
- AMM adjusts prices using constant product formula (x*y=k)
- Virtual YES/NO tokens issued to stakers

---

### Phase 2: Settlement (Dual System)

Settlement can occur via **two independent methods**:

#### Method 1: Commit-Reveal Settlement

**Best For:** Single activators, lower gas markets, faster settlement

**Timeline:**
```
settlementTime + 2hr → Activation window opens (24h-7d configurable)
                     → Anyone can commit settlement
10 minutes later    → Can reveal commitment
                     → Creates UMA assertion (2hr challenge)
After UMA resolves  → Anyone can finalize after 1hr
                     → Activator can finalize immediately
```

**Commit-Reveal Protection:**
- **Why:** Prevents front-running of settlement assertions
- **How:** Commitment = `keccak256(abi.encodePacked(assertingTrue, secret, msg.sender))`
- **Minimum Wait:** 10 minutes between commit and reveal (enforced)

**Example Flow:**
```typescript
// 1. Commit
const secret = ethers.utils.keccak256(ethers.utils.randomBytes(32));
const hash = await market.getCommitmentHash(true, secret);
await market.commitSettlement(hash);

// 2. Wait 10 minutes...

// 3. Reveal
const bond = await oracle.getMinimumBond(stakeToken);
await token.approve(marketAddress, bond);
await market.initiateSettlement(bond, true, secret);

// 4. Wait UMA challenge period (2hr) + finalization period (1hr)...

// 5. Finalize
await market.finalizeSettlement();
```

**If First Assertion Fails:**
- **Challenge Period:** 7 days
- **Options:**
  - Someone calls `initiateSecondAssertion(bond)` → opposite outcome
  - No one asserts → anyone calls `finalizeAsInvalid()` → all refunds

---

#### Method 2: Bond Pool Settlement (Crowdsourced)

**Best For:** High-bond markets, democratic settlement, risk sharing

**Two-Stage Contribution Period:**

**Stage 1: Participants Only (0-24 hours after settlement time)**
- **Who:** Only users who staked in the market
- **Time Weight:** Linearly decays **2.0x → 1.0x** (20000 → 10000 bps)
- **Why:** Reward early participants, prevent free-riding

**Stage 2: Public (24-48 hours after settlement time)**
- **Who:** Anyone (including non-participants)
- **Time Weight:** Linearly decays **1.0x → 0.5x** (10000 → 5000 bps)
- **Why:** Ensure activation if participants don't act

**Contribution Caps (Per Pool):**
```
General User:     30% of required bond
Market Creator:   10% of required bond
Platform Treasury: 0% (blocked entirely)
```

**Time Weight Calculation:**
```solidity
// Stage 1 (0-24h): Linear decay from 2.0x to 1.0x
if (stage == PARTICIPANTS_ONLY) {
    elapsed = block.timestamp - stageStartTime;
    timeWeight = 20000 - ((elapsed * 10000) / STAGE_1_DURATION);
    // At 0h: 20000 (2.0x)
    // At 12h: 15000 (1.5x)
    // At 24h: 10000 (1.0x)
}

// Stage 2 (24-48h): Linear decay from 1.0x to 0.5x
if (stage == PUBLIC) {
    elapsed = block.timestamp - stageStartTime;
    timeWeight = 10000 - ((elapsed * 5000) / STAGE_2_DURATION);
    // At 24h: 10000 (1.0x)
    // At 36h: 7500 (0.75x)
    // At 48h: 5000 (0.5x)
}
```

**Reward Distribution (Winning Pool Only):**
```
Your Shares = contribution amount × time weight
Your Reward = (your shares / total pool shares) × activator fees

Example:
- You contribute $100 at 1.5x weight = 150 shares
- Pool has 1000 total shares
- Activator fees = $200
- Your reward = (150 / 1000) × $200 = $30
```

**Key Functions:**
```solidity
// Check contribution cap (dynamic based on creator/platform/general)
uint256 cap = market.getContributionCap(userAddress, true); // true = YES

// Contribute to pool (returns time weight)
uint256 weight = market.contributeToPool(true, amount);

// Check if pool reached threshold
bool canActivate = market.canActivateFromPool(true);

// Activate settlement (anyone can call once threshold reached)
bytes32 requestId = market.finalizePoolSettlement(true);

// Refund from losing pool (after settlement)
uint256 refunded = market.refundPoolContribution(false);
```

**Critical Integration Notes:**

1. **Contribution Limit Enforcement:**
   - `getContributionCap()` returns dynamic cap based on:
     - User's role (creator vs general vs platform)
     - Current pool state
     - Already contributed amount
   - Always check cap before contributing

2. **Activation Race Condition:**
   - First call to `finalizePoolSettlement()` wins
   - Subsequent calls revert with `BondPoolAlreadyActivated`
   - Check `bondPool.yesActivated` or `bondPool.noActivated` before calling

3. **Refund Timing:**
   - Can only refund from **losing or non-activated** pools
   - Must wait until market is settled
   - Winning pool: bonds returned automatically, share activator fees
   - Losing pool: call `refundPoolContribution()` to get contribution back

4. **Stage Transitions:**
   - Stage 1 → Stage 2: Automatic at 24h mark
   - Stage 2 ends: At 48h (or configured `settlementActivationWindow`)
   - After window: Anyone can call `finalizeAsInvalidNoSettlement()`

---

### Phase 3: Claiming Rewards

**Outcome-Specific Claim Logic:**

**YES Wins:**
```
userReward = (userYesTokens / totalYesTokens) × (totalNoStake - fees)
```

**NO Wins:**
```
userReward = (userNoTokens / totalNoTokens) × (totalYesStake - fees)
```

**INVALID:**
```
userRefund = yesStakeAmount + noStakeAmount (EXACT original stake)
```

**Fee Deduction (from losing side only):**
```
Platform Fee:  1.2% (120 bps) → factory treasury
Creator Fee:   1.8% (180 bps) → split between creator and platform operator
Activator Fee: 2.0% (200 bps) → activator (or bond pool contributors)
Total:         5.0% (500 bps)

Note: If creator uses 0+0 split (external payment), total fees = 3.2% (no creator fee)
```

**Multi-Tier Creator Fee Distribution:**
- See `factory-integration-guide-ai.md` for full explanation
- Either party can call `claimCreatorFees()` to trigger distribution
- Contract splits automatically based on `creatorFeeToCreatorBps` and `creatorFeeToPlatformBps`

**Emergency Withdrawal (If Market Stuck):**
```
Timeline:
settlementTime + 30 days → Factory can activate emergency mode
+7 days wait              → Users can call emergencyWithdraw()
                         → Proportional refund: (userTokens / totalTokens) × totalPool
```

---

## Slippage & AMM Rules

### Slippage Enforcement

**Enforced Minimum:** 1% (reverts if `minTokensOut` implies < 1%)

**Recommended Minimum:** 5% (emits warning event if < 5%)

**Why Enforced:**
- Protects users from sandwich attacks
- Prevents accidental accepts of bad prices
- MEV protection in volatile markets

**Calculation:**
```typescript
// For 5% slippage:
const minTokensOut = expectedTokens.mul(95).div(100);

// Validation (done in contract):
const impliedSlippage = ((amount - minTokensOut) * 10000) / amount;
require(impliedSlippage >= 100, "SlippageTooLow"); // 100 bps = 1%
if (impliedSlippage < 500) emit SlippageWarning(); // 500 bps = 5%
```

### Stake Limits

**Minimum Stake:** 1e15 tokens (0.001 USDC)

**Maximum Stake:** 25% of current TVL (after 100 token liquidity)

**Why Maximum:**
- Prevents single-user market manipulation
- Ensures distributed liquidity
- Only enforced after market has 100 tokens (bootstrap period)

---

## Edge Cases & Gotchas

### 1. INVALID Outcome Behavior

**When Market Becomes INVALID:**
1. No settlement within activation window → `finalizeAsInvalidNoSettlement()`
2. First assertion fails + 7 days pass with no second assertion → `finalizeAsInvalid()`
3. Oracle explicitly resolves as INVALID (rare, but possible)

**Claim Behavior:**
```solidity
// INVALID refunds EXACT original stakes (no fees, no rewards)
uint256 refund = position.yesStakeAmount + position.noStakeAmount;

// Even if prices changed dramatically, you get original stake back
// Example: Staked $100 YES, got 50 tokens @ $2.00 price
//          Staked $100 NO, got 150 tokens @ $0.67 price
//          INVALID refund = $200 (not proportional to tokens held)
```

### 2. Multiple Stakes & Position Tracking

**Virtual Token Accounting:**
- Each stake issues virtual YES or NO tokens
- `UserPosition.yesTokens` and `UserPosition.noTokens` track holdings
- `UserPosition.yesStakeAmount` and `UserPosition.noStakeAmount` track original USDC (for INVALID refunds)

**Important:** If you stake multiple times, amounts are cumulative:
```typescript
// First stake: $100 YES @ 0.50 price → 200 YES tokens
// Second stake: $100 YES @ 0.60 price → 166.67 YES tokens
// Position: yesTokens = 366.67, yesStakeAmount = $200
```

### 3. Bond Pool Contributor Limits

**Per-Pool Limit:** 1000 contributors maximum

**Why:** Prevents unbounded array DoS attacks during fee distribution

**Error:** `TooManyContributors` if pool already has 1000 addresses

**Mitigation:** Contribute early or use commit-reveal settlement instead

### 4. Platform Treasury Blocked from Bond Pools

**Rule:** Platform treasury address CANNOT contribute to bond pools (0% cap)

**Why:** Prevents platform from manipulating settlement outcomes

**Error:** `PlatformCannotContribute` if treasury tries to contribute

**Check:**
```solidity
// Early exit in contributeToPool()
if (msg.sender == platformTreasury) revert PlatformCannotContribute();
```

### 5. Settlement Finalization Timing

**Activator Priority:**
```
After UMA resolves:
- First 1 hour: Only activator can finalize (prevents MEV/front-running)
- After 1 hour: Anyone can finalize (ensures completion)
```

**Why:** Activator posted bond and took risk, deserves priority to claim rewards without competition

### 6. Challenge Period Edge Case

**Scenario:** First assertion disputed and fails

**Timeline:**
```
First assertion fails → 7-day challenge period starts
                     → Anyone can call initiateSecondAssertion(bond)
                     → If no one asserts for 7 days
                     → Anyone can call finalizeAsInvalid()
                     → All users get INVALID refunds
```

**Common Mistake:** Assuming market auto-settles after first assertion fails

**Reality:** Requires explicit second assertion OR explicit INVALID finalization

---

## Integration Patterns

### Pattern 1: Staking with Proper Validation

```typescript
// 1. Check market is active
const isActive = await market.isActive();
if (!isActive) throw new Error("Staking closed");

// 2. Check amount limits
const amount = ethers.utils.parseUnits("100", 6);
const tvl = await market.getTotalValueLocked();
const maxStake = tvl.mul(25).div(100); // 25% of TVL
if (amount.gt(maxStake) && tvl.gt(100e6)) throw new Error("Exceeds 25% limit");

// 3. Calculate slippage (5% recommended)
const expectedTokens = await market.calculateStakeReturn(amount, true);
const minTokensOut = expectedTokens.mul(95).div(100);

// 4. Approve and stake
await token.approve(marketAddress, amount);
await market.stake(amount, true, minTokensOut, deadline);
```

### Pattern 2: Bond Pool Contribution

```typescript
// 1. Check stage and time weight
const stage = await market.getCurrentStage();
if (stage === 0) throw new Error("Settlement window not open");

const timeWeight = await market.calculateTimeWeight();
console.log(`Current weight: ${timeWeight / 100}%`);

// 2. Check contribution cap
const cap = await market.getContributionCap(userAddress, true);
if (cap.eq(0)) throw new Error("Cannot contribute (cap = 0)");

// 3. Contribute within cap
const amount = ethers.utils.parseUnits("50", 6);
if (amount.gt(cap)) amount = cap; // Adjust to cap

await token.approve(marketAddress, amount);
const actualWeight = await market.contributeToPool(true, amount);

// 4. Activate if threshold reached
const bondPool = await market.getBondPool();
if (bondPool.yesPool.gte(bondPool.requiredBond) && !bondPool.yesActivated) {
    await market.finalizePoolSettlement(true);
}
```

### Pattern 3: Settlement Monitoring

```typescript
async function monitorSettlement(market) {
    const info = await market.getMarketInfo();

    // Already settled
    if (info.isSettled) return "SETTLED";

    // Not yet initiated
    if (!info.settlementRequestId || info.settlementRequestId === ethers.constants.HashZero) {
        const now = Math.floor(Date.now() / 1000);
        const canSettle = now >= info.settlementTime + 7200; // 2hr buffer
        return canSettle ? "SETTLEABLE" : "STAKING";
    }

    // Check UMA oracle status
    const [, , oracleAddr] = await market.getModules();
    const oracle = new ethers.Contract(oracleAddr, OracleABI, provider);
    const [resolved, outcome] = await oracle.checkSettlement(info.settlementRequestId);

    if (resolved) return "READY_TO_FINALIZE";
    return "CHALLENGE_PERIOD";
}
```

### Pattern 4: Multi-Pool Refund Handling

```typescript
// After settlement, refund losing pool contributions
async function handlePoolRefunds(market, userAddress) {
    const info = await market.getMarketInfo();
    if (!info.isSettled) return;

    const yesContribution = await market.getUserContribution(userAddress, true);
    const noContribution = await market.getUserContribution(userAddress, false);

    // Determine which pool lost
    const losingOutcome = info.outcome === 1 ? false : true; // 1 = YES, 2 = NO

    if (losingOutcome && yesContribution.amount.gt(0)) {
        await market.refundPoolContribution(true);
    } else if (!losingOutcome && noContribution.amount.gt(0)) {
        await market.refundPoolContribution(false);
    }

    // Note: Winning pool gets bonds back automatically + shares activator fees
    // No refund needed for winning pool
}
```

---

## Critical Constants (Gas Optimization Notes)

**Bond Pool Limits:**
```solidity
MAX_CONTRIBUTORS_PER_POOL: 1000
// DoS prevention: +4,500 gas per contribution check
// Alternative: Use commit-reveal settlement for high-contributor scenarios

STAGE_1_DURATION: 24 hours (participants only)
STAGE_2_DURATION: 24 hours (public)

CONTRIBUTION_CAP_GENERAL: 30% of required bond
CONTRIBUTION_CAP_CREATOR: 10% of required bond
CONTRIBUTION_CAP_PLATFORM: 0% (blocked)
```

**Gas Savings:**
- Platform treasury early exit: ~5,000 gas saved on rejected attempts
- Cached bond requirement: ~3,000 gas per contribution after first
- Contributor limit check: +4,500 gas overhead (necessary for security)

---

## Common Errors & Solutions

### Bond Pool Errors

**TooManyContributors**
- Pool has 1000 contributors, cannot add more
- Solution: Use commit-reveal settlement or wait for winning pool to activate

**PoolNotLost**
- Cannot refund from winning or non-activated pool
- Solution: Only refund from losing pool after settlement

**ContributionCapExceeded**
- Amount exceeds your individual cap (30% general, 10% creator, 0% platform)
- Solution: Call `getContributionCap()` first, adjust amount

**NotParticipant**
- Trying to contribute in Stage 1 without staking
- Solution: Wait for Stage 2 (public) or stake in market first

**BondPoolNotReached**
- Trying to activate pool that hasn't reached required bond
- Solution: Check `canActivateFromPool()` first

**PlatformCannotContribute**
- Platform treasury cannot contribute to bond pools
- Solution: Use different address or commit-reveal settlement

### Staking Errors

**StakingClosed**
- `block.timestamp > stakingDeadline + 2hr`
- Solution: Check `isActive()` before staking

**SlippageExceeded**
- Actual tokens < `minTokensOut`
- Solution: Increase slippage tolerance or reduce stake amount

**InvalidAmount**
- Amount < 0.001 or > 25% of TVL (after 100 token liquidity)
- Solution: Adjust amount within limits

---

## Cross-References

**For market creation:**
- See `factory-integration-guide-ai.md` for factory integration
- Covers: Multi-tier fee system, network configs, rate limiting

**For full documentation:**
- See `market-dapp-guide.md` for complete API reference
- Includes: Full struct definitions, all function signatures, UI/UX patterns

**For UMA oracle:**
- UMA Documentation: https://docs.uma.xyz
- OptimisticOracleV3 interface and assertion lifecycle

---

## Key Takeaways

1. **Dual Settlement System:** Commit-reveal (single activator) vs Bond pools (crowdsourced)
2. **Two-Stage Bond Pools:** Participants-only (2.0x→1.0x) then public (1.0x→0.5x)
3. **Contribution Caps:** 30% general, 10% creator, 0% platform (per pool)
4. **INVALID Refunds:** Exact original stake amounts (not proportional to tokens)
5. **Slippage Enforcement:** 1% minimum (enforced), 5% recommended
6. **Timing Buffers:** 2-hour buffer on all timestamp checks (multi-chain safety)
7. **Challenge Periods:** 7 days if first assertion fails (requires second assertion or INVALID)
8. **Emergency Mode:** 30 days + 7 days wait (factory-activated, proportional refunds)

---

**Document Version:** 1.0
**Created:** November 19, 2025
**Status:** AI-Optimized Guide (assumes ABI access)
**Maintained By:** Kirobo Protocol Team
