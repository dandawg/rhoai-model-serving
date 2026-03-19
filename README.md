# RHOAI Model Serving

Simplified model deployment and serving for Red Hat OpenShift AI (RHOAI). This repository provides hardware profiles, model download jobs, and model serving configurations for deploying AI models on GPU-enabled OpenShift clusters.

## Prerequisites

### Required Infrastructure

1. **OpenShift Cluster with GPUs**
   - OpenShift 4.19+ on AWS
   - See [openshift-infra](https://github.com/redhat-ai-americas/openshift-infra) for GPU MachineSets
   - AWS instance types: `g4dn.xlarge` (NVIDIA T4), `g6.4xlarge` (NVIDIA L4), or `g6e.2xlarge` (NVIDIA L40S)

2. **RHOAI Installed**
   - See [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) for RHOAI installation
   - Requires NVIDIA GPU Operator and RHOAI Operator installed
   - DataScienceCluster configured with model serving enabled

3. **GitOps (Optional but Recommended)**
   - OpenShift GitOps Operator installed
   - ArgoCD configured for automated deployments
   - Run `./bootstrap.sh` if not already installed

### Install OpenShift GitOps (if needed)

```bash
./bootstrap.sh
```

**Note:** If GitOps is already installed (e.g., from deploying another repository), the bootstrap script will detect it and skip installation.

## Quick Start

### 1. Deploy Hardware Profiles

Hardware profiles define GPU node constraints and resource limits for model serving.

```bash
# For NVIDIA T4 GPUs (g4dn.xlarge)
oc apply -f gitops/platform/hardware-profiles/g4dn-xlarge.yaml

# For NVIDIA L4 GPUs (g6.4xlarge)
oc apply -f gitops/platform/hardware-profiles/g6-4xlarge.yaml

# For NVIDIA L40S GPUs (g6e.2xlarge) - single GPU per replica
oc apply -f gitops/platform/hardware-profiles/g6e-2xlarge.yaml

# For NVIDIA L40S GPUs (g6e.12xlarge) - 4 GPUs, llm-d distributed inference
oc apply -f gitops/platform/hardware-profiles/g6e-12xlarge.yaml
```

### 2. Download a Model

Models are downloaded from HuggingFace and stored on PVC storage.

```bash
# Download Qwen3-VL-Embedding-2B (compact, good for T4)
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-pvc.yaml

# Monitor download progress
oc get jobs -n demo -w
oc logs -f job/download-qwen3-vl-embedding-2b -n demo
```

### 3. Deploy Model Serving

Once the model is downloaded, deploy the serving runtime.

```bash
# Deploy Qwen3-VL-Embedding-2B serving
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-serving.yaml

# Wait for InferenceService to be ready
oc wait --for=condition=Ready inferenceservice/qwen3-vl-embedding-2b \
  -n demo --timeout=600s
```

### 4. Test the Model

```bash
# Get the inference endpoint
oc get inferenceservice qwen3-vl-embedding-2b -n demo

# Test from within cluster
oc run curl-test --image=curlimages/curl -it --rm -n demo -- \
  curl -X POST http://qwen3-vl-embedding-2b-predictor.demo.svc.cluster.local/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen3-vl-embedding-2b", "prompt": "Hello, how are you?", "max_tokens": 50}'
```

## Available Models

| Model | Size | GPU | Use Case | Hardware Profile |
|-------|------|-----|----------|------------------|
| **Qwen3-VL-Embedding-2B** | 4.5GB | T4 | Embeddings, compact multimodal | g4dn-xlarge |
| **Qwen3-VL-4B** | 8.5GB | T4 | Vision-language tasks | g4dn-xlarge |
| **Qwen3-VL-8B** | 18GB | L4 | Advanced vision-language | g6-4xlarge |
| **Qwen3-VL-8B-FP8** | 9GB | L4 | Quantized vision-language | g6-4xlarge |
| **Qwen3.5-27B-FP8** | 30.9GB | L40S | Large language model, FP8 quant | g6e-2xlarge |
| **Granite-7B** | 14GB | T4/L4 | General instruction following | g4dn-xlarge |
| **Llama-3-8B** | 16GB | L4 | General purpose (gated) | g6-4xlarge |
| **Nemotron-3-Nano-30B-A3B-FP8** | 32.7GB | 4x L40S | Distributed MoE, llm-d TP=2 | g6e-12xlarge |

## Hardware Profiles

| Profile | Instance Type | GPU | vCPUs | Memory | Best For |
|---------|--------------|-----|-------|---------|----------|
| **g4dn-xlarge** | AWS g4dn.xlarge | NVIDIA T4 (16GB) | 4 | 16GB | Small models, embeddings |
| **g6-4xlarge** | AWS g6.4xlarge | NVIDIA L4 (24GB) | 16 | 64GB | Large models, vision tasks |
| **g6e-2xlarge** | AWS g6e.2xlarge | NVIDIA L40S (48GB) | 8 | 64GB | Large models (13B+), high GPU memory workloads |
| **g6e-12xlarge** | AWS g6e.12xlarge | 4x NVIDIA L40S (48GB each) | 48 | 192GB | Distributed inference with llm-d (TP=2, 2 replicas) |

## Model Deployment Workflow

### Step-by-Step Process

1. **Hardware Profile** → Define GPU node selection and tolerations
2. **Model Download** → Download model from HuggingFace to PVC
3. **Model Serving** → Deploy ServingRuntime and InferenceService

### GitOps Files

Each model has two GitOps files:

- `*-pvc.yaml` - Downloads the model to persistent storage
- `*-serving.yaml` - Deploys the model serving endpoint

Some models (e.g. Qwen3.5-27B-FP8) use a **shared serving runtime** instead of a per-model one. Deploy the runtime first:

```bash
# Deploy the shared OSS vLLM runtime (upstream docker.io/vllm/vllm-openai image)
oc apply -f gitops/platform/models/oss-vllm-runtime.yaml
```

## Shared Serving Runtimes

### OSS vLLM Runtime (`oss-vllm-runtime`)

The `oss-vllm-runtime` uses the upstream [`docker.io/vllm/vllm-openai`](https://hub.docker.com/r/vllm/vllm-openai/tags) image rather than the Red Hat `registry.redhat.io/rhaiis/vllm-cuda-rhel9` image. This is useful when you need a more recent vLLM version than the currently available RHAIIS build.

**Deployed by:** `gitops/platform/models/oss-vllm-runtime.yaml`
**Source:** `platform/models/serving/shared/oss-vllm-runtime/`

#### Pinning to a specific vLLM version

The image tag is controlled by the `images` field in `platform/models/serving/shared/oss-vllm-runtime/kustomization.yaml`:

```yaml
images:
  - name: docker.io/vllm/vllm-openai
    newTag: latest   # Change this to pin a specific release, e.g. v0.9.0
```

Available tags: <https://hub.docker.com/r/vllm/vllm-openai/tags>

Any model that references `runtime: oss-vllm-runtime` in its InferenceService will automatically pick up the updated image on the next ArgoCD sync.

### Deploying Qwen3.5-27B-FP8

#### One-command deploy (recommended)

```bash
oc apply -f gitops/platform/qwen35-27b-fp8-all.yaml
```

This deploys all five ArgoCD Applications in the correct order: base resources, g6e-2xlarge hardware profile, the shared `oss-vllm-runtime`, the model download job, and the InferenceService.

#### Step-by-step deploy

```bash
# 1. Deploy the shared OSS vLLM runtime (once — reused by future models)
oc apply -f gitops/platform/models/oss-vllm-runtime.yaml

# 2. Download the model (~30.9 GB to a 36 Gi PVC on g6e.2xlarge / L40S)
oc apply -f gitops/platform/models/qwen35-27b-fp8-pvc.yaml

# Monitor download
oc logs -f job/download-qwen35-27b-fp8 -n demo

# 3. Deploy serving (after download completes)
oc apply -f gitops/platform/models/qwen35-27b-fp8-serving.yaml

# Wait for InferenceService to be ready
oc wait --for=condition=Ready inferenceservice/qwen35-27b-fp8 \
  -n demo --timeout=600s
```

### Deploying Nemotron-3-Nano-30B-A3B-FP8 with llm-d

This model uses **llm-d distributed inference** rather than KServe's standard `InferenceService`. It is deployed manually from the RHOAI model catalog as an `LLMInferenceService`, with tensor parallelism across 2 GPUs per replica and 2 replicas total — filling all 4 L40S GPUs on a `g6e.12xlarge` node.

**Prerequisites:** Must have the LWS operator and `openshift-ai-inference` Gateway deployed. See [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) for implementation.

#### Step 1: Deploy the hardware profile

```bash
oc apply -f gitops/platform/hardware-profiles/g6e-12xlarge.yaml
```

#### Step 2: Deploy from the RHOAI model catalog

In the RHOAI dashboard, navigate to **Models → Model catalog** and select **NVIDIA-Nemotron-3-Nano-30B-A3B-FP8**. Use the values below when filling out the deployment form:

| Field | Value |
|-------|-------|
| Deployment type | Distributed inference with llm-d |
| Hardware Profile | `AWS g6e.12xlarge (4x NVIDIA L40S)` |
| Replicas | `2` |
| GPUs per replica | `2` |
| Tensor parallelism | `2` |

**vLLM arguments** (enter in the "Additional arguments" or "Extra args" field, one per line):

```
--trust-remote-code
--kv-cache-dtype=fp8
--async-scheduling
--gpu_memory_utilization=0.95
--max-model-len=262144
--max-num-seqs=8
--tensor-parallel-size=2
```

**Environment variables** (enter in the environment variables section):

| Name | Value |
|------|-------|
| `VLLM_USE_FLASHINFER_MOE_FP8` | `1` |
| `VLLM_FLASHINFER_MOE_BACKEND` | `throughput` |
| `VLLM_ALLOW_LONG_MAX_MODEL_LEN` | `1` |

**Resource limits** (per replica pod):

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | `8` | `16` |
| Memory | `64Gi` | `128Gi` |
| `nvidia.com/gpu` | `2` | `2` |

**Key configuration parameters explained:**

| Parameter | Value | Why |
|-----------|-------|-----|
| `--trust-remote-code` | _(flag)_ | Required — Nemotron uses a custom Mamba-2 architecture not bundled with vLLM |
| `--tensor-parallel-size=2` | `2` | Shards the model across 2 GPUs per replica, leaving ~28 GB per GPU for KV cache instead of ~12 GB |
| `--kv-cache-dtype=fp8` | `fp8` | Halves KV cache memory, enabling much longer context windows on L40S |
| `--gpu_memory_utilization=0.95` | `0.95` | Allocates 95% of GPU VRAM to model weights + KV cache |
| `--max-model-len=262144` | `262144` | 256K token context — NVIDIA's recommended max for this model on TP=2 with FP8 KV cache |
| `--max-num-seqs=8` | `8` | Limits concurrent sequences; tune lower to reduce latency spikes |
| `--async-scheduling` | _(flag)_ | NVIDIA-recommended for Nemotron-3-Nano to reduce CPU overhead between decode steps |
| `VLLM_ALLOW_LONG_MAX_MODEL_LEN=1` | `"1"` | **Required** to unlock context lengths beyond 128K tokens |
| `VLLM_USE_FLASHINFER_MOE_FP8=1` | `"1"` | Enables FlashInfer's optimized FP8 MoE dispatch kernels |
| `VLLM_FLASHINFER_MOE_BACKEND=throughput` | `throughput` | Optimizes for batch throughput over single-request latency |
| `parallelism.tensor: 2` | `2` | llm-d-level TP setting (must match `--tensor-parallel-size`) |
| `replicas: 2` | `2` | Two independent serving instances (4 GPUs total), load-balanced by llm-d's EPP scheduler |

> **Note on `--max-model-len`:** Start at `262144` (256K). If the pod OOMKills or vLLM reports insufficient KV cache blocks, reduce to `131072` (128K) or `65536` (64K). The L40S has 48 GB per GPU but only Nemotron's 6 attention layers consume KV cache (the 23 Mamba-2 layers use recurrent state, not KV cache), so 256K is realistic.

#### Step 3: Verify the deployment

```bash
# Check LLMInferenceService status
oc get llminferenceservice nemotron-3-nano-30b-fp8 -n demo

# Check the LeaderWorkerSet pods (2 replicas × 2 GPUs = 4 pods expected)
oc get pods -n demo -l app.kubernetes.io/name=nemotron-3-nano-30b-fp8

# Watch pod logs for vLLM startup (takes 3-5 minutes to load 32.7GB model)
oc logs -f <pod-name> -n demo -c main

# Confirm the inference gateway is ready
oc get gateway openshift-ai-inference -n openshift-ingress

# Test inference
ENDPOINT=$(oc get llminferenceservice nemotron-3-nano-30b-fp8 -n demo \
  -o jsonpath='{.status.url}')
curl -k "${ENDPOINT}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-FP8",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

#### Troubleshooting Nemotron / llm-d

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Pod OOMKills | `max-model-len` too high | Reduce `--max-model-len` to `131072` or `65536` |
| `ValueError: max_model_len ... too large` | Missing env var | Ensure `VLLM_ALLOW_LONG_MAX_MODEL_LEN=1` is set |
| Pods stay Pending | LWS operator not installed | Verify `oc get csv -n openshift-operators \| grep lws` shows `Succeeded` |
| `LLMInferenceService` not found | llm-d CRDs not registered | Check `oc get crd llminferenceservices.serving.kserve.io` |
| Gateway not programmed | `openshift-ai-inference` Gateway missing | Check `oc get gateway -n openshift-ingress` and verify ArgoCD synced `gateway-api` |
| vLLM `trust_remote_code` error | Missing arg | Ensure `--trust-remote-code` is in the container args |

## Working with Gated Models

Some models require HuggingFace license acceptance and an API token. This includes Llama and FLUX models.

```bash
# 1. Accept the model license on HuggingFace:
#    - Llama: https://huggingface.co/meta-llama/Meta-Llama-3-8B-Instruct
#    - FLUX.2-Klein: https://huggingface.co/black-forest-labs/FLUX.2-klein-9b-fp8

# 2. Create HuggingFace token secret (required before deploying any gated model)
oc create secret generic huggingface-token \
  --from-literal=token=hf_YOUR_TOKEN_HERE \
  -n demo

# 3. Deploy model (example: FLUX.2-Klein-9B)
oc apply -f gitops/platform/models/flux2-klein-9b-pvc.yaml
oc apply -f gitops/platform/models/flux2-klein-9b-serving.yaml
```

> **Note:** Unlike other models where the token is optional, FLUX.2-Klein-9B-FP8 **requires** the `huggingface-token` secret to be present. The download job will exit with an error if the secret is missing.

## Direct Deployment (Without GitOps)

If you prefer not to use ArgoCD:

```bash
# Deploy hardware profile
oc apply -k platform/hardware-profiles/g4dn-xlarge

# Download model
oc apply -k platform/models/jobs/qwen3-vl-embedding-2b/pvc

# Deploy serving
oc apply -k platform/models/serving/qwen3-vl-embedding-2b/pvc
```

## Storage Options

### PVC Storage (Default)

- Simple setup
- Works with any storage class
- Direct filesystem access
- Best for demos and single-instance serving

### MinIO Storage (Available for Granite and Llama)

- S3-compatible API
- Multi-tenant access
- Better for production scenarios
- Requires MinIO deployment

```bash
# Deploy model to MinIO
oc apply -f gitops/platform/models/granite-7b-minio.yaml
```

## Monitoring

```bash
# Check model download status
oc get jobs -n demo
oc logs -f job/download-<model-name> -n demo

# Check model serving status
oc get inferenceservice -n demo
oc describe inferenceservice <model-name> -n demo

# Check pods
oc get pods -n demo -l app.kubernetes.io/name=<model-name>

# View serving logs
oc logs -f deployment/<model-name>-predictor -n demo
```

## Troubleshooting

### Model Download Fails

**Issue:** Job fails with "401 Unauthorized"

**Solution:** Create HuggingFace token secret (required for gated models)

```bash
oc create secret generic huggingface-token \
  --from-literal=token=hf_YOUR_TOKEN \
  -n demo
```

### Model Not Scheduling

**Issue:** Pod stays in Pending state

**Solution:** Check GPU availability and node taints

```bash
# Check GPU nodes
oc get nodes -l node.kubernetes.io/instance-type=g4dn.xlarge

# Check GPU resources
oc describe node <gpu-node-name> | grep -A5 nvidia.com/gpu
```

### InferenceService Not Ready

**Issue:** InferenceService stuck in "Not Ready"

**Solution:** Check predictor pod logs

```bash
oc get pods -n demo
oc logs <predictor-pod-name> -n demo
```

### PVC Not Binding

**Issue:** PVC stays in Pending state with message `waiting for first consumer to be created before binding`

**Root cause:** This is a `WaitForFirstConsumer` deadlock with ArgoCD sync waves. The storage class (e.g. AWS EBS `gp3-csi`) uses `WaitForFirstConsumer` binding mode, meaning the PVC won't bind until a pod is scheduled to consume it. ArgoCD, however, waits for the PVC to reach `Bound` (Healthy) before advancing to the next sync wave — which is where the download Job lives. The Job never deploys, so the PVC never binds. Classic chicken-and-egg.

**Established fix (already applied to all models in this repo):**

1. **PVC and Job must share the same sync wave (`"12"`)** — ArgoCD deploys them simultaneously. The Job's pod schedules, consumes the PVC, and the binding completes within one wave.
2. **`SkipDryRunOnMissingResource=true` on the PVC** — prevents ArgoCD's dry-run from interfering with the storage provisioner's admission webhook.
3. **No explicit `storageClassName`** — let the cluster's default storage class handle provisioning. Hardcoding a class name (e.g. `gp3-csi`) that may differ across clusters is a common source of Pending PVCs.

**Required PVC annotation pattern:**
```yaml
annotations:
  argocd.argoproj.io/sync-wave: "12"
  argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

**Required Job annotation:**
```yaml
annotations:
  argocd.argoproj.io/sync-wave: "12"   # Must match PVC wave — NOT "13"
```

**If a PVC is already stuck**, the quickest unblock is to manually apply the Job so its pod triggers binding:
```bash
oc apply -f platform/models/jobs/<model-name>/base/job.yaml -n demo
```

Then push any pending git changes and do a hard refresh in ArgoCD.

## Repository Structure

```
rhoai-model-serving/
├── README.md              # This file
├── bootstrap.sh           # GitOps installer script
├── bootstrap/             # GitOps operator manifests
│   └── gitops-operator/
├── platform/
│   ├── hardware-profiles/     # GPU hardware profiles
│   │   ├── g4dn-xlarge/       # NVIDIA T4 configuration
│   │   ├── g6-4xlarge/        # NVIDIA L4 configuration
│   │   ├── g6e-2xlarge/       # NVIDIA L40S (1 GPU) configuration
│   │   └── g6e-12xlarge/      # NVIDIA L40S (4 GPUs) for llm-d distributed inference
│   └── models/
│       ├── base/              # Common resources (namespace, RBAC)
│       ├── jobs/              # Model download jobs
│       │   ├── flux2-klein-9b/
│       │   ├── granite-7b/
│       │   ├── llama-3-8b/
│       │   ├── qwen3-vl-4b/
│       │   ├── qwen3-vl-8b/
│       │   ├── qwen3-vl-8b-fp8/
│       │   ├── qwen3-vl-embedding-2b/
│       │   └── qwen35-27b-fp8/
│       └── serving/           # Model serving configs
│           ├── flux2-klein-9b/
│           ├── granite-7b/
│           ├── llama-3-8b/
│           ├── qwen3-vl-4b/
│           ├── qwen3-vl-8b/
│           ├── qwen3-vl-8b-fp8/
│           ├── qwen3-vl-embedding-2b/
│           ├── qwen35-27b-fp8/
│           └── shared/
│               └── oss-vllm-runtime/  # Upstream vLLM runtime (reusable)
└── gitops/
    └── platform/
        ├── hardware-profiles/ # ArgoCD apps for hardware profiles
        └── models/            # ArgoCD apps for models
```

## Common Patterns

### Deploy Multiple Models

```bash
# Download models (can run in parallel)
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-pvc.yaml
oc apply -f gitops/platform/models/qwen3-vl-4b-pvc.yaml
oc apply -f gitops/platform/models/granite-7b-pvc.yaml

# Wait for downloads to complete
oc get jobs -n demo -w

# Deploy serving (after downloads complete)
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-serving.yaml
oc apply -f gitops/platform/models/qwen3-vl-4b-serving.yaml
oc apply -f gitops/platform/models/granite-7b-serving.yaml
```

### Clean Up Resources

```bash
# Delete serving
oc delete -f gitops/platform/models/qwen3-vl-embedding-2b-serving.yaml

# Delete download job (PVC remains)
oc delete job download-qwen3-vl-embedding-2b -n demo

# Delete PVC (removes model data)
oc delete pvc qwen3-vl-embedding-2b-model-storage -n demo
```

## Resources

- [Red Hat OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai)
- [KServe Documentation](https://kserve.github.io/website/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [llm-d Documentation](https://github.com/llm-d/llm-d)
- [HuggingFace Hub](https://huggingface.co/models)
- [NVIDIA Nemotron-3-Nano-30B-A3B-FP8 on HuggingFace](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-FP8)

## Related Repositories

- [openshift-infra](https://github.com/redhat-ai-americas/openshift-infra) - GPU MachineSets for AWS
- [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) - RHOAI platform installation
