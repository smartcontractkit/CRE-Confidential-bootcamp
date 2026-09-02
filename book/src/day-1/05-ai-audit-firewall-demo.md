# Case Study 1: Demo

Let's run the AI Audit Firewall end to end. It takes only 4 steps: clone the repo → configure the environment → start the mock server → simulate.

## Step 1: Clone the Templates Repo

```bash
git clone https://github.com/smartcontractkit/cre-templates.git
cd cre-templates/starter-templates/confidential-workflows/ai-audit-firewall
```

## Step 2: Set Up Environment Variables

Create a `.env` at the project root (`ai-audit-firewall/`):

```bash
cp .env.example .env
```

`.env.example` is pre-filled with demo defaults:

```bash
# Ethereum private key (optional for local simulate; required for real chain writes)
CRE_ETH_PRIVATE_KEY=

MOCK_PORT=8787

MOCK_SCANNER_API_KEY=mock-scanner-key
MOCK_PRIMARY_LLM_API_KEY=mock-primary-llm-key
MOCK_SECONDARY_LLM_API_KEY=mock-secondary-llm-key
```

> **Note**: These three `MOCK_*` values are the "secrets" the workflow uses during simulation. The CRE CLI injects them as secrets according to the `secrets.yaml` mapping. In a real deployment they'd be replaced with real scanner and LLM credentials, managed by the Vault DON.

## Step 3: Install Dependencies and Start the Mock Server

```bash
cd ai-audit-firewall-ts
bun install
bun run mock:server
```

You'll see:

```bash
Mock server running at http://127.0.0.1:8787
```

**Keep this terminal running** — every API request will hit it.

## Step 4: Typecheck and Unit Tests (Optional but Recommended)

Open a **second terminal**:

```bash
cd cre-templates/starter-templates/confidential-workflows/ai-audit-firewall/ai-audit-firewall-ts
bun run typecheck
bun run test
```

The tests cover the verdict logic (`determineVerdict`), the capability restrictions (`buildRestrictions`), and the full main flow executed against a mocked HTTP client — reading the tests first is also a great way to understand this workflow.

## Step 5: Simulate the Workflow

Back at the project root, start the simulation with the CRE CLI:

```bash
cd ..
cre workflow simulate ./ai-audit-firewall-ts --target=staging-settings
```

> **Note**: Simulation compiles the workflow to WASM and runs it on your machine, but it makes **real calls** to the mock server's HTTP endpoints. A CRON-triggered workflow executes once directly in simulation.

You'll see output similar to:

```bash
[SIMULATION] Simulator Initialized

[USER LOG] audit-firewall-getsecrets-ok
[USER LOG] audit-firewall-scanner-credentials-ok provider=mock-scanner scopes=contracts:read,verification:read
[USER LOG] audit-firewall-onchain-report-start
[USER LOG] audit-firewall-onchain tx_hash=0x...
[USER LOG] audit-firewall-complete verdict=ALLOW audit_log_id=audit_...

Workflow Simulation Result:
 "{\"verdict\":\"ALLOW\",\"reasoning\":\"...\",\"riskFlags\":{...}, ...}"

[SIMULATION] Execution finished signal received
```

### Reading the Output Line by Line

| Log line | Meaning |
|----------|---------|
| `audit-firewall-getsecrets-ok` | All 3 secrets were successfully fetched inside the (simulated) enclave via a single batched `getSecrets` call |
| `audit-firewall-scanner-credentials-ok` | Scanner credentials validated, with `contracts:read` and `verification:read` scopes |
| `audit-firewall-onchain-report-start` | Started generating the DON-signed report (crossing back to the DON runtime) |
| `audit-firewall-onchain tx_hash=...` | Simulated onchain write completed (no `--broadcast`, so nothing actually goes onchain) |
| `audit-firewall-complete verdict=...` | The final verdict and audit log ID |

### Why the Default Data Yields ALLOW

The two contracts built into the mock server — `MockERC20Token` and `MockProtocolRouter` — are both verified, "clean" contracts with no malicious traits. Both mock LLMs return `allow` with confidence 0.93 (≥ 0.7) and agree with each other, so under the `determineVerdict` rules the final verdict is **ALLOW**.

> The mock LLMs are deterministic, keyword-based rule implementations — not real models — which keeps demo results reproducible. To plug in real LLMs, swap `primary_llm_url` / `secondary_llm_url` in `config.staging.json` for real endpoints.

## Hands-On Experiments (Optional)

### Experiment 1: Make a Contract "Unverified" → Trigger DENY

Edit `mock-server.js` and change `MockERC20Token`'s `verified: true` to `false`. Restart the mock server and simulate again:

```bash
verdict=DENY  reason="One or more contracts are not verified by the scanner."
```

The workflow aborts early at Stage 2: no LLM calls at all — it logs the audit record and executes the DENY firewall action immediately. When verification fails, there's nothing left to audit.

### Experiment 2: A Real Onchain Write

1. Deploy `contracts/AuditFirewallConsumer.sol` to Sepolia (the constructor argument is the [CRE Forwarder address](https://docs.chain.link/cre/guides/workflow/using-evm-client/supported-networks-go#understanding-forwarder-addresses))
2. Fill the deployed address into `evms[0].consumer_address` in `config.staging.json`
3. Set `CRE_ETH_PRIVATE_KEY` in `.env`
4. Run `cre workflow simulate ./ai-audit-firewall-ts --target=staging-settings --broadcast`

`--broadcast` makes the simulator execute a real onchain write transaction, after which you can read `lastVerdict` onchain.

> **Required configuration**: for local simulation on Sepolia, deploy the consumer contract with the **mock Forwarder contract** address `0x15fC6ae953E024d975e77382eEeC56A9101f9F88` as the constructor argument. Forwarder addresses for other networks are listed in the [Forwarder Directory](https://docs.chain.link/cre/guides/workflow/using-evm-client/forwarder-directory-go).

## Recap: What Just Happened

```
Mock Server (local port 8787)
   │  ① proposal ② contract data ③ credential check ④ dual-LLM audit ⑤ log & firewall action
   ▼
CRE Simulator
   │  Compiles main.ts → WASM, executes along the (simulated) enclave path
   │  Secrets injected via the secrets.yaml mapping
   ▼
(optional) Sepolia consumer contract ← DON-signed report
```

## 🎉 Day 1 Complete!

You have successfully:

- ✅ Learned CRE's core concepts (Workflow / Trigger / Capability / DON)
- ✅ Learned how Confidential Workflows work: TEEs, enclaves, the Vault DON, attestation
- ✅ Mastered the 5 core patterns of confidential development: `handlerInTee`, in-enclave batched `getSecrets`, confidential HTTP, capability restrictions, and `usingTheDons()`
- ✅ Run your first Confidential Workflow

Tomorrow we move on to the second case study — **Automated Liquidation Protection** — and see how confidential strategy parameters defend against front-running. See you then!
