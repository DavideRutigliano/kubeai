# Prefill / Decode Disaggregation

## Problem

KubeAI runs a single vLLM instance per Pod handling both prefill (compute-bound) and decode (memory-bound) phases. Under high load, long prompts cause latency spikes for ongoing decode streams. This is the single biggest performance gap versus systems like llm-d.

While working within the following constraints:

* Disaggregation should be opt-in and only supported for the VLLM engine
* The existing single-phase behavior must remain the default
* The implementation should start simple (TCP-based KV transfer) before attempting RDMA/NIXL

## Solution

Introduce a `disaggregated` field in the Model CRD. When enabled, the controller creates two independent Pod pools: one optimized for prefill, one for decode.

### Example Model

```yaml
kind: Model
metadata:
  name: llama-3.1-70b-instruct
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-70B-Instruct"
  engine: VLLM
  resourceProfile: "nvidia-gpu-h100:4"
  disaggregated:
    enabled: true
    prefillReplicas: 2
    decodeReplicas: 4
    kvTransferMethod: tcp  # or "nixl" for RDMA
```

### Model Without Disaggregation (Default, No Change)

```yaml
kind: Model
metadata:
  name: llama-3.1-8b-instruct
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-8B-Instruct"
  engine: VLLM
  resourceProfile: "nvidia-gpu-l4:1"
  # No disaggregated field = single-phase serving (current behavior)
```

### Request Flow

```
Client → Proxy → Prefill Pod (process prompt, compute KV cache)
                       ↓ KV transfer (TCP or NIXL)
                  Decode Pod (generate tokens) → stream to Client
```

### Controller Changes

The `ModelReconciler` creates two sets of Pods with different labels and configurations:

* **Prefill Pods**: `kubeai.org/role: prefill`, higher `--max-num-batched-token`, compute-focused resources
* **Decode Pods**: `kubeai.org/role: decode`, larger KV-cache allocation, memory-focused resources

Each pool scales independently based on its own metrics.

### Phased Implementation

1. **Phase 1**: Two Pod pools with TCP-based KV transfer via vLLM's `--kv-transfer-method tcp`
2. **Phase 2**: NIXL/RDMA transfer for high-performance GPU-to-GPU KV cache movement
3. **Phase 3**: Intelligent phase routing (prefix-aware for prefill, load-aware for decode)

### Affected Components

| Component | Change |
|---|---|
| `api/k8s/v1/model_types.go` | Add `DisaggregatedServing` struct to `ModelSpec` |
| `internal/modelcontroller/engine_vllm.go` | Generate separate Pod specs for prefill/decode roles |
| `internal/modelcontroller/model_controller.go` | Manage two Pod pools per disaggregated Model |
| `internal/loadbalancer/` | Two-phase routing (prefill → KV transfer → decode) |
| `internal/modelautoscaler/autoscaler.go` | Independent scaling for prefill and decode pools |

## Relevant Reading

* [vLLM Disaggregated Prefill](https://docs.vllm.ai/en/latest/serving/disagg_prefill.html)
* [NIXL](https://github.com/ai-dynamo/nixl)
* [llm-d Disaggregated Serving Guide](https://github.com/llm-d/llm-d/tree/main/guides/pd-disaggregation)
