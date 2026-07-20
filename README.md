# Open-Weight LLM Platform — Enterprise Implementation Playbook

[![GitHub stars](https://img.shields.io/github/stars/heisenberg-alt/oss-model-playbook?style=flat&logo=github)](https://github.com/heisenberg-alt/oss-model-playbook/stargazers)
[![Last commit](https://img.shields.io/github/last-commit/heisenberg-alt/oss-model-playbook)](https://github.com/heisenberg-alt/oss-model-playbook/commits/main)

End-to-end documentation for standing up an enterprise-grade, open-weight LLM inference
platform for developer teams of **20, 50, and 100+ developers** — on your own infrastructure
or on Azure.

## Deployment tracks

| Track | Where | Guide |
|---|---|---|
| **A** | On-premises Kubernetes (your GPUs, your datacenter) | [docs/03-onprem-kubernetes.md](docs/03-onprem-kubernetes.md) |
| **B** | Azure Kubernetes Service (KAITO add-on or self-managed vLLM) | [docs/04-aks-azure.md](docs/04-aks-azure.md) |
| **C** | Azure AI Foundry (serverless or managed compute, zero cluster ops) | [docs/05-foundry.md](docs/05-foundry.md) |

## Start here

1. [Decision framework](docs/01-decision-framework.md) — architecture options, pros/cons matrix, decision tree
2. [Capacity planning](docs/02-capacity-planning.md) — concurrency math, model selection, GPU sizing
3. Your track: [A](docs/03-onprem-kubernetes.md) · [B](docs/04-aks-azure.md) · [C](docs/05-foundry.md)
4. [Gateway and developer experience](docs/06-gateway-and-devex.md) — applies to all tracks

Full reading order and summary: [docs/README.md](docs/README.md)

## Companion artifacts

Live site: **[heisenberg-alt.github.io/oss-model-playbook](https://heisenberg-alt.github.io/oss-model-playbook/)**

| Artifact | Purpose |
|---|---|
| [Team Cookbook (PDF)](https://heisenberg-alt.github.io/oss-model-playbook/cookbook.pdf) | Eleven condensed, print-ready recipes — download and share with your team (source: [cookbook.html](cookbook.html)) |
| [Interactive model picker](https://heisenberg-alt.github.io/oss-model-playbook/oss-model-picker.html) | Task × hardware-tier model catalog, including the latest frontier releases (Kimi K3, GLM-5.2, DeepSeek-V4) |

## The one-paragraph summary

For 20+ concurrent developers, Ollama is a workstation tool, not a serving platform — use
**vLLM** (or KAITO, which wraps vLLM) as the cluster-serving engine, front it with a
**LiteLLM gateway** that issues per-team virtual keys with budgets, and serve a
**three-tier model catalog**: a fast model for completions, an efficient MoE coder as the
default, and a reasoning model for escalation. Size roughly one 80 GB-class GPU per 20–25
active developers, autoscale on queue depth, and measure everything in TTFT / TPOT / goodput.

## Verification disclaimer

Model names, VM SKUs, prices, and operator versions were checked against public documentation
as of **July 2026**. GPU pricing and model availability drift monthly — re-verify against the
linked primary sources before committing budget.

## Support this project

If this playbook saves your team time, please
[star the repository](https://github.com/heisenberg-alt/oss-model-playbook) — stars are how
other teams discover it.
