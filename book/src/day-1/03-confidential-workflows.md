# Confidential Workflows: Why Confidential Computing

This section answers three questions: 
- What is a Confidential Workflow? 
- Why do you need one? 
- How Chainlink implements it?

## The Problem It Solves

By default, your workflow's code — along with any secrets or sensitive inputs it processes — runs on Workflow DON nodes, where node operators can, in principle, inspect what it's computing. That's fine for most workflows.

But some computation is sensitive on its own:

- 🎯 **Risk thresholds and strategies**: if the parameters for when to add collateral or rebalance leak, they can be predicted, front-run, and deliberately exploited
- 🔑 **High-value credentials**: exchange API keys with trading or withdrawal permissions, LLM API keys with spending limits, payment network credentials — leaking these is far worse than leaking an ordinary read-only key
- 🧠 **Proprietary models and reasoning**: the data processed by scoring models, audit criteria, or decision logic you don't want third parties to see

**Confidential Workflows** close this gap: sensitive computation executes inside a **secure enclave**, designed so that what is actually being computed remains confidential from node operators during execution.

## What Is a Confidential Workflow?

A **Confidential Workflow** is a CRE workflow that designates part of its logic to run inside a running instance of a **TEE (Trusted Execution Environment)** — an **enclave** — instead of on Workflow DON nodes.

> A **TEE** is a hardware-isolated execution environment designed so that even the machine's own operator cannot inspect the computation and data it processes during execution. CRE currently supports **AWS Nitro Enclaves**.

The key point: **a Confidential Workflow is still fundamentally a standard CRE workflow**, with an explicit confidential execution path added where you need it. It's fully compatible with the trigger / callback / capability model you already know, and it's built, deployed, and operated the same way — you decide what stays inside the enclave and what crosses back out to the Workflow DON for consensus-verified execution (such as generating a signed report to submit onchain).

## How Chainlink Implements Confidential Execution

In CRE, running a workflow confidentially means moving execution of the sensitive part into an enclave, and giving your code a runtime built for that environment (the `TeeRuntime`):

```
1. Declare a confidential handler
   Register the handler that should run inside a TEE with
   handlerInTee (TS) / cre.HandlerInTee (Go), specifying which
   TEE types/regions your workflow accepts
            │
            ▼
2. Trigger fires
   The Workflow DON hands the triggered request to an enclave
   instead of executing the callback locally
            │
            ▼
3. Enclave execution
   Your callback runs inside the enclave, receiving a
   TeeRuntime instead of the regular DON runtime
            │
            ▼
4. Dynamic secret fetch
   Secrets are requested and decrypted inside the enclave at
   the moment your code needs them — released by the Vault DON
   directly into the attested enclave
            │
            ▼
5. In-enclave capability calls
   HTTP and other supported calls execute directly from inside
   the enclave — URLs, headers, and response bodies stay
   confidential from node operators; trust comes from enclave
   attestation rather than DON-level consensus
            │
            ▼
6. Crossing back to the DON (optional)
   For anything that needs Workflow DON consensus — like
   generating a signed report via runtime.report() — you
   explicitly cross back out with runtime.usingTheDons()
            │
            ▼
7. Execution completes
   DON consensus verifies the enclave's attestation, proving
   the integrity of the workflow logic that executed within it
```

In code, there are only two core changes:

```typescript
import { handlerInTee, type TeeRuntime } from "@chainlink/cre-sdk";

// The callback receives a TeeRuntime
const onCronTrigger = async (runtime: TeeRuntime<Config>): Promise<string> => {
  // Fetch secrets inside the enclave — released by the Vault DON directly into the enclave
  const apiKey = runtime.getSecret({ id: "exchange_api_key" }).result().value;
  // Make confidential HTTP calls from inside the enclave ...
  return "done";
};

const initWorkflow = (config: Config): Workflow<Config> => {
  const cron = new CronCapability();
  return [
    handlerInTee(                          // ← instead of cre.handler
      cron.trigger({ schedule: config.schedule }),
      onCronTrigger,
      [{ tee: "nitro", regions: ["us-west-2"] }],  // ← TEE type + regions
    ),
  ];
};
```

## The Confidentiality Boundary: What's Protected and What Isn't

Understanding the confidentiality boundary is critical to designing your workflow correctly:

| ✅ Protected by default | ❌ Not automatically protected |
|------------------------|-------------------------------|
| Secrets the Vault DON releases into the enclave | Workflow triggers, chain reads, and chain writes — these always execute on Workflow DON nodes, never inside the enclave |
| Sensitive inputs and intermediate values you don't explicitly share outside the enclave | Your workflow's source code, deployed binary, and orchestration metadata |
| Capability calls made from inside the enclave (URL / headers / request & response bodies) | Capability requests and responses that aren't routed through the enclave |
| Enclave execution memory (as long as your computation runs inside it) | Reports, transaction calldata, and any output you deliver outside the enclave boundary |

> ⚠️ **Workflow logic is not confidential.** Your handler's source code and compiled binary are not confidential just because part of its logic runs inside an enclave. Confidential Workflows protect only the **data processed during execution inside the enclave** including Vault DON secrets (such as API keys), sensitive HTTP response payloads, and intermediate values not explicitly shared outside the enclave.

## When Do Secrets Need Enclave-Level Protection?

Not every secret needs enclave-level protection. A simple rule of thumb: **if disclosure would expose more than the workflow needs, or data that will never be made public onchain, it's a candidate for enclave execution.**

| High-value secrets — put them in the enclave | Regular DON execution is usually fine |
|----------------------------------------------|---------------------------------------|
| Exchange credentials with trading/withdrawal access, institutional custody credentials | API keys for publicly available data (weather, block explorers, public market data) |
| OAuth client secrets, KMS keys, payment processor, banking, and payment network credentials | Public wallet addresses |
| LLM API keys with spending limits, proprietary data provider credentials | Other credentials with similarly limited impact if disclosed |

## Typical Use Cases

Confidential Workflows apply anywhere the computation behind a decision — not just a workflow's API calls — needs to remain confidential from node operators:

| Use case | What confidentiality buys you |
|----------|-------------------------------|
| **AI smart contract audit firewall** (Day 1 case study) | Audit criteria and third-party API credentials stay inside the enclave |
| **Automated liquidation protection** (Day 2 case study) | Risk thresholds and the defensive strategy run inside the enclave, designed to prevent them from being predicted and front-run |
| Automated portfolio rebalancing | Allocation policy and trade sizing stay confidential, so the rebalance can't be anticipated and traded against |
| Automated trading | A private strategy can't be copied or traded against, even though the resulting transactions are onchain |
| Automated payment orchestration | Routing logic and account details stay confidential from node operators |
| Proprietary data computation | Both the data you can't expose and the computation over it stay confidential |

## Key Takeaways

| Concept | One-liner |
|---------|-----------|
| **TEE / Enclave** | A hardware-isolated execution environment — even the machine's operator can't see the computation inside |
| **Confidential Workflow** | A standard CRE workflow that designates part of its logic to run inside an enclave |
| **TeeRuntime** | The runtime given to callbacks executing inside the enclave |
| **Vault DON** | The DON that releases secrets directly into the attested enclave |
| **Attestation** | Proof that the enclave is running the expected workflow logic, verified by DON consensus |
| **`handlerInTee`** | The API for registering a confidential handler (TS; `cre.HandlerInTee` in Go) |
| **`runtime.usingTheDons()`** | Explicitly cross from the enclave back to the DON runtime for consensus operations |

> 📌 **Availability**: Confidential Workflows are in **Private Beta** — deployment requires enrollment. But **local simulation with the CRE CLI is fully open**, and every demo in this bootcamp runs in local simulation.

## What's Next

Enough theory — let's look at our first real case study: the **AI Audit Firewall**, a pre-execution security firewall that uses confidential computing to protect its audit credentials and process.
