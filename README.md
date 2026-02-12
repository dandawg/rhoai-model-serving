# RHOAI Model Serving

Simplified model deployment and serving for Red Hat OpenShift AI (RHOAI). This repository provides hardware profiles, model download jobs, and model serving configurations for deploying AI models on GPU-enabled OpenShift clusters.

## Prerequisites

### Required Infrastructure

1. **OpenShift Cluster with GPUs**
   - See [openshift-infra](https://github.com/redhat-ai-americas/openshift-infra) for GPU MachineSets
   - AWS instance types: `g4dn.xlarge` (NVIDIA T4), `g6.4xlarge` (NVIDIA L4), or `g6e.2xlarge` (NVIDIA L40S)

2. **RHOAI Installed**
   - See [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) for RHOAI installation
   - Requires NVIDIA GPU Operator and RHOAI Operator installed
   - DataScienceCluster configured with model serving enabled

3. **GitOps (Optional but Recommended)**
   - OpenShift GitOps Operator installed
   - ArgoCD configured for automated deployments

## Quick Start

### 1. Deploy Hardware Profiles

Hardware profiles define GPU node constraints and resource limits for model serving.

```bash
# For NVIDIA T4 GPUs (g4dn.xlarge)
oc apply -f gitops/platform/hardware-profiles/g4dn-xlarge.yaml

# For NVIDIA L4 GPUs (g6.4xlarge)
oc apply -f gitops/platform/hardware-profiles/g6-4xlarge.yaml

# For NVIDIA L40S GPUs (g6e.2xlarge)
oc apply -f gitops/platform/hardware-profiles/g6e-2xlarge.yaml
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
| **Granite-7B** | 14GB | T4/L4 | General instruction following | g4dn-xlarge |
| **Llama-3-8B** | 16GB | L4 | General purpose (gated) | g6-4xlarge |

## Hardware Profiles

| Profile | Instance Type | GPU | vCPUs | Memory | Best For |
|---------|--------------|-----|-------|---------|----------|
| **g4dn-xlarge** | AWS g4dn.xlarge | NVIDIA T4 (16GB) | 4 | 16GB | Small models, embeddings |
| **g6-4xlarge** | AWS g6.4xlarge | NVIDIA L4 (24GB) | 16 | 64GB | Large models, vision tasks |
| **g6e-2xlarge** | AWS g6e.2xlarge | NVIDIA L40S (48GB) | 8 | 64GB | Large models (13B+), high GPU memory workloads |

## Model Deployment Workflow

### Step-by-Step Process

1. **Hardware Profile** → Define GPU node selection and tolerations
2. **Model Download** → Download model from HuggingFace to PVC
3. **Model Serving** → Deploy ServingRuntime and InferenceService

### GitOps Files

Each model has two GitOps files:

- `*-pvc.yaml` - Downloads the model to persistent storage
- `*-serving.yaml` - Deploys the model serving endpoint

## Working with Gated Models (Llama)

Some models require HuggingFace license acceptance:

```bash
# 1. Accept license at https://huggingface.co/meta-llama/Meta-Llama-3-8B-Instruct

# 2. Create HuggingFace token secret
oc create secret generic huggingface-token \
  --from-literal=token=hf_YOUR_TOKEN_HERE \
  -n demo

# 3. Deploy model
oc apply -f gitops/platform/models/llama-3-8b-pvc.yaml
oc apply -f gitops/platform/models/llama-3-8b-serving.yaml
```

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

**Issue:** PVC stays in Pending state

**Solution:** Check storage class and availability

```bash
oc get storageclass
oc describe pvc <pvc-name> -n demo
```

## Repository Structure

```
rhoai-model-serving/
├── platform/
│   ├── hardware-profiles/     # GPU hardware profiles
│   │   ├── g4dn-xlarge/       # NVIDIA T4 configuration
│   │   ├── g6-4xlarge/        # NVIDIA L4 configuration
│   │   └── g6e-2xlarge/       # NVIDIA L40S configuration
│   └── models/
│       ├── base/              # Common resources (namespace, RBAC)
│       ├── jobs/              # Model download jobs
│       │   ├── granite-7b/
│       │   ├── llama-3-8b/
│       │   ├── qwen3-vl-4b/
│       │   ├── qwen3-vl-8b/
│       │   ├── qwen3-vl-8b-fp8/
│       │   └── qwen3-vl-embedding-2b/
│       └── serving/           # Model serving configs
│           ├── granite-7b/
│           ├── llama-3-8b/
│           ├── qwen3-vl-4b/
│           ├── qwen3-vl-8b/
│           ├── qwen3-vl-8b-fp8/
│           └── qwen3-vl-embedding-2b/
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
- [HuggingFace Hub](https://huggingface.co/models)

## Related Repositories

- [openshift-infra](https://github.com/redhat-ai-americas/openshift-infra) - GPU MachineSets for AWS
- [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) - RHOAI platform installation
