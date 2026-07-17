# 10 · Cost & TCO — 3-Year Comparison and Break-Even

> All figures are **planning estimates in USD** at typical mid-2026 rates. GPU street prices,
> Azure PAYG/reservation rates, and per-token prices move constantly — rebuild this sheet with
> current quotes before any commitment. Percentages and *structure* age better than absolutes.

## 1. Unit economics of each track

### Track A · On-prem (CapEx + OpEx)

| Item | Estimate |
|---|---|
| 1× H100 80GB server slot (amortized incl. chassis/net share) | ~$30–40K CapEx → ~$0.9–1.2K/mo over 3 yr |
| 1× L40S 48GB slot | ~$10–13K CapEx → ~$0.3–0.4K/mo |
| Power+cooling per H100 (~1 kW loaded, PUE 1.4) | ~$100–200/mo |
| DC space, spares, support contracts | +15–25% on hardware |
| Platform engineering (fraction of 1–2 FTE) | $4–15K/mo depending on scale |

Effective cost of an H100 24×7: **~$1.3–1.9K/month** — versus ~$5–8K/month PAYG-equivalent in
cloud. On-prem wins big *if* the GPU stays busy and you already have DC + K8s competence.

### Track B · AKS

| Item | PAYG | 1-yr reserved | 3-yr reserved |
|---|---|---|---|
| NC24ads A100 v4 (1× A100 80GB) | ~$3.7–4.7/hr → $2.7–3.4K/mo | ~35% off | ~55–60% off → ~$1.3–1.7K/mo |
| Spot (same SKU) | often $1.1–1.9/hr | — | — |
| + AKS standard tier, LB, storage, egress | ~$100–300/mo overhead |
| Platform engineering | Lower than A (no hardware ops) |

3-yr-reserved AKS ≈ on-prem cost *without* the CapEx risk — the honest headline of this doc.

### Track C · Foundry

- **Serverless**: pay per token. Illustrative open-model rates: small models
  ~$0.1–0.5 / 1M output; 70B–class $0.7–3 / 1M; R1-class reasoning $2–8 / 1M output.
- **Managed compute**: same VM-hour economics as Track B (+ management premium, − cluster overhead).

## 2. Worked scenarios (default coder tier, mixed workload)

Token volumes from [02-capacity-planning.md](02-capacity-planning.md): avg dev ≈ 1–3M
in + 0.2–0.5M out tokens/day (agent-heavy: 3–10× that).

### 20 devs (~10M out-tok/day, light-moderate)

| Option | Monthly estimate | Notes |
|---|---|---|
| Foundry serverless only | $0.7–2.5K | Zero ops. **Winner at this volume** unless residency forbids |
| AKS 1× A100 (3-yr) + spot burst | $1.5–2.2K | Wins if usage grows or 20 devs are agent-heavy |
| On-prem 2× L40S | ~$1–1.5K + FTE fraction | Only if DC + K8s team already exist |

### 50 devs (~40–80M out-tok/day, agents mainstream)

| Option | Monthly estimate | Notes |
|---|---|---|
| Foundry serverless only | $4–20K (volatile) | Cost now scales with every agent loop |
| AKS 2× A100 reserved + spot + serverless reasoning | $3.5–5K | **Recommended** — hybrid flattens the curve |
| On-prem 2× H100 + serverless reasoning | $3–4.5K incl. FTE fraction | Wins at sustained high utilization |

### 100 devs (~100–250M out-tok/day)

| Option | Monthly estimate | Notes |
|---|---|---|
| Foundry serverless only | $10–60K | Linear forever; budget variance is the killer |
| AKS 4× A100 reserved + 2–4 spot + serverless overflow | $7–11K | **Recommended for most** |
| On-prem 8× H100 node + serverless overflow | $8–14K incl. ops | Wins ≥ ~60% sustained utilization + existing DC |

## 3. Break-even rules of thumb

1. **Serverless vs one dedicated A100 (3-yr reserved, ~$1.5K/mo):** dedicated wins above
   roughly **20–50M output tokens/month** on that GPU (rate-dependent). A 50-dev agentic team
   crosses this in the first week of the month.
2. **On-prem vs AKS reserved:** on-prem needs sustained utilization > ~50–60% *and* existing
   facilities/platform staffing; below that, reservation flexibility beats CapEx.
3. **Reasoning tier:** at ≤ 10% traffic share, serverless reasoning almost always beats owning
   2× H100 for it. Own the base load, rent the peaks and the frontier.

## 4. Comparison with per-seat SaaS

Sanity check: 100 devs × premium AI coding subscriptions ($20–60+/seat/mo, usage-capped)
= $2–6K+/mo with hard limits. The self-hosted platform at $7–11K/mo delivers *unmetered*
agentic usage + data control + model choice. The business case is strongest for **agent-heavy
teams hitting SaaS usage caps** and **code-privacy-constrained orgs** — for light chat-only
usage, SaaS or serverless remains cheaper. Run your own numbers with pilot data.

## 5. Cost governance checklist

- [ ] Reservations/savings plan on base GPU capacity (never PAYG steady-state)
- [ ] Spot for burst replicas; scale-to-zero off-hours (KEDA cron) where timezone allows
- [ ] Per-team budgets in the gateway; monthly showback report from spend logs
- [ ] Azure budget alerts on the RG at 80/100/120% of forecast
- [ ] Quarterly: re-quote serverless prices vs dedicated (per-token prices keep falling —
      the break-even moves every quarter)
- [ ] Track goodput/GPU ([07-performance.md](07-performance.md)) — tuning that lifts goodput 30%
      is worth one GPU in three
