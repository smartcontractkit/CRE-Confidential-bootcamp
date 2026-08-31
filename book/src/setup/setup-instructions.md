# 🖥️ Environment Setup

Please complete the following steps **before** the bootcamp to ensure a smooth learning experience.

## This Tutorial

This tutorial is available at:

```bash
https://smartcontractkit.github.io/CRE-Confidential-bootcamp/
```

## Important Prerequisites

To get the most out of this bootcamp, we recommend preparing the following environment **before you start**. Some items will be briefly covered in class so we can spend more time on hands-on work.

### Required

- **Node.js v20 or later** - [Download here](https://nodejs.org/)
- **Bun v1.3 or later** - [Download here](https://bun.sh/docs/installation)
- **CRE CLI** - [Installation guide](https://docs.chain.link/cre/getting-started/cli-installation)
- **CRE account** - Sign up at [cre.chain.link](https://cre.chain.link) and complete `cre login` (see [CRE CLI Quick Setup](./cre-cli-setup.md))
- **Deployment access** - Fill in [this form](https://docs.google.com/forms/d/e/1FAIpQLSdk8mxDZAXpEX1PHgjzCoBeKxSoQysoO9sxOb-gpBrDrjOhtA/viewform) with your own information to get deployment access
- **Git** - [Download here](https://git-scm.com/downloads)

### Optional (only needed for the onchain write exercise)

- **Add the Ethereum Sepolia network to your wallet** - [Add it here](https://chainlist.org/chain/11155111)
- **Get Sepolia ETH from a faucet** - [Chainlink Faucet](https://faucets.chain.link/sepolia)

> **Note**: Both case studies serve deterministic API data from a local mock server, so you can complete every demo **without** a real LLM API key.

### Recommended

- 📚 **Install mdBook** - to build and read this book locally
  ```bash
  cargo install mdbook
  ```

## Reference Code Repository

The two case studies used in this bootcamp come from the official CRE templates repo. We will clone and walk through them together in class:

```bash
git clone https://github.com/smartcontractkit/cre-templates.git
```

The two case studies live at:

| Case Study | Path |
|-----------|------|
| AI Audit Firewall | `cre-templates/starter-templates/confidential-workflows/ai-audit-firewall` |
| Automated Liquidation Protection | `cre-templates/starter-templates/confidential-workflows/automated-liquidation-protection` |

> **Note**: You don't need to read the code ahead of time! We will walk through the key parts line by line during the bootcamp. Just clone the repo in advance.
