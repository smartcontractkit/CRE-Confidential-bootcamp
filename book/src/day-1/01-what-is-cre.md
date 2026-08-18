# What Is CRE

Before writing any code, let's build a clear mental model of what CRE is and how it works.

## What Is CRE?

**Chainlink Runtime Environment (CRE)** is an orchestration layer that lets you write and run your own workflows in TypeScript or Go, powered by Chainlink Decentralized Oracle Networks (DONs). With CRE, you can compose different capabilities (such as HTTP, onchain reads/writes, signing, and consensus) into verifiable workflows that connect smart contracts to APIs, cloud services, AI systems, other blockchains, and more. These workflows execute on DONs with built-in consensus, acting as a secure, tamper-resistant, and highly available runtime.

### The Problem CRE Solves

Smart contracts have a fundamental limitation: **they can only see data on their own chain**.

- ❌ Cannot fetch data from external APIs (exchange balances, risk signals)
- ❌ Cannot call AI models (LLM audits, policy reasoning)
- ❌ Cannot protect privacy (onchain data and execution are visible to everyone)

CRE bridges this gap by providing a **verifiable runtime** where you can:

- ✅ Fetch data from any API (including private data behind API keys)
- ✅ Call AI services for reasoning and decision-making
- ✅ Write verified results back onchain
- ✅ Process sensitive data in a **confidential execution environment** (the subject of this bootcamp 🔒)

...with **cryptographic consensus** guaranteeing that every step is verified.

## Core Concepts

### 1. Workflow

A **Workflow** is the offchain code you develop, written in TypeScript or Go. CRE compiles it to WebAssembly (WASM) and runs it across a Decentralized Oracle Network (DON).

```typescript
// A workflow is just TypeScript code!
const initWorkflow = (config: Config) => {
  return [
    cre.handler(trigger, callback),
  ]
}
```

### 2. Trigger

A **Trigger** is the event that starts a workflow. CRE supports three types:

| Trigger | When it fires | Example |
|---------|---------------|---------|
| **CRON** | On a schedule | "Check position health every 5 minutes" |
| **HTTP** | When an HTTP request arrives | "Start an audit when the API is called" |
| **Log** | When a smart contract emits an event | "Settle when SettlementRequested fires" |

> Both case studies in this bootcamp use the **CRON Trigger** — the most common trigger for automated monitoring scenarios.

### 3. Capability

A **Capability** is what a workflow **can do** — a microservice that performs a specific task:

| Capability | What it does |
|------------|--------------|
| **HTTP** | Make HTTP requests to external APIs |
| **EVM Read** | Read data from smart contracts |
| **EVM Write** | Write data to smart contracts |
| **Confidential HTTP** | Make HTTP requests from inside a confidential execution environment (URL, headers, and response body are confidential from node operators) |

Each capability runs on its own dedicated DON with built-in consensus.

### 4. Decentralized Oracle Network (DON)

A **DON** is a network of independent nodes that:

1. Independently execute your workflow
2. Compare their results
3. Reach consensus using a Byzantine Fault Tolerant (BFT) protocol
4. Return a single, verified result

## The Trigger-and-Callback Pattern

This is the core architectural pattern you will use in every CRE workflow:

```typescript
cre.handler(
  trigger,    // WHEN to execute (cron, http, log)
  callback    // WHAT to execute (your business logic)
)
```

Each trigger fire starts a **fresh, independent, stateless execution**: the callback runs, does its work, returns a result, and completes. Inside the callback, you invoke capabilities through SDK clients; each call is asynchronous and returns a consensus-verified result.

## Execution Flow

When a trigger fires, here's what happens:

```
1. Trigger fires (cron schedule, HTTP request, or on-chain event)
            │
            ▼
2. Workflow DON receives the trigger
            │
            ▼
3. Each node executes your callback independently
            │
            ▼
4. When callback invokes a capability (HTTP, EVM Read, etc.):
            │
            ▼
5. Capability DON performs the operation
            │
            ▼
6. Nodes compare results via BFT consensus
            │
            ▼
7. Single verified result returned to your callback
            │
            ▼
8. Callback continues with trusted data
```

## From "Verifiable" to "Confidential"

The model above is already powerful, but it carries an implicit assumption: **your workflow's code and data run on DON nodes, and node operators can, in principle, inspect what is being computed**.

For most applications that's fine. But if your workflow handles:

- 🔑 High-value credentials (exchange API keys with trading/withdrawal permissions)
- 📊 Risk thresholds and strategy parameters that must not be made public
- 🤖 Proprietary scoring models and the data they reason over

...then you need a **Confidential Workflow**. That's the topic of the next section — and the core of this bootcamp.

## Key Takeaways

| Concept | One-liner |
|---------|-----------|
| **Workflow** | Your automation logic, compiled to WASM |
| **Trigger** | The event that starts execution (CRON, HTTP, Log) |
| **Callback** | The function containing your business logic |
| **Capability** | A microservice performing a specific task (HTTP, EVM Read/Write, Confidential HTTP) |
| **DON** | A set of network nodes executing under consensus |
| **Consensus** | The BFT protocol guaranteeing verified results |

## What's Next

Now that you understand the fundamentals of CRE, let's see how CRE achieves confidential computing through TEEs!
