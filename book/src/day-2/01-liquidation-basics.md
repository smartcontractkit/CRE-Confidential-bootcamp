# Liquidation and How to Prevent It

> No more CRE or Confidential Workflow fundamentals today — we jump straight into the new case study. But because it involves quite a few DeFi finance concepts, let's spend 15 minutes getting them clear first.

## Overcollateralized Lending: The Foundation of DeFi Borrowing

In DeFi lending protocols like Aave and Compound, borrowing works through **overcollateralization**:

```
You deposit $10,000 worth of ETH as collateral
        │
        ▼
The protocol lets you borrow up to a certain ratio (say, $7,000 USDC)
        │
        ▼
Your collateral must always "sufficiently cover" your debt
```

Why overcollateralization? Because the protocol has no identity information about you and no way to chase you for repayment — **the collateral is the only guarantee**.

## Three Key Metrics

### 1. LTV (Loan-to-Value)

```
LTV = debt value / collateral value
```

Example: deposit $10,000 of ETH, borrow $7,000 USDC → LTV = 70%.

Each collateral asset has a **maximum LTV** (say, 75%) that determines how much you can borrow at most.

### 2. Liquidation Threshold

The liquidation threshold is a line slightly above the max LTV (say, 78%). **When your LTV crosses the liquidation threshold, the position is deemed undercollateralized, and anyone can liquidate it.**

### 3. Health Factor (HF) ⭐

This is the most commonly used risk metric:

```
                     collateral value × liquidation threshold
Health Factor (HF) = ────────────────────────────────────────
                               debt value
```

| HF value | Position status |
|----------|----------------|
| **HF > 1** | Safe, sufficiently collateralized |
| **HF = 1** | The liquidation line! Can be liquidated on arrival |
| **HF < 1** | Undercollateralized, can be liquidated |

Example: $10,000 of ETH collateral (liquidation threshold 78%), $7,000 debt → HF = 10000 × 0.78 / 7000 ≈ **1.11**.

> ⚠️ **HF moves with prices.** ETH price drops → collateral value shrinks → HF falls → danger when it approaches 1.0. This is what drives "liquidation cascades" during periods of high crypto market volatility.

## What Liquidation Costs You

When HF < 1, a **liquidator** can:

1. Repay part of your debt on your behalf (say, 50%)
2. Seize collateral worth the repaid amount **plus a bonus** (the liquidation bonus, typically 5%–10%) at a discount

For the borrower, liquidation means:

- 💸 **Liquidation penalty**: the collateral seized is worth more than the debt repaid
- 📉 **Forced selling at the bottom**: your collateral is sold during a market crash — precisely the worst price
- 🔒 **Loss of the position**: if the market rebounds afterward, you no longer have collateral to benefit

## How to Prevent Liquidation

The core idea is one sentence: **raise your HF before it gets close to 1**. There are two broad approaches:

### Approach 1: Increase collateral (grow the numerator)

| Action | Description |
|--------|-------------|
| `add_collateral` | Add collateral directly using stablecoin reserves |
| `bridge_and_add_collateral` | Bridge assets from another chain, then add |
| `swap_reserve_to_collateral` | Swap reserves into the collateral asset, then deposit |

### Approach 2: Reduce debt (shrink the denominator)

| Action | Description |
|--------|-------------|
| `repay_with_reserves` | Repay part of the debt directly with reserves |
| `swap_reserve_to_borrowed_and_repay` | Swap reserves into the borrowed asset, then repay |
| `partial_debt_repayment` | Repay a percentage (say, 18%) of the debt |
| `full_debt_repayment` | Repay in full, eliminating the risk entirely |

### Manual vs. Automated

The problem with manual defense: **liquidations often happen at 3 AM, within minutes**. When ETH crashes, going from HF 1.15 to liquidated can take just minutes — far too fast for a human to react.

So you need automation — a system that monitors risk signals 24/7 and executes defensive actions as danger approaches. That's exactly where CRE shines.

## Why the Defense Strategy Needs to Be "Confidential"

Automated liquidation protection has a subtle game-theoretic problem:

> **If your defense strategy is public, it can be exploited.**

- If the market knows "this address adds collateral whenever HF drops below 1.25," attackers can manipulate prices against you, anticipate your moves, and **front-run** them
- If your reserve size and deployable capital caps are public, an adversary can calculate exactly "how much capital it takes to push you past the liquidation line"
- Your exchange credentials and strategy parameters (target HF, deployment caps, sequencing preferences) are all high-value intelligence

So a production-grade automated liquidation protection system needs:

1. **Automation**: 24/7 monitoring + automatic execution → CRE Workflow
2. **Confidentiality**: thresholds, strategy, and credentials hidden from node operators → **Confidential Workflow**

## Key Takeaways

| Concept | One-liner |
|---------|-----------|
| **Overcollateralization** | Deposit collateral worth more than what you borrow |
| **LTV** | The debt / collateral ratio |
| **Liquidation threshold** | LTV crossing it → can be liquidated |
| **Health Factor (HF)** | (collateral × liquidation threshold) / debt; < 1 means danger |
| **Liquidation** | A liquidator repays your debt and seizes discounted collateral + a bonus |
| **Defense** | Add collateral (numerator ↑) or repay debt (denominator ↓) |
| **Confidential defense** | Keep the strategy and thresholds secret so they can't be predicted or front-run |

## What's Next

With the concepts clear, let's look at Case Study 2: an **Automated Liquidation Protection** system built with a CRE Confidential Workflow.
