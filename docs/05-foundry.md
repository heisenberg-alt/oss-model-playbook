# 05 · Track C — Azure AI Foundry (Managed Compute & Serverless)

The zero-cluster track. Foundry's model catalog deploys open-weight models three ways; two are
relevant for open weights:

| Mode | Billing | Models | Content filtering | Best for |
|---|---|---|---|---|
| **Standard / serverless API** | Per token (+ minimal endpoint infra/min for serverless) | Flagship open models sold via Azure: Llama, DeepSeek (R1/V3), Mistral, gpt-oss, Grok, Qwen (availability varies by region) | ✓ Built in | Spiky/low volume; reasoning escalation tier |
| **Managed compute** | Per compute-minute (dedicated GPU VMs) | Any catalog model incl. Hugging Face collection, NVIDIA NIMs, custom/fine-tuned | ✗ BYO guardrails | Sustained volume without cluster ops; custom models |

> Requires an AI Hub / Foundry project. Managed compute needs **dedicated VM quota**
> (separate from AKS quota) in your subscription for the chosen GPU SKU.

## 1. When Foundry beats running a cluster

- No platform team, or the platform team's roadmap is full.
- Usage below the break-even line (see [10-cost-tco.md](10-cost-tco.md)) — roughly: if your
  whole team consumes < ~30–60M tokens/day, serverless per-token usually costs less than one
  dedicated A100 running 24×7.
- You want built-in content filtering + Azure AI evaluation/tracing integration.
- Hybrid overflow: dedicated GPUs handle the base load; the gateway routes surplus/reasoning
  traffic to Foundry serverless — no idle frontier GPUs.

## 2. Serverless / standard deployment (C1)

Portal path: Foundry → Model catalog → pick model (e.g. `DeepSeek-R1`, `gpt-oss-120b`,
`Llama-3.3-70B-Instruct`) → Deploy → serverless/standard. CLI equivalent:

```bash
# Foundry resource + project (once)
az cognitiveservices account create -n llm-foundry -g $RG -l $LOC \
  --kind AIServices --sku S0

# Deploy a model from the catalog (names/versions: check the catalog)
az cognitiveservices account deployment create -n llm-foundry -g $RG \
  --deployment-name deepseek-r1 \
  --model-name DeepSeek-R1 --model-format DeepSeek --model-version "1" \
  --sku-name GlobalStandard --sku-capacity 1
```

Consume via the OpenAI-compatible endpoint:

```bash
curl https://llm-foundry.services.ai.azure.com/models/chat/completions?api-version=2024-05-01-preview \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"deepseek-r1","messages":[{"role":"user","content":"why does this deadlock?"}]}'
```

Notes:
- Prefer **Entra ID (keyless) auth** over API keys where supported.
- Data-zone/global vs regional processing options differ per model — pick regional if you have
  residency constraints.
- Quotas are tokens-per-minute based; request increases like any AOAI deployment.

## 3. Managed compute (C2)

Deploys the model onto dedicated GPU VMs that Azure operates (managed online endpoint under
the hood). Portal: Model catalog → model (e.g. a Hugging Face `Qwen/Qwen3-Coder-30B-A3B-Instruct`)
→ Deploy → Managed compute → pick instance type + count.

SDK example (Python, AzureML SDK v2 — managed compute rides on ML online endpoints):

```python
from azure.ai.ml import MLClient
from azure.ai.ml.entities import ManagedOnlineEndpoint, ManagedOnlineDeployment
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), SUB_ID, RG, PROJECT)

ml.online_endpoints.begin_create_or_update(
    ManagedOnlineEndpoint(name="qwen-coder-ep", auth_mode="key")).result()

ml.online_deployments.begin_create_or_update(ManagedOnlineDeployment(
    name="blue",
    endpoint_name="qwen-coder-ep",
    model="azureml://registries/HuggingFace/models/Qwen-Qwen3-Coder-30B-A3B-Instruct/labels/latest",
    instance_type="Standard_NC24ads_A100_v4",
    instance_count=1,
)).result()
```

Operational characteristics:
- **Autoscale**: instance-count scaling rules (CPU/GPU util or schedule); slower and coarser
  than K8s HPA — size for peak or accept queueing.
- **Blue/green**: multiple deployments behind one endpoint with traffic split — use for model
  version rollouts.
- **Billing stops** when you delete the deployment; schedule off-hours teardown via automation
  if usage is 9–5.
- **No content filter** on managed compute — put guardrails in the gateway.

## 4. Pros / cons recap vs Tracks A & B

**Pros**: no cluster, no GPU driver matrix, no engine upgrades; fastest compliance story
(Azure-managed, private networking supported); catalog breadth incl. NIM-optimized containers.

**Cons**: per-token cost curve (C1) grows linearly forever; C2 has cluster-track economics with
less tuning control (limited engine flags, no custom quant pipeline, no MIG bin-packing);
model catalog/regional gaps; endpoint cold starts on redeploy.

## 5. Recommended Foundry usage inside the reference architecture

Even for Track A/B shops, use Foundry serverless for:

1. **Reasoning tier** (5–10% of traffic) — R1-class quality without owning 2× H100.
2. **Burst overflow** — gateway routes to Foundry when local queue p95 exceeds SLO.
3. **Model evaluation** — trial a new model serverless for a week before pulling weights into
   the cluster.

All three are pure gateway config ([06-gateway-and-devex.md](06-gateway-and-devex.md) §4).
