# 02 · Capacity Planning — Sizing for 20 / 50 / 100 Developers

## 1. Workload model: what developers actually generate

Developer LLM traffic is **bursty and prompt-heavy**. Plan with these empirically-reasonable
baselines (validate with your own pilot — see §6):

| Parameter | Chat/completions usage | Agentic usage (Copilot agent mode, Cline, Aider) |
|---|---|---|
| Requests per active dev per hour | 10–30 | 60–300 (agents loop) |
| Mean input tokens / request | 1–4K | 8–40K (files + tool output) |
| Mean output tokens / request | 200–800 | 300–1,500 |
| Peak concurrency factor | 10–15% of team | 20–30% of team |

**Concurrency estimate** (the number that sizes your cluster):

$$C_{peak} = N_{devs} \times f_{active} \times \frac{\bar{t}_{request}}{\bar{t}_{think}}$$

In practice, for mixed chat + agent workloads, use the simplified planning values:

| Team size | Expected peak concurrent requests | Sustained aggregate throughput needed |
|---|---|---|
| 20 devs | 4–8 in flight | 1,000–3,000 output tok/s |
| 50 devs | 10–20 in flight | 3,000–8,000 output tok/s |
| 100 devs | 20–45 in flight | 6,000–15,000 output tok/s |

> Agent-heavy teams should use the top of each range. A single agentic task can hold 3–5
> requests in flight and consume 500K+ tokens end-to-end.

## 2. The 3-tier model catalog

Serve **three models**, route by task (the gateway does this — [06-gateway-and-devex.md](06-gateway-and-devex.md)):

| Tier | Role | Recommended model | Weights (quantized) | Traffic share |
|---|---|---|---|---|
| **Fast** | Completions, commit msgs, quick Q&A | Qwen2.5-Coder-7B (AWQ/FP8) | ~5–8 GB | 30–50% of requests, 10% of tokens |
| **Default** | Chat, code gen, agentic dev | Qwen3-Coder-30B-A3B (FP8) | ~20–32 GB | 40–60% of requests, 60% of tokens |
| **Reasoning** | Hard debugging, architecture | gpt-oss-120B / DeepSeek-R1-Distill-32B / Qwen3-235B | 20–70 GB | 5–10% of requests, 30% of tokens |

Why MoE for the default tier: Qwen3-Coder-30B-A3B activates only ~3B parameters/token →
decode speed of a small model with near-32B quality; ideal throughput-per-GPU for the
highest-volume tier.

## 3. GPU memory budget

$$\text{VRAM} = W_{weights} + KV_{cache} + O_{overhead(\sim 2-4GB)}$$

KV cache per token (FP16, no MLA): $2 \times n_{layers} \times n_{kv\_heads} \times d_{head} \times 2\,\text{bytes}$.
Practical planning numbers **per 1,000 tokens of context, per sequence**:

| Model | KV cache / 1K tokens |
|---|---|
| 7B dense (GQA) | ~65 MB |
| 30B-A3B MoE | ~100 MB |
| 32B dense | ~160 MB |
| 70B dense | ~330 MB |

Worked example — Qwen3-Coder-30B FP8 on one A100/H100 80GB:
weights ≈ 30 GB, overhead ≈ 4 GB → **~46 GB left for KV** → ~460K total cached tokens →
e.g. **14 concurrent sequences at 32K context**, or 28 at 16K. This is why context length
discipline matters more than parameter count for concurrency.

## 4. Reference hardware configurations

### 20 developers

| Option | Hardware | Serves | Notes |
|---|---|---|---|
| On-prem entry | 1 node · 2× L40S 48GB | Fast (1 GPU) + Default (1 GPU) | ~USD 25–35K node; reasoning → Foundry serverless |
| On-prem better | 1 node · 1× H100 80GB + 1× L40S | Default on H100, Fast on L40S | Headroom for 32K agent contexts |
| AKS | 1× `Standard_NC24ads_A100_v4` (A100 80GB) + scale-to-zero second node | Default + Fast co-hosted or split | Reserve if steady |
| Foundry only | — | Serverless gpt-oss / Llama / DeepSeek | Viable if < ~50M tokens/day |

### 50 developers

| Option | Hardware | Layout |
|---|---|---|
| On-prem | 1–2 nodes · 4× L40S or 2× H100 | Default ×2 replicas (HA), Fast ×1, Reasoning ×1 (R1-32B) |
| AKS | 2× NC24ads A100 v4 (base, reserved) + 1 spot node (burst) | Default ×2, Fast ×1 co-hosted; KAITO or vLLM |
| Hybrid | 2 dedicated GPUs + Foundry serverless for reasoning tier | Most cost-efficient at this size |

### 100 developers

| Option | Hardware | Layout |
|---|---|---|
| On-prem | 2× nodes · 4× H100 80GB each (NVLink) | Default ×3–4 replicas, Fast ×2, Reasoning: gpt-oss-120B on 2× H100 TP=2 |
| AKS | 4× NC24ads A100 v4 reserved + 2–4 spot A100/A10 burst + cluster autoscaler | Same layout; HPA on queue depth |
| Hybrid | 4–6 dedicated GPUs + serverless overflow routing at p95 queue depth | Recommended |

**Sizing heuristic: one 80 GB-class GPU per 20–25 developers** for mixed workloads on an
efficient MoE default model; halve that density for agent-heavy teams (one GPU per 10–12 devs).

## 5. Availability math

- N+1 replicas for the default tier at 50+ devs (a GPU node reboot otherwise = full outage).
- Model loading takes 1–10 min (weights pull + engine warmup) — treat replicas as slow-start;
  keep weights on fast local NVMe or pre-baked images to cut this (see track docs).
- Degradation path: default tier down → gateway fails over to fast tier + banner, not hard 503.

## 6. Pilot before purchase (mandatory for Track A)

Run 2 weeks on cloud GPUs before buying hardware:

1. Deploy Track B minimal (1 GPU node, default model, gateway).
2. Onboard 5–10 representative devs (mix of chat and agent users).
3. Capture from the gateway per request: input/output tokens, model, latency, team.
4. Extract: requests/dev/day (p50/p95), token distribution, peak-hour concurrency.
5. Scale linearly to full team size, add 40% headroom, then size hardware.

```sql
-- LiteLLM spend logs → planning numbers
SELECT date_trunc('hour', "startTime") AS hr,
       count(*) AS requests,
       sum(prompt_tokens) AS in_tok,
       sum(completion_tokens) AS out_tok,
       count(DISTINCT "user") AS active_devs
FROM "LiteLLM_SpendLogs"
GROUP BY 1 ORDER BY out_tok DESC LIMIT 24;
```
