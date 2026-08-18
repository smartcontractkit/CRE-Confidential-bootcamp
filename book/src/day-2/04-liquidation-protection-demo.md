# Case Study 2: Demo

The flow is the same as yesterday: clone the repo → configure the environment → start the mock server → simulate. If you already cloned `cre-templates` on Day 1, skip straight to Step 2.

## Step 1: Clone the Templates Repo

```bash
git clone https://github.com/smartcontractkit/cre-templates.git
cd cre-templates/starter-templates/confidential-workflows/automated-liquidation-protection
```

## Step 2: Set Up Environment Variables

Create a `.env` at the project root:

```bash
cp .env.example .env
```

`.env.example` is pre-filled with demo defaults — note that it **contains all 9 policy parameters**:

```bash
CRE_ETH_PRIVATE_KEY=

MOCK_PORT=8787
MOCK_EXCHANGE_API_KEY=mock-exchange-key
MOCK_OPENAI_API_KEY=mock-openai-key

# —— Everything below is "policy-as-secrets" ——
MOCK_LIQUIDATION_WARNING_ACTION_THRESHOLD=18          # Liquidation proximity warning line, %
MOCK_LIQUIDATION_MINIMUM_HEALTH_FACTOR=1.25           # Minimum health factor
MOCK_LIQUIDATION_TARGET_HEALTH_FACTOR=1.5             # Target health factor
MOCK_LIQUIDATION_MAX_STABLECOIN_RESERVE_DEPLOYMENT=5000   # Max reserve deployment per execution, $
MOCK_LIQUIDATION_MIN_STABLECOIN_RESERVE_BALANCE=2000      # Minimum reserve balance, $
MOCK_LIQUIDATION_MAX_COLLATERAL_ALLOCATION=80             # Collateral allocation cap, %
MOCK_LIQUIDATION_MAX_PARTIAL_DEBT_REPAYMENT=40            # Partial repayment cap, %
MOCK_LIQUIDATION_DEFENSIVE_ACTION_SEQUENCING_PREFERENCE=collateral-first
MOCK_LIQUIDATION_PREFERRED_VENUES=binance,onchain,coinbase
```

> **Food for thought**: why do these values live in `.env` / secrets rather than in `config.staging.json`? — Because they're your **private strategy**. In a real deployment they're managed by the Vault DON and only ever appear inside the enclave.

## Step 3: Install Dependencies and Start the Mock Server

```bash
cd automated-liquidation-protection-ts
bun install
bun run mock:server
```

You'll see:

```bash
Mock server running at http://127.0.0.1:8787
```

**Keep this terminal running.**

## Step 4: Typecheck and Unit Tests (Optional but Recommended)

Open a **second terminal**:

```bash
cd cre-templates/starter-templates/confidential-workflows/automated-liquidation-protection/automated-liquidation-protection-ts
bun run typecheck
bun run test
```

The tests focus on `computeRiskScore` and `enforcePolicy` — the two **deterministic** functions. This illustrates an important principle: LLM output is hard to test, but the rules that hold the bottom line must be 100% testable.

## Step 5: Simulate the Workflow

Back at the project root:

```bash
cd ..
cre workflow simulate ./automated-liquidation-protection-ts --target=staging-settings
```

You'll see output similar to:

```bash
[SIMULATION] Simulator Initialized

[USER LOG] liquidation-getsecret-ok
[USER LOG] liquidation-defense-executed action_count=2 execution_id=defense_...

Workflow Simulation Result:
 "{\"status\":\"DEFENDED\",\"actionCount\":2,\"riskScore\":50.25,\"executionId\":\"defense_...\"}"

[SIMULATION] Execution finished signal received
```

### Reading the Output Line by Line

| Log line | Meaning |
|----------|---------|
| `liquidation-getsecret-ok` | All 11 secrets (2 credentials + 9 policy parameters) fetched inside the enclave |
| `liquidation-defense-executed` | The defense plan was executed: 2 actions, with an execution ID |

### Why the Default Data Triggers DEFENDED

The mock's built-in position has **already breached the warning line**:

```
Liquidation proximity: 12%  <  warning line 18%   → action required
Health factor:         1.14 <  minimum 1.25        → action required
```

The mock LLM returns this defense decision:

```json
{
  "shouldDefend": true,
  "reasoning": "Liquidation proximity is elevated; add collateral and reduce debt exposure within policy limits.",
  "actions": [
    { "type": "add_collateral",        "amountUsd": 3200, "venue": "onchain" },
    { "type": "partial_debt_repayment", "repayPct": 18,   "venue": "binance" }
  ]
}
```

Then `enforcePolicy` performs its deterministic checks:

- `add_collateral` $3,200 ≤ per-execution cap $5,000 ✅; post-action reserve 10,000 − 3,200 = $6,800 ≥ floor $2,000 ✅
- `partial_debt_repayment` 18% ≤ cap 40% ✅ → converted amount $32,000 × 18% = $5,760
- Ordered per `collateral-first`: `add_collateral` first, then `partial_debt_repayment`

Finally `POST /execute-defense` executes both actions → **DEFENDED**. The risk score of 50.25 matches what we hand-computed in the code walkthrough — verify it yourself.

## Hands-On Experiments (Optional)

### Experiment 1: Make the Position Safe → Trigger SAFE

Edit `mock-server.js` and change `liquidation_proximity_pct` from `12` to `30` (proximity 30% > warning line 18%). Restart the mock server and simulate again:

```bash
[USER LOG] liquidation-no-action proximity=30.000 threshold=18 reason=Position is sufficiently healthy; no defensive action required.

Workflow Simulation Result:
 "SAFE"
```

### Experiment 2: Change Policy Secrets, Watch the Behavior Change

Edit the policy values in `.env` (**no code changes needed!**), for example:

```bash
# Lower the per-execution cap to 2000 — add_collateral gets truncated to $2,000
MOCK_LIQUIDATION_MAX_STABLECOIN_RESERVE_DEPLOYMENT=2000

# Or switch to a "debt-first" sequencing preference
MOCK_LIQUIDATION_DEFENSIVE_ACTION_SEQUENCING_PREFERENCE=debt-first
```

Re-run the simulation and observe how the returned action amounts and ordering change. This demonstrates the power of policy-as-secrets: **tune the strategy without touching the code, and the strategy stays invisible to outsiders**.

### Experiment 3: Hit the Reserve Red Line

```bash
# Raise the minimum reserve balance to 9000 — any deployment breaches the floor
MOCK_LIQUIDATION_MIN_STABLECOIN_RESERVE_BALANCE=9000
```

The simulation now fails with a `breaches reserve floor` error thrown by `enforcePolicy` — the red-line rule's hard constraint in action.

## Recap: What Just Happened

```
Mock Server (local port 8787)
   │  ① risk snapshot ② LLM policy decision ③ execute defense
   ▼
CRE Simulator
   │  Compiles main.ts → WASM, executes along the (simulated) enclave path
   │  11 secrets injected via the secrets.yaml mapping
   │  LLM decision → enforcePolicy hard checks → execution
   ▼
Result: SAFE (no action) or DEFENDED (defense executed)
```

## What's Next

Both case studies are done! Let's wrap up the bootcamp and plan your next steps.
