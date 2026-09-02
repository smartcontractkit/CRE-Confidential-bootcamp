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

## 2. Policy-as-Secrets: 1 `getSecrets` Call (10 Secrets)

This is the case study's core design decision — **keep every policy parameter in the Vault DON**, not in the config file or the code. All 10 secrets are fetched together in a single batched `getSecrets` call rather than 10 separate `getSecret` round trips:

```typescript
export const onCronTrigger = async (runtime: TeeRuntime<Config>): Promise<string> => {
  const { mock_base_url, openai_url, openai_model, secrets_ids } = runtime.config;

  const secrets = runtime
    .getSecrets([
      { id: secrets_ids.exchange_api_key_id },
      { id: secrets_ids.openai_api_key_id },
      { id: secrets_ids.liquidation_warning_action_threshold_secret_id },
      { id: secrets_ids.minimum_health_factor_secret_id },
      { id: secrets_ids.target_health_factor_secret_id },
      { id: secrets_ids.maximum_stablecoin_reserve_deployment_secret_id },
      { id: secrets_ids.minimum_stablecoin_reserve_balance_secret_id },
      { id: secrets_ids.maximum_collateral_allocation_secret_id },
      { id: secrets_ids.maximum_partial_debt_repayment_secret_id },
      { id: secrets_ids.preferred_venues_secret_id },
    ])
    .result();

  // ① Two traditional credentials
  const exchangeApiKey = secrets[secrets_ids.exchange_api_key_id].value;
  const openAiApiKey = secrets[secrets_ids.openai_api_key_id].value;

  // ② Seven numeric policy-parameter secrets
  const liquidationWarningActionThreshold = parseRequiredSecretNumber(
    secrets[secrets_ids.liquidation_warning_action_threshold_secret_id].value,
    secrets_ids.liquidation_warning_action_threshold_secret_id,
  );
  const minimumHealthFactor = parseRequiredSecretNumber(
    secrets[secrets_ids.minimum_health_factor_secret_id].value,
    secrets_ids.minimum_health_factor_secret_id,
  );
  const targetHealthFactor = parseRequiredSecretNumber(
    secrets[secrets_ids.target_health_factor_secret_id].value,
    secrets_ids.target_health_factor_secret_id,
  );
  const maxStablecoinReserveDeployment = parseRequiredSecretNumber(
    secrets[secrets_ids.maximum_stablecoin_reserve_deployment_secret_id].value,
    secrets_ids.maximum_stablecoin_reserve_deployment_secret_id,
  );
  const minStablecoinReserveBalance = parseRequiredSecretNumber(
    secrets[secrets_ids.minimum_stablecoin_reserve_balance_secret_id].value,
    secrets_ids.minimum_stablecoin_reserve_balance_secret_id,
  );
  const maxCollateralAllocation = parseRequiredSecretNumber(
    secrets[secrets_ids.maximum_collateral_allocation_secret_id].value,
    secrets_ids.maximum_collateral_allocation_secret_id,
  );
  const maxPartialDebtRepayment = parseRequiredSecretNumber(
    secrets[secrets_ids.maximum_partial_debt_repayment_secret_id].value,
    secrets_ids.maximum_partial_debt_repayment_secret_id,
  );

  // ③ NOT a secret — sequencing preference now comes straight from public workflow config
  const defensiveSequencePreference = parseExecutionSequencePreference(
    runtime.config.defensive_action_sequencing_preference,
  );  // "collateral-first" | "debt-first" | "balanced"

  // ④ List-typed preferred_venues secret has a dedicated parser
  const preferredVenues = parseVenueListSecret(
    secrets[secrets_ids.preferred_venues_secret_id].value,
  );  // ["binance", "onchain", "coinbase"]

  runtime.log("liquidation-getsecrets-ok");
```

Why do it this way?

- **Parameters written in the config are visible to anyone who can read the workflow configuration**; placed in the Vault DON, they only ever appear inside the enclave.
- **Batching into one `getSecrets` call** resolves all 10 secrets (2 credentials + 8 policy-parameter secrets) in a single round trip to the Vault DON, instead of 10 individual `getSecret` calls.
- Numeric secrets are validated with `parseRequiredSecretNumber` (must be a finite number or it throws) — fail fast on malformed secret content instead of making decisions with a bad threshold.
- `defensive_action_sequencing_preference` is the one exception: it's read directly from `runtime.config`, not from `getSecrets` — it's plain workflow configuration, not Vault DON-protected.
- These values are then assembled into the in-memory `Policy` object used throughout the execution.

## 3. The Risk Snapshot and the Deterministic Risk Score

```typescript
const client = new HTTPClient();
const risk = collectRiskSnapshot(runtime, client, mock_base_url, exchangeApiKey);
// GET /risk-state → prices, HF, liquidation proximity, LTV, volatility, reserves... (confidential HTTP)

// ... policy object assembled from the secrets fetched above ...

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

const llmResponse = postJson(
  runtime,
  client,
  openai_url,
  {
    model: openai_model,
    input: [
      {
        role: "system",
        content:
          "You are a liquidation-defense policy engine. Emit strict JSON only with key names exactly as requested.",
      },
      {
        role: "user",
        content: prompt,
      },
    ],
  },
  {
    ...JSON_HEADERS,
    // ← confidential header: only ever readable inside the enclave
    Authorization: `Bearer ${openAiApiKey}`,
  },
);

const decision = parseLlmDecision(extractOpenAiText(llmResponse));
// → { shouldDefend, reasoning, actions[] }
```

Notice the prompt design: **the entire `policy` object is sent straight to the LLM**. In a non-confidential workflow this would be dangerous (strategy leakage), but inside the enclave the request and response bodies stay confidential end to end — which is exactly why this case study must run confidentially.

## 5. Deterministic Guardrails for the LLM (The Gem of This Case Study) ⭐

The LLM can propose, but it is **not trusted unconditionally**. `enforcePolicy` applies deterministic rules to hard-check and correct the LLM's output:

```typescript
export const enforcePolicy = (
  decision: LiquidationDecision,
  policy: Policy,
  risk: RiskState,
): ExecutableAction[] => {
  if (!decision.shouldDefend || decision.actions.length === 0) {
    return [];
  }

  let projectedReserve = risk.usdc_reserve;
  const executable: ExecutableAction[] = [];

  for (const action of decision.actions) {
    const amount = Math.max(0, action.amountUsd ?? 0);
    const repayPctRaw = Math.max(0, action.repayPct ?? 0);
    const cappedRepayPct = Math.min(repayPctRaw, policy.max_partial_debt_repayment_pct);

    // ① Collateral actions: cap against max_reserve_deployment_usdc, then check the reserve floor
    if (
      action.type === "add_collateral" ||
      action.type === "bridge_and_add_collateral" ||
      action.type === "swap_reserve_to_collateral"
    ) {
      const capped = Math.min(amount, policy.max_reserve_deployment_usdc);
      projectedReserve -= capped;

      if (projectedReserve < policy.min_reserve_balance_usdc) {
        throw new Error(
          `action ${action.type} breaches reserve floor: projected ${projectedReserve.toFixed(2)} < floor ${policy.min_reserve_balance_usdc}`,
        );
      }

      executable.push({
        ...action,
        amountUsd: capped,
        repayPct: 0,
        venue: chooseVenue(action, policy.preferred_venues),
      });
      continue;
    }

    // ② Reserve-funded repayment actions: same cap + reserve floor check as ①
    if (
      action.type === "repay_with_reserves" ||
      action.type === "swap_reserve_to_borrowed_and_repay"
    ) {
      const capped = Math.min(amount, policy.max_reserve_deployment_usdc);
      projectedReserve -= capped;

      if (projectedReserve < policy.min_reserve_balance_usdc) {
        throw new Error(
          `action ${action.type} breaches reserve floor: projected ${projectedReserve.toFixed(2)} < floor ${policy.min_reserve_balance_usdc}`,
        );
      }

      executable.push({
        ...action,
        amountUsd: capped,
        repayPct: 0,
        venue: chooseVenue(action, policy.preferred_venues),
      });
      continue;
    }

    // ③ Partial repayment: % capped by max_partial_debt_repayment_pct; amount comes from outstanding debt, not the reserve
    if (action.type === "partial_debt_repayment") {
      const boundedRepayPct = Math.min(cappedRepayPct, 100);
      executable.push({
        ...action,
        amountUsd: (risk.outstanding_debt_usd * boundedRepayPct) / 100,
        repayPct: boundedRepayPct,
        venue: chooseVenue(action, policy.preferred_venues),
      });
      continue;
    }

    // ④ Full repayment: NOT capped by max_reserve_deployment_usdc — repays the entire debt — but still checked against the reserve floor
    if (action.type === "full_debt_repayment") {
      const amountUsd = risk.outstanding_debt_usd;
      projectedReserve -= amountUsd;

      if (projectedReserve < policy.min_reserve_balance_usdc) {
        throw new Error(
          `action ${action.type} breaches reserve floor: projected ${projectedReserve.toFixed(2)} < floor ${policy.min_reserve_balance_usdc}`,
        );
      }

      executable.push({
        ...action,
        amountUsd,
        repayPct: 100,
        venue: chooseVenue(action, policy.preferred_venues),
      });
      continue;
    }

    executable.push({
      ...action,
      amountUsd: amount,
      repayPct: 0,
      venue: chooseVenue(action, policy.preferred_venues),
    });
  }

  // ⑤ Order the actions per the sequencing preference
  return orderActions(executable, policy.execution_sequence_preference);
};
```

This is the classic "AI + rules" architecture: **the LLM generates a plan in a complex situation, and deterministic code holds the line on capital safety**. Even if the LLM hallucinates an action to "deploy $1,000,000," it gets truncated by the cap or stopped by the red-line exception. Note the asymmetry on `full_debt_repayment`: it skips the deployment cap entirely (it must repay 100% of the debt, not a truncated amount), but it's still stopped cold by the reserve-floor red line if the payoff would drain reserves too far.

### Action Sequencing Preferences

```typescript
const priorities = {
  "collateral-first": { add_collateral: 1, bridge_and_add_collateral: 2, ..., full_debt_repayment: 7 },
  "debt-first":       { repay_with_reserves: 1, ..., add_collateral: 5, ... },
  "balanced":         { partial_debt_repayment: 1, bridge_and_add_collateral: 2, repay_with_reserves: 3, ... },
};
```

The same defense plan can execute in a completely different order for different users, driven by `defensive_action_sequencing_preference`. Unlike the numeric thresholds and preferred-venues list, this value is read from public workflow config rather than the Vault DON — it's the one policy parameter that isn't a secret.

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
  runtime,
  client,
  `${mock_base_url}/execute-defense`,
  {
    riskScore,
    liquidationProximityPct: risk.liquidation_proximity_pct,
    reasoning: decision.reasoning,
    actions,
  },
  {
    ...JSON_HEADERS,
    "x-exchange-api-key": exchangeApiKey,
  },
);

runtime.log(
  `liquidation-defense-executed action_count=${actions.length} execution_id=${asString(defenseResponse.execution_id, "unknown")}`,
);

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
| Confidential assets | Scanner + LLM credentials | Credentials + **8 policy-parameter secrets** (sequencing preference is public config) |
| Decision-making | Dual LLMs + conservative merge rules | LLM decision + **deterministic guardrails** |
| Capability restrictions | `preHook` allowlist (HTTP ≤ 8, Report ≤ 1, secrets ≤ 3) | None declared (default limits) |
| Data leaving the enclave | Verdict code + risk mask (optionally onchain) | Only the defense action instructions (to the execution endpoint) |
| Number of LLMs | 2 (Primary / Secondary) | 1 policy engine |

## Recap: Why Confidentiality Is Necessary

| Risk | Regular workflow | Confidential Workflow |
|------|------------------|----------------------|
| Risk thresholds (when you act) | Visible to nodes → predictable and exploitable | Enter the enclave only, as secrets |
| Capital caps (how much you can deploy) | Visible to nodes → exposes your defensive boundary | Enter the enclave only, as secrets |
| Execution preferences (how you act) | Visible to nodes → front-runnable | Read from public workflow config, not a Vault secret — an intentional exception; capital caps and thresholds still stay enclave-only |
| Exchange / LLM credentials | Plaintext passes through node memory | Released by the Vault DON directly into the enclave |
| Risk snapshot (your position and reserves) | Visible to nodes | Confidential HTTP responses stay inside the enclave |

## What's Next

Time to run it! Start the mock server and watch the workflow reach a DEFENDED decision on the default risk data.
