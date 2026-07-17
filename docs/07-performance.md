# 07 · Performance — Metrics, Expected Numbers, Benchmarking, Tuning

## 1. The metrics that matter

| Metric | Definition | Dev-platform target |
|---|---|---|
| **TTFT** | Time to first token (queue + prefill) | Chat p95 < 1.5 s · completions p95 < 400 ms |
| **TPOT / ITL** | Time per output token after the first | p95 < 60 ms (≈ ≥ 17 tok/s per stream — faster than reading speed) |
| **E2E latency** | TTFT + TPOT × output_tokens | Informational; dominated by output length |
| **Throughput** | Aggregate output tok/s per GPU | Utilization/cost metric |
| **Goodput** | Throughput *while meeting* TTFT/TPOT SLOs | The real capacity number — use this for sizing |
| **Queue depth / time** | `vllm:num_requests_waiting`, time-in-queue | p95 queue time < 500 ms; alert > 2 s |
| **Preemption rate** | KV-cache evictions forcing recompute | ~0; > 1% means KV over-commit — lower `max-num-seqs` or context cap |

Load ↑ ⇒ throughput ↑ but TTFT/TPOT ↑. **Goodput** is where those curves cross your SLO —
that's the per-GPU capacity you plan with, not the max-throughput number from benchmarks.

## 2. Expected performance (planning baselines)

Realistic vLLM numbers, FP8/AWQ weights, moderate context (4–16K in / 0.5–1K out). Use as
**sanity ranges**, then benchmark your own stack (§3):

| Model | GPU | Single-stream decode | Aggregate throughput (batched) | Concurrent streams @ SLO |
|---|---|---|---|---|
| Qwen2.5-Coder-7B | 1× L40S 48GB | 60–100 tok/s | 1,500–3,000 tok/s | 30–60 |
| Qwen2.5-Coder-7B | 1× A100 80GB | 80–130 tok/s | 2,500–5,000 tok/s | 50–100 |
| Qwen3-Coder-30B-A3B (MoE) | 1× A100 80GB | 60–100 tok/s | 1,500–3,500 tok/s | 20–40 |
| Qwen3-Coder-30B-A3B | 1× H100 80GB | 90–140 tok/s | 2,500–5,500 tok/s | 30–60 |
| Qwen2.5-Coder-32B dense | 1× A100 80GB | 25–40 tok/s | 800–1,800 tok/s | 10–25 |
| Llama-3.3-70B (TP=2) | 2× A100 80GB | 20–35 tok/s | 700–1,500 tok/s | 8–20 |
| gpt-oss-120B (MXFP4) | 1× H100 80GB | 70–120 tok/s | 1,500–3,000 tok/s | 15–35 |

Notes: long agentic prompts (30K+) shift work to prefill — expect TTFT of 2–8 s uncached and
aggregate throughput to drop 30–50%; **prefix caching claws most of this back** for iterative
agent loops (same files re-sent each turn). Reasoning models multiply output tokens 5–20×,
which is why they get their own tier and budget.

Cross-check against [02-capacity-planning.md](02-capacity-planning.md): 50 devs needing
3–8K tok/s ≈ 2× A100 on the MoE default model — matches the reference configs.

## 3. Benchmarking methodology

Benchmark **your** stack — quant, context mix, and engine flags move numbers 2–5×.

```bash
# vLLM's built-in load generator against a live OpenAI endpoint
vllm bench serve \
  --backend openai-chat \
  --base-url https://llm.internal.example.com/v1 \
  --model default \
  --dataset-name random \
  --random-input-len 8192 --random-output-len 512 \
  --num-prompts 300 \
  --request-rate 4          # sweep: 1, 2, 4, 8, 16 → find the SLO knee
```

Method:
1. Fix a workload shape per tier (completions: 512 in/64 out; chat: 4K/512; agent: 16–32K/1K).
2. Sweep request rate; record TTFT p50/p95, TPOT p95, throughput at each step.
3. **Goodput = highest rate where p95 TTFT and TPOT still meet SLO.** That's the GPU's capacity.
4. Re-run after every engine upgrade, model swap, or flag change; keep results in the repo.

Alternatives: `genai-perf` (NVIDIA), `llmperf` (Ray). Same sweep logic.

## 4. Tuning levers, ordered by ROI

### 1 · Quantization (biggest single lever)
FP8 (Hopper/Ada) or AWQ/GPTQ-INT4: 2–4× less weight memory → more KV → more concurrency,
often +close-to-2× goodput, with small quality deltas for coding. Prefer official FP8 releases
(e.g. `Qwen/...-FP8`). Validate quality with a fixed eval set before/after (§5). Avoid
< 4-bit for the default tier.

### 2 · Prefix caching (`--enable-prefix-caching`)
Agent loops re-send system prompts + file context every turn. Cache hits skip prefill →
TTFT drops from seconds to ~100 ms on iterative turns. Practically free; enable everywhere.
Track `vllm:prefix_cache_hits` — healthy agent workloads reach 50–80% hit rate.

### 3 · Chunked prefill (`--enable-chunked-prefill`)
Prevents a single 100K-token prefill from stalling every in-flight decode. Mandatory when
agents (long prompts) and chat (latency-sensitive) share a replica.

### 4 · Context length caps (`--max-model-len`)
KV is the concurrency budget ([02-capacity-planning.md](02-capacity-planning.md) §3). Serve
64K instead of 256K on the default tier → 4× more concurrent sequences. Devs who truly need
256K get the reasoning tier.

### 5 · `--max-num-seqs` and GPU memory utilization
`--gpu-memory-utilization 0.90–0.93`; tune `max-num-seqs` so preemption rate ≈ 0. Too high ⇒
KV thrash (recomputes destroy TPOT); too low ⇒ idle GPU.

### 6 · Speculative decoding
Draft model or n-gram speculation: 1.5–2.5× single-stream decode speedup at low-moderate
batch sizes; benefit shrinks as batch grows. Best on the reasoning tier (long, low-concurrency
generations). Benchmark — it can *hurt* saturated throughput.

### 7 · Tier separation (architecture, not a flag)
Latency-critical completions on their own small-model replica; batch-friendly agent traffic on
the default tier. Mixing 100 ms-SLO and 10 s-SLO traffic on one replica forces you to tune for
the worst case of both.

### 8 · Advanced (100-dev+ scale)
- **Disaggregated prefill/decode** (vLLM production-stack, llm-d, NVIDIA Dynamo): separate
  prefill and decode fleets; smooths TPOT under long-prompt load.
- **KV-cache-aware routing** (llm-d / production-stack router): route a session to the replica
  already holding its prefix cache — big agent-loop win over naive least-busy.
- **MIG bin-packing** to reclaim stranded VRAM on the fast tier.

## 5. Quality guardrails while tuning

Performance tuning must not silently degrade output quality:

1. Maintain a ~100-prompt eval set from real team traffic (redacted) + a code benchmark
   (HumanEval+ / LiveCodeBench subset).
2. Run it on every: model swap, quant change, engine major upgrade.
3. Compare pass-rates and spot-check diffs; > 2–3 point regression ⇒ investigate quant/parser
   (tool-call parsing bugs masquerade as "model got dumber").
