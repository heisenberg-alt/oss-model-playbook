# 04 · Track B — Azure Kubernetes Service (AKS) Implementation

Two sub-paths, same cluster foundation:

- **B1 — KAITO (AI toolchain operator add-on)**: managed operator; one CRD deploys GPU nodes +
  vLLM + model. Least effort, preset-driven. *(Add-on currently ships KAITO v0.6.0; verify.)*
- **B2 — Self-managed vLLM**: your own GPU node pools + the manifests from
  [03-onprem-kubernetes.md](03-onprem-kubernetes.md) §5. Full engine control (custom quants,
  spec decode, exact versions).

Recommendation: **start B1** for speed; move hot paths to B2 when you need tuning surface.

## 1. Prerequisites

- Azure CLI ≥ 2.76, `kubectl`, subscription **GPU quota** in the target region for your VM
  family (NCads A100 v4 / NCads H100 v5 / NVads A10 v5). Request via Azure portal → Quotas →
  Compute; approval for A100/H100 families can take days. Check regional availability first.
- Suggested regions with broad GPU availability: eastus, eastus2, southcentralus, westeurope,
  swedencentral, australiaeast (verify per SKU).

```bash
export RG=llm-platform-rg LOC=eastus2 CLUSTER=llm-aks
az group create -n $RG -l $LOC
```

## 2. Cluster foundation (both sub-paths)

```bash
az aks create -g $RG -n $CLUSTER -l $LOC \
  --tier standard \
  --node-count 2 --node-vm-size Standard_D8ds_v5 \        # system/gateway pool (CPU)
  --network-plugin azure --network-plugin-mode overlay \
  --enable-managed-identity \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-azure-monitor-metrics \                        # managed Prometheus
  --enable-ai-toolchain-operator \                        # KAITO add-on (omit for pure B2)
  --generate-ssh-keys

az aks get-credentials -g $RG -n $CLUSTER
```

Private cluster + Private Link for enterprise: add `--enable-private-cluster` and front the
gateway with an internal load balancer; devs reach it over ExpressRoute/VPN/P2S.

## 3. Path B1 — KAITO workspaces

KAITO auto-provisions the right GPU node pool per workspace and runs the model on vLLM with an
OpenAI-compatible API. Deploy the default tier:

```yaml
# workspace-qwen3-coder.yaml — check kaito-project/kaito presets for exact supported
# model names/instance types in your add-on version
apiVersion: kaito.sh/v1beta1
kind: Workspace
metadata:
  name: ws-qwen-coder
  namespace: llm
resource:
  instanceType: "Standard_NC24ads_A100_v4"   # 1× A100 80GB
  count: 1
  labelSelector:
    matchLabels: { apps: qwen-coder }
inference:
  preset:
    name: qwen2.5-coder-32b-instruct         # pick from KAITO preset registry
```

```bash
kubectl create ns llm
kubectl apply -f workspace-qwen3-coder.yaml
kubectl get workspace -n llm -w     # node ready ≤ ~10 min, workspace ≤ ~20 min
kubectl get svc -n llm              # ClusterIP service, OpenAI-compatible /v1
```

Models not in the preset list (e.g. a specific FP8 quant) deploy via KAITO's **custom model**
template (inference with a HuggingFace model ID + your runtime image) — or drop to B2.

KAITO notes:
- Deleting a workspace does **not** delete its GPU node pool — delete it explicitly
  (`az aks nodepool delete`) to stop billing.
- Add-on constraints: Linux node pools only, NVIDIA instance types only, public regions.
- KAITO also supports fine-tuning workspaces (QLoRA) on the same cluster — useful later.

## 4. Path B2 — self-managed GPU node pools + vLLM

```bash
# Base pool: reserved/steady capacity
az aks nodepool add -g $RG --cluster-name $CLUSTER -n gpubase \
  --node-vm-size Standard_NC24ads_A100_v4 \
  --node-count 2 --min-count 1 --max-count 4 --enable-cluster-autoscaler \
  --node-taints nvidia.com/gpu=present:NoSchedule \
  --labels workload=llm-inference gpu=a100-80gb \
  --node-osdisk-type Ephemeral --node-osdisk-size 512    # fast local weight cache

# Burst pool: spot (evictable) for cheap overflow
az aks nodepool add -g $RG --cluster-name $CLUSTER -n gpuspot \
  --priority Spot --eviction-policy Delete --spot-max-price -1 \
  --node-vm-size Standard_NC24ads_A100_v4 \
  --node-count 0 --min-count 0 --max-count 4 --enable-cluster-autoscaler \
  --node-taints kubernetes.azure.com/scalesetpriority=spot:NoSchedule \
  --labels workload=llm-inference-burst
```

AKS GPU node pools include the NVIDIA device plugin/driver stack by default (or use
`--gpu-driver none` + your own GPU Operator install for driver-version control).

Then apply the vLLM Deployments/Services/HPA from [03-onprem-kubernetes.md](03-onprem-kubernetes.md) §5–6 with two Azure adjustments:

**Weights storage** — replace hostPath with Azure Blob (blobfuse2 CSI) or premium Azure Files
for the shared cache; still stage to ephemeral OS disk / local temp for load speed:

```yaml
volumes:
  - name: models
    persistentVolumeClaim: { claimName: models-blob-pvc }   # blobfuse2, ReadOnlyMany
```

**Spot handling** — burst replicas tolerate the spot taint and must be preemption-safe (the
gateway retries idempotent requests on another replica when a node is evicted):

```yaml
tolerations:
  - { key: kubernetes.azure.com/scalesetpriority, value: spot, effect: NoSchedule }
  - { key: nvidia.com/gpu, operator: Exists }
```

## 5. Autoscaling stack on AKS

| Layer | Mechanism | Signal |
|---|---|---|
| Pod | HPA (or KEDA) | `vllm:num_requests_waiting` avg > 4 |
| Node | Cluster autoscaler (per pool) | Unschedulable GPU pods |
| Cost floor | Scale base pool min=1, spot min=0 | Nights/weekends drop to 1 GPU |

KEDA (AKS add-on) with the Prometheus scaler is the cleanest way to drive pod scaling from
managed Prometheus:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: vllm-default-scaler, namespace: llm }
spec:
  scaleTargetRef: { name: vllm-qwen3-coder-30b }
  minReplicaCount: 2
  maxReplicaCount: 6
  cooldownPeriod: 600
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://<managed-prometheus-query-endpoint>
        query: sum(vllm:num_requests_waiting{model_name="qwen3-coder-30b"})
        threshold: "8"
```

New GPU node ready time is ~5–10 min + model load — autoscaling absorbs *sustained* growth,
not second-level spikes. Queue absorbs spikes; keep p95 queue time in your SLO ([08-operations.md](08-operations.md)).

## 6. Azure-native integrations

- **Identity**: gateway auth via Entra ID (OIDC) so devs use corp credentials; workload
  identity for pods accessing Blob/Key Vault (no secrets in manifests).
- **Secrets**: HF tokens / gateway master keys in Azure Key Vault via CSI Secrets Store driver.
- **Observability**: managed Prometheus + managed Grafana (`az grafana create`); import a vLLM
  dashboard and DCGM GPU dashboard ([08-operations.md](08-operations.md)).
- **Registry**: mirror vLLM images to ACR (`az acr import`) — avoids Docker Hub rate limits and
  gives CVE scanning via Defender.
- **Policy**: Azure Policy for AKS to enforce no-public-LoadBalancer, approved registries only.

## 7. Cost controls specific to AKS

1. **1-yr/3-yr reservations or savings plan** on the base GPU pool (30–60% off PAYG).
2. **Spot for burst** replicas only — never for the only replica of a tier.
3. **Scale-to-zero** the reasoning tier off-hours (KEDA cron scaler), route off-hours reasoning
   to Foundry serverless.
4. **Right-size**: if the fast tier's MIG slice sits idle, co-host it on the default GPU with
   MIG rather than a dedicated A10 node.
5. Budget alerts on the resource group (`az consumption budget`), tagged per team.

Detailed numbers: [10-cost-tco.md](10-cost-tco.md).
