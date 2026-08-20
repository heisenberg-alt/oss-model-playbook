# Playbook chapters — reading order

Chapter index for the [Open-Weight LLM Platform Playbook](../README.md). Overview, deployment
tracks, companion artifacts, and the verification disclaimer live in the repo root README.

1. [01-decision-framework.md](01-decision-framework.md) — architecture options, pros/cons matrix, decision tree. **Start here.**
2. [02-capacity-planning.md](02-capacity-planning.md) — concurrency math, model selection, GPU sizing for 20/50/100 devs.
3. Pick your track:
   - **A** · [03-onprem-kubernetes.md](03-onprem-kubernetes.md) — on-premises Kubernetes (your GPUs, your datacenter)
   - **B** · [04-aks-azure.md](04-aks-azure.md) — AKS with the KAITO add-on or self-managed vLLM
   - **C** · [05-foundry.md](05-foundry.md) — Azure AI Foundry serverless / managed compute, zero cluster ops
4. [06-gateway-and-devex.md](06-gateway-and-devex.md) — the unified OpenAI-compatible gateway, per-team keys/budgets, IDE integration (VS Code / Copilot BYOM, JetBrains, CLI agents). **Applies to all tracks.**
5. [07-performance.md](07-performance.md) — metric definitions, expected numbers, benchmarking methodology, tuning levers.
6. [08-operations.md](08-operations.md) — observability, SLOs, alerting, runbooks, upgrade strategy.
7. [09-security-governance.md](09-security-governance.md) — network, identity, data privacy, model license compliance.
8. [10-cost-tco.md](10-cost-tco.md) — 3-year TCO comparison and break-even analysis.
