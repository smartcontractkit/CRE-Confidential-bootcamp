# Case Study 1: Key Code Walkthrough

Open `ai-audit-firewall/ai-audit-firewall-ts/main.ts` and let's walk through how a Chainlink Confidential Workflow is implemented. The file is about 770 lines, but **there are only 5 core confidential-computing patterns** — master them and you've mastered Confidential Workflow development.

## 1. Registering a Confidential Handler with `handlerInTee`

This is the only entry-point change needed to turn a regular workflow into a Confidential Workflow:

```typescript
import {
  CronCapability,
  Runner,
  handlerInTee,          // ← API for registering a confidential handler
  type TeeRuntime,       // ← the enclave-specific runtime type
  type Workflow,
} from "@chainlink/cre-sdk";

export const initWorkflow = (config: Config): Workflow<Config> => {
  // ... config validation ...

  const cron = new CronCapability();

  return [
    handlerInTee(
      cron.trigger({ schedule: config.schedule }),   // ① Trigger: unchanged
      onCronTrigger,                                  // ② Callback: receives a TeeRuntime
      [{ tee: "nitro", regions: ["us-west-2"] }],     // ③ TEE requirements: AWS Nitro + region
      {
        preHook: (cfg: Config) => buildRestrictions(cfg), // ④ Capability restrictions (see §5)
      },
    ),
  ];
};
```

Key points:

- **`handlerInTee(trigger, callback, teeRequirements, options)`**: declares that this handler's callback must execute inside a TEE, and specifies the acceptable TEE types (`nitro`) and regions.
- The callback signature changes from `Runtime<Config>` to **`TeeRuntime<Config>`** — which exposes in-enclave capabilities (such as the enclave release path of `getSecrets`, and confidential HTTP calls) plus the interface for crossing back to the DON.
- As with a regular workflow, the Workflow DON still listens for the CRON trigger; when it fires, the DON hands execution to the enclave.

## 2. Fetching Secrets Inside the Enclave (the Vault DON)

```typescript
export const runAuditFirewall = async (
  runtime: TeeRuntime<Config>,
  client = new HTTPClient(),
): Promise<string> => {
  const { mock_base_url, scanner_url, primary_llm_url, secondary_llm_url, secrets_ids } = runtime.config;

  // ① One batched call fetches all 3 secrets at once — released by the Vault
  //    DON directly into the attested enclave at the moment your code needs them
  const secrets = runtime
    .getSecrets([
      { id: secrets_ids.scanner_api_key_id },
      { id: secrets_ids.primary_llm_api_key_id },
      { id: secrets_ids.secondary_llm_api_key_id },
    ])
    .result();

  // ② Look each value up by its secret ID from the batched result
  const scannerApiKey = secrets[secrets_ids.scanner_api_key_id].value;
  const primaryLlmApiKey = secrets[secrets_ids.primary_llm_api_key_id].value;
  const secondaryLlmApiKey = secrets[secrets_ids.secondary_llm_api_key_id].value;

  runtime.log("audit-firewall-getsecrets-ok");
  // ...
```

Key points:

- The `runtime.getSecrets([...])` call happens **inside the enclave**, and the plaintext secrets **never pass through Workflow DON nodes** — the Vault DON releases them directly into the enclave after verifying the enclave's attestation. Batching all 3 secrets into one call is more efficient than issuing 3 separate `getSecret` calls.
- Secret IDs are not hardcoded; they're injected via the `secrets_ids` field in `config.staging.json`. In local simulation, the actual secret values come from the `.env` file at the project root (mapped through `secrets.yaml`).

## 3. Confidential HTTP Calls

Every outbound request in the workflow goes through the same `HTTPClient`. Because the calls originate inside the enclave, **the URL, headers (including API keys), and response body all remain confidential from node operators**:

```typescript
const getJson = (
  runtime: TeeRuntime<Config>,
  client: HTTPClient,
  url: string,
  headers: Record<string, string>,
): Record<string, unknown> => {
  const response = client
    .sendRequest(runtime, {
      url,
      method: "GET",
      headers,                      // ← carries x-scanner-api-key, confidential
    })
    .result();

  const raw = decodeBody(response.body);
  if (response.statusCode >= 400) {
    throw new Error(`request failed status=${response.statusCode} body=${raw}`);
  }
  return parseJson(raw);            // ← the response body stays inside the enclave
};
```

POST requests work the same way, except the body must be base64-encoded first:

```typescript
const bodyBytes = new TextEncoder().encode(JSON.stringify(body));
const encodedBody = Buffer.from(bodyBytes).toString("base64");

const response = client
  .sendRequest(runtime, {
    url,
    method: "POST",
    body: encodedBody,
    headers,
  })
  .result();
```

### What Is the Transaction Data Being Fetched?

The first call the workflow makes (`GET /transaction-proposal`, via `collectTransactionProposal`) retrieves the key fields of a transaction that **has not been submitted onchain yet — it is a proposed transaction about to be sent/executed**. You can see this from the `TransactionProposal` type (lines 40–51 of `main.ts`):

```typescript
type TransactionProposal = {
  chain_selector: number;
  chain_name: string;
  tx_hash: string;
  from_address: string;
  token_contract_address: string;
  protocol_contract_address: string;
  calldata: string;
  value_wei: string;
  signer: string;
  requested_action: string;
};
```

| Field | Meaning |
|-------|---------|
| `chain_selector` / `chain_name` | The target chain |
| `tx_hash` | The proposal's transaction hash |
| `from_address` / `signer` | The initiator and the signer |
| `token_contract_address` | The token contract involved |
| `protocol_contract_address` | The protocol contract to interact with |
| `calldata` / `value_wei` | The call data and the amount of native token being transferred |
| `requested_action` | The action being requested (e.g., `transfer`) |

This is exactly the "pre-execution" nature of the firewall: the workflow screens the transaction **before it ever touches the chain**, and the two contract addresses above become the audit targets in the next stage.

### A Key Defense in Stage 2: Validate Credentials Before Trusting Data

Before trusting the contract data returned by the scanner, the workflow validates the credential's own permission scopes — an easily overlooked but professional security detail:

```typescript
const validateScannerCredentials = (runtime, client, scannerUrl, scannerApiKey) => {
  const response = getJson(runtime, client, `${scannerUrl}/credentials/verify`, {
    ...JSON_HEADERS,
    "x-scanner-api-key": scannerApiKey,
  });

  const validation = parseScannerCredentialValidation(response);
  const hasVerificationScope = validation.scopes.includes("verification:read");
  const hasContractScope = validation.scopes.includes("contracts:read");

  // Invalid credential or missing required scopes → throw and abort this execution
  if (!validation.valid || !hasVerificationScope || !hasContractScope) {
    throw new Error(`scanner credentials failed validation ...`);
  }
  return validation;
};
```

## 4. The Dual-LLM Audit and the Conservative Verdict

### Two Models, Two Perspectives

```typescript
// Primary: audits the token contract + transaction proposal
const primaryAnalysis = requestAuditModel(
  runtime,
  client,
  primary_llm_url,
  primaryLlmApiKey,
  "audit-primary",
  buildPrimaryPrompt(proposal, tokenContract),
);

// Secondary: audits the protocol contract, given Primary's findings as prior context
const secondaryAnalysis = requestAuditModel(
  runtime,
  client,
  secondary_llm_url,
  secondaryLlmApiKey,
  "audit-secondary",
  buildSecondaryPrompt(proposal, tokenContract, protocolContract, primaryAnalysis),
);
```

Each model is required to emit strict JSON: four risk signals (`obfuscatedTax`, `privilegeEscalation`, `externalCallRisk`, `logicBomb`) + a recommendation (`allow`/`deny`/`review`) + confidence + reasoning.

### Merging the Verdicts: Better Safe Than Sorry

```typescript
export const determineVerdict = (primary, secondary): FirewallVerdict => {
  const combinedFlags = mergeFlags(primary.riskFlags, secondary.riskFlags);

  // Either model flags any risk signal → block
  if (hasMaliciousRisk(combinedFlags)) {
    return "DENY";
  }

  // Either model is unsure / low confidence / models disagree → manual review
  const reviewRequested = primary.recommendation === "review" || secondary.recommendation === "review";
  const lowConfidence = primary.confidence < 0.7 || secondary.confidence < 0.7;
  if (reviewRequested || lowConfidence || primary.recommendation !== secondary.recommendation) {
    return "MANUAL_REVIEW";
  }

  return "ALLOW";
};
```

Here's how the "better safe than sorry" mechanism plays out:

| Trigger condition | Result | Why it's conservative |
|-------------------|--------|----------------------|
| Any risk flag from either model is `true` (`mergeFlags` is pure OR logic) | `DENY` | No consensus needed — one model flagging a problem is enough |
| Either model recommends `review` | `MANUAL_REVIEW` | If one auditor isn't sure, there's no automatic pass |
| Either model has `confidence < 0.7` | `MANUAL_REVIEW` | Even an "allow" conclusion doesn't count if the model isn't confident |
| The two models' recommendations disagree | `MANUAL_REVIEW` | Disagreement between auditors = not trustworthy |

Notice that `ALLOW` is the hardest verdict to reach: it requires **both** models to agree on "allow," **both** with confidence ≥ 0.7, and **zero** risk flags between them.

The workflow then writes the full context to the audit log (`POST /audit-log`) and triggers the firewall action (`POST /firewall-action`) — both requests also carry confidential credentials and originate inside the enclave.

## 5. Capability Restrictions: Least Privilege

`handlerInTee`'s `preHook` lets you declare a **strict capability allowlist** for the execution — even if the workflow code were tampered with, it couldn't invoke capabilities or secrets beyond the declaration:

```typescript
export const buildRestrictions = (config: Config) => {
  const httpRestrictor = new HTTPClientRestrictor();
  const capabilityRestrictions = [
    httpRestrictor.limitSendRequest(8),          // HTTP: max 8 calls
    { method: { id: CONSENSUS_CAPABILITY_ID, method: "Report", maxCalls: 1 } }, // Report: max 1 call
  ];

  // If EVM writes are configured, limit writeReport to 1 call
  const evmConfig = config.evms?.[0];
  if (evmConfig?.chain_selector_name) { /* ... */ }

  return {
    capabilities: {
      type: "CAPABILITY_RESTRICTION_TYPE_CLOSED",  // closed allowlist
      maxTotalCalls: 10,
      restrictions: capabilityRestrictions,
    },
    secrets: {
      maxSecrets: 3,                                // max 3 secrets
      restrictions: [
        { exactSecret: { id: secrets_ids.scanner_api_key_id, namespace: "main" } },
        { exactSecret: { id: secrets_ids.primary_llm_api_key_id, namespace: "main" } },
        { exactSecret: { id: secrets_ids.secondary_llm_api_key_id, namespace: "main" } },
      ],
    },
  };
};
```

This is defense in depth for confidential computing: the enclave keeps data invisible to nodes, while restrictions constrain what the workflow "is allowed to do."

## 6. Crossing Back to the DON: Writing the Verdict Onchain

So far, everything has stayed inside the enclave. But writing the verdict **onchain** (an operation that requires DON consensus) means explicitly crossing back to the regular DON runtime:

```typescript
const writeVerdictOnChain = async (runtime, result): Promise<string | undefined> => {
  const evmConfig = runtime.config.evms?.[0];
  if (!evmConfig) return undefined;

  const network = getNetwork({ chainFamily: "evm", chainSelectorName: evmConfig.chain_selector_name });
  const evmClient = new EVMClient(network.chainSelector.selector);

  // Encode the verdict as an ABI payload: (uint8 verdictCode, uint8 riskMask, uint64 chainSelector)
  const reportPayload = encodeVerdictReport(result, BigInt(network.chainSelector.selector));

  // ★ Explicitly cross back to the Workflow DON: generate a DON-signed report
  const donRuntime = runtime.usingTheDons();
  const reportResponse = donRuntime
    .report({
      encodedPayload: hexToBase64(reportPayload),
      encoderName: "evm",
      signingAlgo: "ecdsa",
      hashingAlgo: "keccak256",
    })
    .result();

  // Deliver the signed report to the consumer contract via the Forwarder
  const writeResult = evmClient
    .writeReport(donRuntime, {
      receiver: evmConfig.consumer_address,
      report: reportResponse,
      gasConfig: { gasLimit: evmConfig.gas_limit },
    })
    .result();

  if (writeResult.txStatus !== TxStatus.SUCCESS) {
    throw new Error(`onchain write failed with status ${writeResult.txStatus}`);
  }
  return bytesToHex(writeResult.txHash || new Uint8Array(32));
};
```

This is exactly what the confidentiality boundary design means:

- **What crosses out of the enclave is your explicit choice** — here, only three encoded values cross (verdict code, risk mask, chain selector); all audit process data stays inside the enclave.
- `runtime.usingTheDons()` returns a runtime for operations that need DON consensus; once data passes through it, it's handled like any non-confidential capability call.
- Onchain, `AuditFirewallConsumer.sol` receives the report: it extends `ReceiverTemplate` and decodes `(uint8, uint8, uint64)` in `_processReport`, recording the `Verdict` (`Allow`/`Deny`/`ManualReview`).

```solidity
function _processReport(bytes calldata report) internal override {
    (uint8 verdictCode, uint8 riskMask, uint64 chainSelector) =
        abi.decode(report, (uint8, uint8, uint64));
    // verdictCode: 1=Allow, 2=Deny, 3=ManualReview
    ...
    emit VerdictReceived(verdict, riskMask, chainSelector);
}
```

## 7. Putting It Together: The Main Flow

```typescript
export const runAuditFirewall = async (runtime, client = new HTTPClient()) => {
  // ① Fetch 3 secrets inside the enclave (1 batched getSecrets call)
  // ② GET /transaction-proposal          — get the proposed transaction
  // ③ GET /credentials/verify            — validate scanner credentials
  // ④ GET /contracts/{token} /{protocol} — fetch contract source & ABI
  //    └─ any contract unverified → DENY + log, early return
  // ⑤ POST /v1/analysis/primary          — LLM #1 audits the token contract
  // ⑥ POST /v1/analysis/secondary        — LLM #2 audits the protocol contract
  // ⑦ determineVerdict                   — conservative merge
  // ⑧ POST /audit-log + /firewall-action — record & enforce
  // ⑨ writeVerdictOnChain                — cross back to the DON (optional)
  // ⑩ Return the JSON result
};
```

## Recap: Why Confidentiality Is Necessary

Now consider what this case study would look like without a Confidential Workflow:

| Risk | Regular workflow | Confidential Workflow |
|------|------------------|----------------------|
| LLM / scanner API keys | Plaintext passes through DON node memory, visible to node operators | Released by the Vault DON directly into the enclave; invisible to nodes |
| Contract source code and audit intermediates | Visible to nodes | Processed only in enclave memory |
| Audit requests (URL / headers / body) | Visible to nodes | Confidential HTTP; invisible to nodes |
| Audit criteria exploited to evade detection | Possible | The evaluation process never leaves the enclave |

## What's Next

With the code explained, let's run it — start the mock server and simulate this workflow with the CRE CLI!
