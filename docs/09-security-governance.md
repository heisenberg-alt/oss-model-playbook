# 09 · Security & Governance

## 1. Threat model (what's different about an LLM platform)

| Asset | Threats | Controls |
|---|---|---|
| Prompts/completions (contain proprietary source code) | Exfiltration via logs, third-party callbacks, over-broad observability | Metadata-only logging; no external callbacks; encrypt spend-log DB; retention ≤ 30–90 d |
| Model weights | License violations, tampered weights (supply chain) | Pull from official HF orgs only; pin revisions + verify checksums; internal mirror for air-gap |
| Serving endpoints | Unauthenticated access, SSRF from agent tool use | Gateway-only exposure; mTLS/TLS; NetworkPolicy deny-all; egress control on GPU pods |
| GPU capacity | Denial-of-wallet (runaway loops), noisy neighbor | Per-key RPM/TPM limits, budgets, priority classes |
| Outputs | Insecure generated code, prompt injection via retrieved content | SAST/code review unchanged (LLM output = untrusted input); agent tool allow-lists |

## 2. Identity & access

- **Devs → gateway**: virtual keys minimum; better, OIDC/Entra ID SSO in front of the gateway
  (LiteLLM supports JWT auth; on Azure, put the internal endpoint behind Entra-protected
  ingress or APIM). Keys scoped per team, auto-expiring, rotated quarterly.
- **Pods → cloud resources** (AKS): Workload Identity — no static credentials for Blob/Key Vault.
- **Admin plane**: K8s RBAC — platform team only in `llm` namespace; gateway admin UI behind
  SSO + allow-list.
- **Foundry**: prefer keyless (Entra) auth on standard deployments; managed-compute endpoint
  keys live in Key Vault, injected via CSI driver.

## 3. Network

- Private-only everywhere: internal load balancer / private cluster on AKS; ExpressRoute/VPN
  for remote devs; no public IPs on any LLM component.
- NetworkPolicy in `llm` namespace: default deny; allow gateway→engines:8000,
  Prometheus→:8000/metrics, gateway→Postgres, gateway→Foundry FQDN (egress via Private Link
  where available).
- Engines have **no reason to reach the internet** after weights are cached — block egress
  (`HF_HUB_OFFLINE=1`), which also kills a whole class of exfiltration paths.

## 4. Data handling policy (write this down, publish to the team)

1. Prompts and completions are **processed in memory, not persisted** by the platform
   (gateway logs metadata: tokens, latency, model, team — not content).
2. Exception: opt-in debug sampling with named approval, TTL ≤ 7 days.
3. Prefix/KV caches are in-GPU-memory only and evicted; no cross-tenant leakage (vLLM isolates
   sequences), but **do not** enable multi-tenant *cross-org* sharing of one replica if teams
   have data-separation requirements — separate replicas per boundary.
4. Foundry serverless traffic is processed by Azure per its data-privacy terms (not used for
   training); document that reasoning-tier traffic leaves the cluster and give teams a
   cluster-only opt-out (`reasoning-local` alias → R1-Distill-32B replica).

## 5. Model license compliance

| License | Models | Enterprise posture |
|---|---|---|
| Apache-2.0 | Qwen (most), Mistral/Devstral, gpt-oss | ✅ Use freely, attribute |
| MIT | DeepSeek (R1/V3), Phi, GLM-4.6 | ✅ Use freely |
| Modified MIT | Kimi K2 (attribution clause at very large scale) | ✅ with legal review of the clause |
| Llama Community | Llama 3.x | ⚠️ AUP + 700M-MAU clause; legal sign-off; redistribution rules |
| Gemma Terms | Gemma 3 | ⚠️ Use-restriction policy; legal sign-off |

Process: models enter the catalog via a lightweight review (license, provenance, eval results);
the gateway's `model_list` **is** the allow-list — anything not in it is unreachable.

## 6. Guardrails & responsible use

- Foundry standard/serverless includes content filtering; self-hosted and managed compute do
  not — if policy requires it, add a guardrail hook at the gateway (LiteLLM guardrails /
  Llama Guard sidecar / Azure AI Content Safety API) on the tiers that need it. For an
  internal dev-tools platform, most orgs run **without** output filtering but with acceptable-use
  policy + audit metadata.
- Agent tool use: the platform serves tokens; tool *execution* happens in the IDE/agent. Publish
  a tool allow-list baseline for team agent configs (no arbitrary shell on shared infra, no
  credentials in prompts).
- Audit: retain gateway metadata logs (who, which model, when, token counts) ≥ 1 year for
  usage attestation; this is cheap since content isn't stored.

## 7. Compliance quick-map

| Requirement | Track A | Track B | Track C |
|---|---|---|---|
| Data never leaves premises | ✅ | ❌ (leaves premises, stays in your tenant) | ❌ (same) |
| Data stays in-tenant / region | ✅ | ✅ (region-pinned) | ✅ regional deployments (verify per model) |
| CMK encryption at rest | Your storage layer | Azure disk/storage CMK | Supported on Foundry resources |
| SOC2/ISO evidence for infra | You produce it | Inherit Azure attestations + your workload | Mostly inherited |
