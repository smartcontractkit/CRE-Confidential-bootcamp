# Hello World: Your First Workflow

> Template source: [`cre-templates/starter-templates/hello-confidential-workflows`](https://github.com/smartcontractkit/cre-templates/tree/main/starter-templates/hello-confidential-workflows) (available in TypeScript and Go; the two implementations are behaviorally equivalent)

Before we get into the theory of confidential computing, let's run the smallest possible Confidential Workflow end to end. It takes only 4 steps: clone the repo → configure the environment → simulate → (optionally) deploy.

## What This Workflow Does

This template is the minimal end-to-end shape of a Confidential Workflow, in four steps:

| Step | What it demonstrates | API |
|------|----------------------|-----|
| 1 | Register a handler whose callback runs inside a secure enclave | `cre.handlerInTee(trigger, fn, tees)` |
| 2 | Fetch a secret released by the Vault DON, inside the enclave | `runtime.getSecret({ id })` |
| 3 | Make an HTTP call from inside the enclave (request and response stay confidential) | `HTTPClient.sendRequest(teeRuntime, req)` |
| 4 | Cross back to the Workflow DON for anything needing consensus | `runtime.usingTheDons()` |

Concretely, on every CRON tick the workflow:

1. Hands the triggered request to an enclave (AWS Nitro, `us-west-2`) instead of executing the callback on Workflow DON nodes
2. Fetches the `API_TOKEN` secret — released by the Vault DON directly into the attested enclave
3. Calls the configured URL from inside the enclave with the secret in the `Authorization` header
4. Scores the confidential response against `scoreThreshold` → verdict `APPROVE` / `REJECT`
5. Crosses back to the DON with `usingTheDons()` and generates a signed report containing **only the verdict and score** — never the secret or the raw response body

The default endpoint is `https://postman-echo.com/headers`, which echoes request headers back — no signup or real API key needed. The workflow uses it to confirm the secret really was injected inside the enclave, reported as the boolean `secret reached API: true` rather than by ever logging the token.

## Simulate the Workflow

### Step 1: Clone the Templates Repo

```bash
git clone https://github.com/smartcontractkit/cre-templates.git
cd cre-templates/starter-templates/hello-confidential-workflows/hello-confidential-workflows-ts
```

### Step 2: Set Up Environment Variables

```bash
cp .env.example .env
```

Then set `SECRET_API_TOKEN` in `.env` — with the default echo endpoint, any non-empty value works. The `secrets.yaml` at the project root maps the workflow-facing secret ID to that environment variable:

```yaml
secretsNames:
    API_TOKEN:
        - SECRET_API_TOKEN
```

> **Note**: In local simulation the CRE CLI injects secret values from `.env` according to this mapping. In a real deployment, the same secret ID (`API_TOKEN`) is resolved from the Vault DON instead — your workflow code doesn't change. That's why deployment requires an extra step (see below).

### Step 3: Install Dependencies

```bash
cd my-workflow && bun install && cd ..
```

### Step 4: Simulate

From the project root, start the simulation with the CRE CLI:

```bash
cre workflow simulate my-workflow --target staging-settings --non-interactive --trigger-index 0
```

You'll see output similar to:

```bash
[SIMULATION] Running trigger trigger=cron-trigger@1.0.0
╭────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Trigger requested TEE Execution your trigger will run in one of the following Tees:                │
│     - AWS Nitro in us-west-2                                                                       │
│ The simulator is not a real TEE, and is meant to debug.                                            │
│ Do not use it for sensitive information.                                                           │
│ During real execution, user logs for this trigger will not be visible, and will not leave the TEE. │
│ They are presented in the simulator for debugging only.                                            │
╰────────────────────────────────────────────────────────────────────────────────────────────────────╯

[USER LOG] Enclave computation complete. verdict=REJECT

✓ Workflow Simulation Result:
"REJECT (score: 371, secret reached API: true)"
```

### Reading the Output

- The simulator confirms the TEE constraint it resolved (**AWS Nitro in us-west-2**) and warns that **it is not a real enclave** — logs are shown for debugging only; in real execution they never leave the TEE.
- `secret reached API: true` means the Vault DON secret was fetched inside the enclave and arrived in the outbound request's `Authorization` header.
- The verdict can flip between `APPROVE` and `REJECT` from run to run — the score derives from the live response body, and the echo endpoint includes a per-request trace ID. Lower `scoreThreshold` in `config.staging.json` to see `APPROVE` consistently.

## Deploy the Workflow

Deployment takes 3 steps: add the secret to the Vault DON → deploy → verify. We'll use the **private registry** (authorized by your CRE login session — no wallet, no gas).

### Step 1: Add the Secret to the Vault DON (Before Deploying!)

A deployed workflow **cannot read your local `.env` file** — it fetches secrets from the Vault DON at runtime. So before deploying, you must store `API_TOKEN` in the Vault DON. Make sure `SECRET_API_TOKEN` is set in your `.env`, then run:

```bash
cre secrets create secrets.yaml --target staging-settings --secrets-auth=browser
```

The CLI reads `secrets.yaml`, picks up the value from `SECRET_API_TOKEN` in `.env`, opens a browser window to authorize against the Vault DON with your CRE login session, and stores the secret. You'll see:

```bash
Secret created: secret_id=API_TOKEN, owner=<your-organization-owner>, namespace=main
```

Verify it landed (only the ID is shown, never the value):

```bash
cre secrets list --target staging-settings --secrets-auth=browser
```

> **Note**: In a Confidential Workflow, the Vault DON releases this secret **only into an attested enclave** at the moment `getSecret()` runs — it is never exposed in plaintext to Workflow DON nodes.

### Step 2: Deploy

Add `deployment-registry: "private"` under `user-workflow` in `my-workflow/workflow.yaml`:

```yaml
staging-settings:
  user-workflow:
    workflow-name: "hello-confidential-staging"
    deployment-registry: "private"
```

Then deploy from the project root:

```bash
cre workflow deploy my-workflow --target staging-settings
```

The CLI compiles the workflow to WASM, uploads the artifacts, and registers the workflow — active immediately:

```bash
Deploying Workflow: hello-confidential-staging
Compiling workflow...
✓ Workflow compiled successfully
Uploading files...
✓ Workflow registered in private registry

Details:
   Registry:         private
   Workflow Name:    hello-confidential-staging
   Workflow ID:      <workflow-id>
   Status:           Active
   Owner:            <your-organization-owner>
```

### Step 3: Verify and Manage

```bash
cre workflow list --registry private        # confirm it's registered and Active
cre workflow pause my-workflow --target staging-settings     # pause
cre workflow activate my-workflow --target staging-settings  # resume
cre workflow delete my-workflow --target staging-settings    # permanently remove
```

The workflow now runs on its CRON schedule: every execution happens inside a real enclave, fetches `API_TOKEN` from the Vault DON, and produces a DON-signed report.

> ⚠️ **Production reminder**: the template logs `Enclave computation complete. verdict=...` inside the enclave for debugging. Remove every `runtime.log()` inside the TEE handler before any real deployment — anything logged from within a Confidential Workflow could leak the data the enclave is meant to protect.

## Key Takeaways

| Concept | One-liner |
|---------|-----------|
| `cre workflow simulate` | Runs the workflow locally along a simulated enclave path — secrets come from `.env` |
| `cre secrets create` | Stores secrets in the Vault DON — **required before deploying** |
| `cre workflow deploy` | Compiles, uploads, and registers the workflow on a DON |
| Private registry | Chainlink-hosted registry authorized by your CRE login — no wallet or gas |
| Vault DON | Releases secrets directly into the attested enclave at execution time |

## What's Next

You've just run a Confidential Workflow — but why does it need an enclave, and what exactly stays confidential? Let's build that mental model properly: **Confidential Workflows: Why Confidential Computing**.
