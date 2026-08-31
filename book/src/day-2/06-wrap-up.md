# Wrap-Up: End-to-End Recap and Next Steps

## Two Days in Review

### Day 1: CRE + Confidential Workflow Fundamentals

- ✅ **CRE core concepts**: Workflows / Triggers / Capabilities / DONs / consensus
- ✅ **Confidential Workflows**: TEEs (AWS Nitro), enclaves, the Vault DON, attestation
- ✅ **The confidentiality boundary**: what's protected (secrets, in-enclave data, confidential HTTP) and what isn't (code, triggers, chain interactions)
- ✅ **Case Study 1: AI Audit Firewall** — dual-LLM confidential audit + conservative verdicts + optional onchain delivery

### Day 2: Automated Liquidation Protection Hands-On

- ✅ **Finance concepts**: overcollateralization, LTV, liquidation thresholds, health factors, liquidation mechanics and defenses
- ✅ **Case Study 2: Automated Liquidation Protection** — policy-as-secrets, LLM decisions + deterministic guardrails, defense execution

## Core Patterns Cheat Sheet

The Confidential Workflow development patterns distilled from these two case studies can be reused directly in your own projects:

| Pattern | API | Purpose |
|---------|-----|---------|
| Confidential handler | `handlerInTee(trigger, callback, [{ tee: "nitro", regions }])` | Run the callback inside an enclave |
| In-enclave secret fetch | `runtime.getSecret({ id })` | Secrets released by the Vault DON directly into the enclave |
| Confidential HTTP | `client.sendRequest(runtime, {...})` | URL / headers / body stay confidential from nodes |
| Capability restrictions | `preHook: (cfg) => buildRestrictions(cfg)` | Least-privilege allowlist (capability call counts, secret list) |
| Crossing back to the DON | `runtime.usingTheDons()` | Operations needing consensus (e.g., signed onchain reports) |
| Policy-as-secrets | Put thresholds/parameters in the Vault DON | Prevent your strategy from being predicted or front-run |
| Deterministic backstop | Validate LLM output with rules | Capital safety red lines don't depend on model reliability |

## Decision Framework: Does Your Project Need a Confidential Workflow?

```
Does your workflow handle any of the following?
├─ Credentials with trading / withdrawal / spending permissions?
├─ Risk thresholds, strategy parameters, or configs that must not be public?
├─ Sensitive data processed by proprietary models or scoring logic?
└─ Payment data, PII, or other compliance-sensitive information?
        │
   Any "yes" → use handlerInTee to move that part into an enclave
   All "no"   → a regular workflow with standard secrets is enough
```

## Keep Learning

### Go Deeper on Confidential Computing

- [Confidential Workflows concepts (official docs)](https://docs.chain.link/cre/concepts/confidential-workflows)
- [Making a Workflow Confidential (step-by-step guide)](https://docs.chain.link/cre/guides/workflow/using-confidential-workflows/making-workflow-confidential)
- [Confidential HTTP Client](https://docs.chain.link/cre/capabilities/confidential-http)
- [Secrets management](https://docs.chain.link/cre/guides/workflow/secrets)

### More Templates

- [Automated Portfolio Rebalancing](https://github.com/smartcontractkit/cre-templates/tree/main/starter-templates/confidential-workflows/automated-portfolio-rebalancing) — the third confidential template in this series, great self-study material
- [CRE Templates Hub](https://docs.chain.link/cre-templates) — all official templates

## Deploying to Production

Local simulation is fully open; deploying a Confidential Workflow to run on a DON requires:

1. Request **Early Access** deployment access: `cre account access` or visit [app.chain.link/cre/request-access](https://app.chain.link/cre/request-access)
2. Request **Confidential Workflows** access (Private Beta): see [Requesting Confidential Workflows Access](https://docs.chain.link/cre/account/confidential-workflows-access)
3. Follow the [deployment guide](https://docs.chain.link/cre/guides/operations/deploying-workflows)

> ⚠️ **Production reminder**: remove or gate debug logs inside enclave logic before deploying — anything logged from within a Confidential Workflow could leak the data the enclave is meant to protect.

## Stay Connected

- [Join the Chainlink Discord](https://discord.com/invite/chainlink) — the most active channel for technical Q&A
- [Subscribe to the developer newsletter](https://pages.chain.link/subscribe)
- [Follow Chainlink on X](https://x.com/chainlink)


## Congratulations!

🎉 **Congratulations on completing the CRE Confidential Bootcamp!** 🎉

We can't wait to see the confidential computing applications you build.
