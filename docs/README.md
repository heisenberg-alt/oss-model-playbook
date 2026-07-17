# Self-Hosted Open-Weight LLM Platform — Enterprise Implementation Playbook

End-to-end documentation for standing up an enterprise-grade, open-weight LLM inference
platform for developer teams of **20, 50, and 100+ developers**, with three deployment tracks:

| Track | Where | Doc |
|---|---|---|
| **A** | On-premises Kubernetes (your GPUs, your datacenter) | [03-onprem-kubernetes.md](03-onprem-kubernetes.md) |
| **B** | Azure Kubernetes Service (AKS + KAITO or self-managed vLLM) | [04-aks-azure.md](04-aks-azure.md) |
| **C** | Azure AI Foundry (managed compute / serverless, zero cluster ops) | [05-foundry.md](05-foundry.md) |

## Reading order

1. [01-decision-framework.md](01-decision-framework.md) — architecture options, pros/cons matrix, decision tree. **Start here.**
2. [02-capacity-planning.md](02-capacity-planning.md) — concurrency math, model selection, GPU sizing for 20/50/100 devs.
3. Pick your track: [03](03-onprem-kubernetes.md) / [04](04-aks-azure.md) / [05](05-foundry.md).
4. [06-gateway-and-devex.md](06-gateway-and-devex.md) — the unified OpenAI-compatible gateway, per-team keys/budgets, IDE integration (VS Code / Copilot BYOM, JetBrains, CLI agents). **Applies to all tracks.**
5. [07-performance.md](07-performance.md) — metric definitions, expected numbers, benchmarking methodology, tuning levers.
6. [08-operations.md](08-operations.md) — observability, SLOs, alerting, runbooks, upgrade strategy.
7. [09-security-governance.md](09-security-governance.md) — network, identity, data privacy, model license compliance.
8. [10-cost-tco.md](10-cost-tco.md) — 3-year TCO comparison and break-even analysis.

## The one-paragraph summary

For 20+ concurrent developers, **Ollama is a workstation tool, not a serving platform** — use
**vLLM** (or KAITO, which wraps vLLM) as the cluster-serving engine, front it with a
**LiteLLM gateway** that issues per-team virtual keys with budgets, run
**Qwen3-Coder-30B-A3B (MoE)** as the default model plus **a small fast model** for
completions and **a reasoning model** (DeepSeek-R1-Distill / gpt-oss) for escalation. Size
roughly **one 80 GB-class GPU per 20–25 active developers** for mixed chat+agent workloads,
autoscale on queue depth, and measure everything in **TTFT / TPOT / goodput**, not vibes.

## Companion artifact

[oss-model-picker.html](../oss-model-picker.html) — interactive model picker (task × hardware tier) for the model catalog referenced throughout these docs.

## Verification disclaimer

Model names, VM SKUs, prices, and operator versions in these docs were checked against public
documentation as of **July 2026** (KAITO add-on v0.6.0, Foundry classic deployment options).
GPU pricing and model availability drift monthly — re-verify against the linked sources before
committing budget.
