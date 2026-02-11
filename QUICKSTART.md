# Quick Reference

## Common Commands

### Deploy Everything for a Single Model

**Option 1: Consolidated (Recommended)**
```bash
# Qwen3-VL-Embedding-2B on T4 GPU
oc apply -f gitops/platform/qwen3-vl-embedding-2b-all.yaml

# Qwen3-VL-4B on L4 GPU
oc apply -f gitops/platform/qwen3-vl-4b-all.yaml
```

**Option 2: Separate Files**
```bash
# Qwen3-VL-Embedding-2B on T4 GPU
oc apply -f gitops/platform/hardware-profiles/g4dn-xlarge.yaml
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-pvc.yaml
oc apply -f gitops/platform/models/qwen3-vl-embedding-2b-serving.yaml

# Qwen3-VL-4B on L4 GPU
oc apply -f gitops/platform/hardware-profiles/g6-4xlarge.yaml
oc apply -f gitops/platform/models/qwen3-vl-4b-pvc.yaml
oc apply -f gitops/platform/models/qwen3-vl-4b-serving.yaml
```

### Check Status

```bash
# Model download
oc get jobs -n demo
oc logs -f job/download-qwen3-vl-embedding-2b -n demo

# Model serving
oc get inferenceservice -n demo
oc get pods -n demo
```

### Test Model

```bash
# From within cluster
oc run curl-test --image=curlimages/curl -it --rm -n demo -- \
  curl -X POST http://MODEL-NAME-predictor.demo.svc.cluster.local/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "MODEL-NAME", "prompt": "Hello", "max_tokens": 50}'
```

## Model-to-GPU Mapping

| Model | Recommended GPU | GitOps Files |
|-------|----------------|--------------|
| Qwen3-VL-Embedding-2B | T4 (g4dn.xlarge) | `qwen3-vl-embedding-2b-*.yaml` |
| Qwen3-VL-4B | T4 (g4dn.xlarge) | `qwen3-vl-4b-*.yaml` |
| Granite-7B | T4 (g4dn.xlarge) | `granite-7b-*.yaml` |
| Qwen3-VL-8B | L4 (g6.4xlarge) | `qwen3-vl-8b-*.yaml` |
| Qwen3-VL-8B-FP8 | L4 (g6.4xlarge) | `qwen3-vl-8b-fp8-*.yaml` |
| Llama-3-8B | L4 (g6.4xlarge) | `llama-3-8b-*.yaml` |

## File Naming Convention

```
<model-name>-pvc.yaml       → Downloads model to PVC
<model-name>-minio.yaml     → Downloads model to MinIO (select models)
<model-name>-serving.yaml   → Deploys model serving
```

## Troubleshooting Quick Fixes

### Model stuck in Pending
```bash
# Check GPU nodes exist
oc get nodes -l node.kubernetes.io/instance-type=g4dn.xlarge
```

### Download fails with 401
```bash
# Create HuggingFace token
oc create secret generic huggingface-token \
  --from-literal=token=hf_YOUR_TOKEN -n demo
```

### InferenceService not Ready
```bash
# Check predictor logs
oc get pods -n demo
oc logs <predictor-pod-name> -n demo
```

## Clean Up

```bash
# Delete serving (keeps model)
oc delete -f gitops/platform/models/MODEL-NAME-serving.yaml

# Delete download job (keeps PVC)
oc delete job download-MODEL-NAME -n demo

# Delete PVC (removes model data!)
oc delete pvc MODEL-NAME-model-storage -n demo
```
