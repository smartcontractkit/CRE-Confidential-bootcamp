# Case Study 2: Key Code Walkthrough

Open `automated-liquidation-protection/automated-liquidation-protection-ts/main.ts` (about 640 lines). All the confidential patterns from yesterday are reused here, so we'll focus on what's **new in this case study**: policy-as-secrets, deterministic guardrails backing up the LLM, and action sequencing.

## 1. Confidential Handler Registration (Recap)

```typescript
import {
  CronCapability,
  HTTPClient,
  NITRO_REGIONS,
  Runner,
  handlerInTee,
  type TeeRuntime,
  type Workflow,
} from "@chainlink/cre-sdk";

export const initWorkflow = (config: Config): Workflow<Config> => {
  // ... config and secrets_ids validation ...

  const cron = new CronCapability();

  return [
    handlerInTee(
      cron.trigger({ schedule: config.schedule }),
      onCronTrigger,
      [{ tee: "nitro", regions: [NITRO_REGIONS[0]] }],
    ),
  ];
};
```

Exactly the same pattern as Case Study 1: CRON trigger + Nitro enclave execution. The differences start inside the callback.

## 2. Policy-as-Secrets: 11 `getSecret` Calls

This is the case study's core design decision — **keep every policy parameter in the Vault DON**, not in the config file or the code:

```typescript
export const onCronTrigger = async (runtime: TeeRuntime<Config>): Promise<string> => {
  const { mock_base_url, openai_url, openai_model, secrets_ids } = runtime.config;

  // ① Two traditional credentials
  const exchangeApiKey = runtime.getSecret({ id: secrets_ids.exchange_api_key_id }).result().value;
  const openAiApiKey = runtime.getSecret({ id: secrets_ids.openai_api_key_id }).result().value;

  // ② Nine policy parameters — all of them secrets!
  const liquidationWarningActionThreshold = parseRequiredSecretNumber(
    runtime.getSecret({ id: secrets_ids.liquidation_warning_action_threshold_secret_id }).result().value,
    secrets_ids.liquidation_warning_action_threshold_secret_id,
  );
  const minimumHealthFactor = parseRequiredSecretNumber(
    runtime.getSecret({ id: secrets_ids.minimum_health_factor_secret_id }).result().value,
    secrets_ids.minimum_health_factor_secret_id,
  );
  // ... target_health_factor, max/min reserve, collateral allocation cap, partial repayment cap ...

  // ③ Enum- and list-typed policies have dedicated parsers
  const defensiveSequencePreference = parseExecutionSequencePreference(
    runtime.getSecret({ id: secrets_ids.defensive_action_sequencing_preference_secret_id }).result().value,
  );  // "collateral-first" | "debt-first" | "balanced"
  const preferredVenues = parseVenueListSecret(
    runtime.getSecret({ id: secrets_ids.preferred_venues_secret_id }).result().value,
  );  // ["binance", "onchain", "coinbase"]
```

Why do it this way?

- **Parameters written in the config are visible to anyone who can read the workflow configuration**; placed in the Vault DON, they only ever appear inside the enclave.
- Numeric secrets are validated with `parseRequiredSecretNumber` (must be a finite number or it throws) — fail fast on malformed secret content instead of making decisions with a bad threshold.
- These values are then assembled into the in-memory `Policy` object used throughout the execution.

## 3. The Risk Snapshot and the Deterministic Risk Score

```typescript
const client = new HTTPClient();
const risk = collectRiskSnapshot(runtime, client, mock_base_url, exchangeApiKey);
// GET /risk-state → prices, HF, liquidation proximity, LTV, volatility, reserves... (confidential HTTP)

const riskScore = computeRiskScore(risk, policy);
```

`computeRiskScore` is a **deterministic** scoring function — no LLM involved, reproducible and auditable:

```typescript
export const computeRiskScore = (risk: RiskState, policy: Policy): number => {
  // The closer to liquidation, the higher the score (+5 points per 1% closer)
  const proximityRisk = Math.max(0, policy.liquidation_warning_action_threshold - risk.liquidation_proximity_pct) * 5;
  // Adds points as LTV approaches the liquidation threshold (with a 5% buffer)
  const ltvBufferRisk = Math.max(0, risk.loan_to_value_pct - (risk.liquidation_threshold_pct - 5)) * 2;
  // Heavy penalty when the health factor is below the minimum (+1 point per 0.01 below)
  const healthRisk = Math.max(0, policy.minimum_health_factor - risk.collateral_health_factor) * 100;
  // Volatility weighting
  const volatilityRisk = risk.volatility_index * 25;

  return proximityRisk + ltvBufferRisk + healthRisk + volatilityRisk;
};
```

Let's compute it with the mock data: proximity (18−12)×5=30 + LTV 0 + health (1.25−1.14)×100=11 + volatility 0.37×25=9.25 → **50.25**. Meaningful risk — defense is warranted.

## 4. The LLM as a Policy Engine

```typescript
const prompt = createOpenAiPrompt(risk, riskScore, policy);
// { objective: "Return strict JSON...", policy, risk, riskScore }

const llmResponse = postJson(runtime, client, openai_url,
  {
    model: openai_model,
    input: [
      { role: "system", content: "You are a liquidation-defense policy engine. Emit strict JSON only..." },
      { role: "user", content: prompt },
    ],
  },
  { ...JSON_HEADERS, Authorization: `Bearer ${openAiApiKey}` },  // ← confidential header
);

const decision = parseLlmDecision(extractOpenAiText(llmResponse));
// → { shouldDefend, reasoning, actions[] }
```

Notice the prompt design: **the entire `policy` object is sent straight to the LLM**. In a non-confidential workflow this would be dangerous (strategy leakage), but inside the enclave the request and response bodies stay confidential end to end — which is exactly why this case study must run confidentially.

## 5. Deterministic Guardrails for the LLM (The Gem of This Case Study) ⭐

The LLM can propose, but it is **not trusted unconditionally**. `enforcePolicy` applies deterministic rules to hard-check and correct the LLM's output:

```typescript
export const enforcePolicy = (decision, policy, risk): ExecutableAction[] => {
  if (!decision.shouldDefend || decision.actions.length === 0) {
    return [];
  }

  let projectedReserve = risk.usdc_reserve;
  const executable: ExecutableAction[] = [];

  for (const action of decision.actions) {
    // ① Cap the amount: never exceed the max reserve deployment per execution
    const capped = Math.min(amount, policy.max_reserve_deployment_usdc);
    projectedReserve -= capped;

    // ② Reserve floor red line: if a breach would result → throw and abort
    if (projectedReserve < policy.min_reserve_balance_usdc) {
      throw new Error(`action ${action.type} breaches reserve floor: ...`);
    }

    // ③ Cap partial repayment %: never exceed max_partial_debt_repayment_pct
    const boundedRepayPct = Math.min(repayPct, policy.max_partial_debt_repayment_pct, 100);

    // ④ When no venue is specified, choose from the preferred venue list
    executable.push({ ...action, venue: chooseVenue(action, policy.preferred_venues) });
  }

  // ⑤ Order the actions per the sequencing preference
  return orderActions(executable, policy.execution_sequence_preference);
};
```

This is the classic "AI + rules" architecture: **the LLM generates a plan in a complex situation, and deterministic code holds the line on capital safety**. Even if the LLM hallucinates an action to "deploy $1,000,000," it gets truncated by the cap or stopped by the red-line exception.

### Action Sequencing Preferences

```typescript
const priorities = {
  "collateral-first": { add_collateral: 1, bridge_and_add_collateral: 2, ..., full_debt_repayment: 7 },
  "debt-first":       { repay_with_reserves: 1, ..., add_collateral: 5, ... },
  "balanced":         { partial_debt_repayment: 1, bridge_and_add_collateral: 2, repay_with_reserves: 3, ... },
};
```

The same defense plan can execute in a completely different order for different users — and that preference is itself a secret, so outsiders can't predict your onchain behavior from it.

## 6. Execute the Defense, or Declare SAFE

```typescript
// Liquidation proximity still above the warning line,
// or no executable actions after policy checks → SAFE
if (risk.liquidation_proximity_pct > policy.liquidation_warning_action_threshold || actions.length === 0) {
  runtime.log(
    `liquidation-no-action proximity=${risk.liquidation_proximity_pct.toFixed(3)} threshold=${policy.liquidation_warning_action_threshold} reason=${decision.reasoning}`,
  );
  return "SAFE";
}

// Otherwise, execute the defense plan
const defenseResponse = postJson(
  runtime, client, `${mock_base_url}/execute-defense`,
  { riskScore, liquidationProximityPct: risk.liquidation_proximity_pct, reasoning: decision.reasoning, actions },
  { ...JSON_HEADERS, "x-exchange-api-key": exchangeApiKey },
);

runtime.log(`liquidation-defense-executed action_count=${actions.length} execution_id=...`);

return JSON.stringify({
  status: "DEFENDED",
  actionCount: actions.length,
  riskScore,
  executionId: asString(defenseResponse.execution_id, "unknown"),
});
```

Only two possible endings: `SAFE` (no action needed) or `DEFENDED` (N actions executed) — clear, monitorable, and alertable.

## 7. Side-by-Side with Case Study 1

| Dimension | Case 1: AI Audit Firewall | Case 2: Liquidation Protection |
|-----------|---------------------------|-------------------------------|
| Confidential assets | Scanner + LLM credentials | Credentials + **9 policy parameters** |
| Decision-making | Dual LLMs + conservative merge rules | LLM decision + **deterministic guardrails** |
| Capability restrictions | `preHook` allowlist (HTTP ≤ 8, Report ≤ 1, secrets ≤ 3) | None declared (default limits) |
| Data leaving the enclave | Verdict code + risk mask (optionally onchain) | Only the defense action instructions (to the execution endpoint) |
| Number of LLMs | 2 (Primary / Secondary) | 1 policy engine |

## Recap: Why Confidentiality Is Necessary

| Risk | Regular workflow | Confidential Workflow |
|------|------------------|----------------------|
| Risk thresholds (when you act) | Visible to nodes → predictable and exploitable | Enter the enclave only, as secrets |
| Capital caps (how much you can deploy) | Visible to nodes → exposes your defensive boundary | Enter the enclave only, as secrets |
| Execution preferences (how you act) | Visible to nodes → front-runnable | Enter the enclave only, as secrets |
| Exchange / LLM credentials | Plaintext passes through node memory | Released by the Vault DON directly into the enclave |
| Risk snapshot (your position and reserves) | Visible to nodes | Confidential HTTP responses stay inside the enclave |

## What's Next

Time to run it! Start the mock server and watch the workflow reach a DEFENDED decision on the default risk data.
