# 08 · Operations — Observability, SLOs, Alerting, Runbooks

## 1. Observability stack

| Layer | Source | Key series |
|---|---|---|
| Engine | vLLM `/metrics` (Prometheus) | `vllm:num_requests_running/waiting`, `vllm:time_to_first_token_seconds`, `vllm:time_per_output_token_seconds`, `vllm:gpu_cache_usage_perc`, `vllm:num_preemptions_total`, prefix-cache hits |
| GPU | DCGM exporter (GPU Operator / AKS add-on) | `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_XID_ERRORS`, power draw |
| Gateway | LiteLLM Prometheus callback | requests/tokens/latency/spend per model·team·key, fallback + retry counts |
| Platform | kube-state-metrics, node exporter | pod restarts, PVC usage, node pressure |

Stack: Prometheus + Grafana (on-prem: kube-prometheus-stack Helm chart; AKS: managed
Prometheus + managed Grafana). Import the community vLLM and DCGM dashboards, then add a
**per-team consumption** dashboard from gateway metrics.

## 2. SLOs (starting points)

| SLO | Target | Error budget policy |
|---|---|---|
| Gateway availability | 99.5% monthly (dev tool, not payments) | Freeze risky changes when budget burnt |
| Chat TTFT p95 | < 1.5 s | Page at > 3 s for 10 min |
| Completions TTFT p95 | < 400 ms | Ticket at > 800 ms sustained |
| TPOT p95 | < 60 ms | Ticket; page at > 120 ms |
| Queue time p95 | < 500 ms | Capacity review trigger at > 2 s daily peaks |

## 3. Alert rules (PromQL sketches)

```yaml
groups:
- name: llm-platform
  rules:
  - alert: LLMQueueSaturated
    expr: sum by (model_name) (vllm:num_requests_waiting) > 10
    for: 5m
    labels: { severity: warning }
  - alert: LLMTTFTBreach
    expr: histogram_quantile(0.95, sum by (le, model_name) (rate(vllm:time_to_first_token_seconds_bucket[5m]))) > 3
    for: 10m
    labels: { severity: critical }
  - alert: LLMPreemptionStorm       # KV thrash — see 07 §4.5
    expr: rate(vllm:num_preemptions_total[10m]) > 0.5
    for: 10m
  - alert: GPUXidErrors             # hardware fault
    expr: increase(DCGM_FI_DEV_XID_ERRORS[15m]) > 0
    labels: { severity: critical }
  - alert: GPUThermal
    expr: DCGM_FI_DEV_GPU_TEMP > 85
    for: 10m
  - alert: TeamBudgetNearlyExhausted
    expr: litellm_remaining_team_budget_metric < 0.1
```

## 4. Runbooks

### TTFT breach / queue saturation
1. Which tier? (`model_name` label). 2. GPU util ~100% + queue high ⇒ capacity: scale replicas
(HPA should have; check GPU quota/nodes), or temporarily route overflow → Foundry fallback.
3. GPU util *low* + TTFT high ⇒ engine issue: check preemptions, a giant-prompt user
(gateway logs, token histogram), or prefix-cache regression after an upgrade.

### Preemption storm
Lower `--max-num-seqs` or `--max-model-len`, or reduce `gpu-memory-utilization` by 0.02.
Long-term: more KV (bigger GPU / more replicas / tighter context caps).

### Spot node eviction (AKS)
Expected behavior: gateway retries route around it. Verify base pool absorbed load; check
eviction rate — if > a few/day in your region/SKU, switch burst pool SKU or region.

### Model replica crash-loop
`kubectl logs` — 90% of cases: OOM (KV overcommit after a flag change), weights path missing
(cache volume), or an incompatible engine flag after image bump. Roll back image tag first
(deployments pin tags), diagnose offline.

### Runaway agent loop (one user consuming the cluster)
Gateway per-user `rpm_limit`/`tpm_limit` should cap it. Identify via spend logs top-N; contact
user; if abusive pattern, lower that key's limits. This is a governance event, not an incident.

## 5. Change management

| Change | Procedure |
|---|---|
| Engine (vLLM) upgrade | Pin tags; upgrade one replica (canary) → benchmark sweep ([07](07-performance.md) §3) + eval set ([07](07-performance.md) §5) → roll fleet |
| Model version swap | Deploy new model as a new Deployment; gateway alias flips 10% → 50% → 100%; keep old replica hot for 48 h rollback |
| Driver/CUDA (on-prem) | GPU Operator rolling upgrade, one node at a time, PDBs on serving pods |
| K8s upgrade | Standard node-surge upgrades; GPU node images validated in staging first (driver ↔ kernel matrix) |
| Gateway config | Config is a ConfigMap in git; PR review; rolling restart (2 replicas ⇒ zero downtime) |

## 6. Capacity review cadence

Monthly, from gateway spend logs + goodput benchmarks:
1. Peak-hour concurrency & queue p95 trend — approaching goodput ceiling? Order/reserve GPUs
   *before* the SLO burns (on-prem lead time: weeks).
2. Tier mix drift — reasoning share creeping above ~10% means either legit need (add capacity)
   or lazy routing (tighten allow-lists).
3. Per-team consumption vs budget — rebalance quotas.
4. New model releases — quarterly candidate evaluation via Foundry serverless trial
   ([05-foundry.md](05-foundry.md) §5), promote through the alias if it wins.
