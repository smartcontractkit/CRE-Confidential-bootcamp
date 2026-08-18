# CRE Confidential Bootcamp: Build Confidential Workflows

A hands-on 2-day bootcamp for building confidential computing workflows with [**Chainlink Runtime Environment (CRE)**](https://docs.chain.link/cre).

> **Disclaimer**: This tutorial represents an educational example to use a Chainlink system, product, or service and is provided to demonstrate how to interact with Chainlink's systems, products, and services to integrate them into your own. This template is provided "AS IS" and "AS AVAILABLE" without warranties of any kind, it has not been audited, and it may be missing key checks or error handling to make the usage of the system, product or service more clear. Do not use the code in this example in a production environment without completing your own audits and application of best practices. Neither Chainlink Labs, the Chainlink Foundation, nor Chainlink node operators are responsible for unintended outputs that are generated due to errors in code.

## 📚 Book

The bootcamp tutorial is available in a markdown book format.

Either visit:

**https://smartcontractkit.github.io/CRE-Confidential-bootcamp/**

Or run locally:

```bash
cargo install mdbook
cd book
mdbook serve --open
```

## Curriculum

**Day 1: CRE + Confidential Workflow Fundamentals**

- CRE core concepts (Workflows / Triggers / Capabilities / DONs)
- Confidential Workflows: TEEs, enclaves, the Vault DON, and the confidentiality boundary
- Case Study 1: [AI Audit Firewall](https://github.com/smartcontractkit/cre-templates/tree/main/starter-templates/confidential-workflows/ai-audit-firewall) — workflow flow, key code, and why confidentiality is necessary
- Case Study 1 demo (local mock server + `cre workflow simulate`)

**Day 2: Hands-On — Automated Liquidation Protection**

- Finance primer: what liquidation is, health factors, and how to prevent liquidation
- Case Study 2: [Automated Liquidation Protection](https://github.com/smartcontractkit/cre-templates/tree/main/starter-templates/confidential-workflows/automated-liquidation-protection) — policy-as-secrets, LLM decisions backed by deterministic guardrails
- Case Study 2 demo
- Wrap-up and next steps

## What You'll Learn

Two complete CRE Confidential Workflow case studies (each available in both TypeScript and Go):

- **AI Audit Firewall** — a confidential pre-execution security audit: two LLMs audit contracts inside an enclave, producing an ALLOW / DENY / MANUAL_REVIEW verdict that can be delivered onchain as a DON-signed report
- **Automated Liquidation Protection** — confidential automated defense before liquidation: 11 secrets (2 credentials + 9 policy parameters) released into the enclave, an LLM policy engine making decisions, and deterministic rules enforcing the guardrails

Core patterns you will master:

- `handlerInTee` — execute handlers inside a TEE (AWS Nitro Enclave)
- In-enclave `getSecret` — secrets released by the Vault DON directly into the attested enclave
- Confidential HTTP — URLs, headers, and response bodies kept confidential from node operators
- `runtime.usingTheDons()` — explicitly cross back to the DON for consensus operations (e.g., signed onchain reports)
- Policy-as-secrets — keep risk thresholds and execution preferences in the Vault DON so they can't be predicted or front-run

## Required Setup

Complete these **before** the bootcamp:

- [Node.js v20+](https://nodejs.org/)
- [Bun v1.3+](https://bun.sh/)
- [CRE CLI](https://docs.chain.link/cre/getting-started/cli-installation) (and complete `cre login`)
- [Git](https://git-scm.com/downloads)
- (Optional, only for the onchain write exercise) [Ethereum Sepolia in your wallet](https://chainlist.org/chain/11155111) + [Sepolia ETH from faucet](https://faucets.chain.link/)

> Both case studies serve deterministic data from a local mock server — **no real LLM API key is required**.

## Running the Examples

### 1. Clone the templates repository

Both case studies come from the official CRE templates repo:

```bash
git clone https://github.com/smartcontractkit/cre-templates.git
cd cre-templates/starter-templates/confidential-workflows
```

#### Template structure

```
confidential-workflows/
├── ai-audit-firewall/                    # Case Study 1 (Day 1)
│   ├── project.yaml                      # CRE project config
│   ├── secrets.yaml                      # Secret mappings (3 secrets)
│   ├── contracts/                        # Optional onchain consumer contract
│   ├── ai-audit-firewall-ts/             # TypeScript workflow
│   └── ai-audit-firewall-go/             # Go workflow
└── automated-liquidation-protection/     # Case Study 2 (Day 2)
    ├── project.yaml
    ├── secrets.yaml                      # Secret mappings (11 secrets)
    ├── automated-liquidation-protection-ts/
    └── automated-liquidation-protection-go/
```

### 2. Set up environment variables

Using Case Study 1 as the example (Case Study 2 works the same way):

```bash
cd ai-audit-firewall
cp .env.example .env   # Default values work out of the box for local simulation
```

### 3. Install dependencies and start the mock server

```bash
cd ai-audit-firewall-ts
bun install
bun run mock:server    # Keep this terminal running: Mock server running at http://127.0.0.1:8787
```

### 4. Simulate the workflow

In a second terminal:

```bash
cd cre-templates/starter-templates/confidential-workflows/ai-audit-firewall
cre workflow simulate ./ai-audit-firewall-ts --target=staging-settings
```

For Case Study 2, simply swap the path to `automated-liquidation-protection`:

```bash
cd cre-templates/starter-templates/confidential-workflows/automated-liquidation-protection
cp .env.example .env
cd automated-liquidation-protection-ts && bun install && bun run mock:server
# In a second terminal:
cre workflow simulate ./automated-liquidation-protection-ts --target=staging-settings
```

Detailed steps, expected output, and hands-on experiments for each case study are in the demo chapters (the last section of Day 1 and Day 2 in the book).
