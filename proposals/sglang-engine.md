# SGLang Engine

## Problem

KubeAI supports vLLM, Ollama, FasterWhisper, and Infinity. [SGLang](https://github.com/sgl-project/sglang) is a fast serving runtime that has become a genuine competitor to vLLM for several workloads:

* **Structured output** (JSON schema enforcement) is a first-class feature in SGLang via its grammar-based engine, while vLLM relies on outlines (slower, separate library).
* **Speculative decoding** (draft model + target model) has a more mature implementation in SGLang and achieves better latency gains on codegen and chat models.
* **Radix attention** (SGLang's equivalent of prefix caching) uses an explicit radix tree for KV-cache reuse, which is more memory-efficient than vLLM's hash-based prefix caching.
* **Multi-modal models** (vision-language) have better throughput on SGLang for certain model families (Qwen-VL, LLaVA).

Users currently work around this by setting `image` to a custom SGLang image and manually passing `args`, but they lose engine-specific validation and the correct port default.

This proposal depends on [Engine Interface Refactor](./engine-interface-refactor.md) being implemented first.

## Solution

Add `SGLang` as a first-class `Engine` enum value and implement the `Engine` interface for it.

### Model CRD Change

```yaml
# api/k8s/v1/model_types.go
# +kubebuilder:validation:Enum=OLlama;VLLM;FasterWhisper;Infinity;SGLang
Engine string `json:"engine"`
```

### Example Models

```yaml
# Basic SGLang model
kind: Model
metadata:
  name: qwen2-5-72b-instruct
spec:
  url: "hf://Qwen/Qwen2.5-72B-Instruct"
  engine: SGLang
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-h100:4"

# SGLang with speculative decoding (draft model via args)
kind: Model
metadata:
  name: llama-3-1-70b-instruct-spec
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-70B-Instruct"
  engine: SGLang
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-h100:4"
  args:
    - "--speculative-draft-model-path=/draft-model"
    - "--speculative-num-steps=5"
    - "--speculative-eagle-topk=4"

# SGLang for vision-language (multi-modal)
kind: Model
metadata:
  name: qwen2-vl-7b
spec:
  url: "hf://Qwen/Qwen2-VL-7B-Instruct"
  engine: SGLang
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-l4:1"
  args:
    - "--chat-template=qwen2-vl"
```

### Engine Interface Implementation

```go
// internal/modelcontroller/engine_sglang.go

type SGLangEngine struct{}

func (e *SGLangEngine) Name() string { return "SGLang" }
func (e *SGLangEngine) SupportsDisaggregation() bool { return false } // future work
func (e *SGLangEngine) SupportsAdapters() bool { return false }       // future work: SGLang has --lora-paths
func (e *SGLangEngine) HealthEndpoint() string { return "/health" }
func (e *SGLangEngine) DefaultPort() int { return 30000 }
func (e *SGLangEngine) SupportsDRA() bool { return true }
func (e *SGLangEngine) SupportsMultiNode() bool { return false } // future: SGLang supports TP across nodes

func (e *SGLangEngine) PodForModel(m *v1.Model, c ModelConfig) *corev1.Pod {
    // ... see Affected Components
}
```

### Key Differences from vLLM

| Aspect | vLLM | SGLang |
|---|---|---|
| Default port | 8000 | 30000 |
| Model flag | `--model` | `--model-path` |
| Served name | `--served-model-name` | `--served-model-name` (same) |
| Tensor parallelism | `--tensor-parallel-size` | `--tp-size` |
| Health endpoint | `/health` | `/health` |
| LoRA flag | `--enable-lora` | `--lora-paths` (static only in v1) |
| Prefix caching | `--enable-prefix-caching` | enabled by default (radix attention) |

### Pod Builder (`engine_sglang.go`)

The builder follows the vLLM pattern exactly but with SGLang-specific args:

```go
args := []string{
    "--model-path=" + sglangModelFlag,
    "--served-model-name=" + m.Name,
    "--host=0.0.0.0",
    "--port=30000",
}

// Derive tensor parallelism from resource profile multiplier.
if tpSize := c.resourceProfileCount; tpSize > 1 {
    args = append(args, fmt.Sprintf("--tp-size=%d", tpSize))
}

args = append(args, m.Spec.Args...)
```

### CEL Validation

Add a rule to prevent adapters on SGLang (until LoRA support lands):

```go
// api/k8s/v1/model_types.go
// +kubebuilder:validation:XValidation:rule="!has(self.adapters) || (self.engine == \"VLLM\")",message="adapters only supported with VLLM engine."
```

The existing rule already covers this since it only allows adapters for `VLLM`. No change needed.

### Resource Profile Images

The system config `ModelServers` image map needs a new key for SGLang:

```yaml
# config.yaml (Helm values)
modelServers:
  SGLang:
    images:
      default: "lmsysorg/sglang:latest"
      "nvidia-gpu": "lmsysorg/sglang:latest-cuda"
```

### Affected Components

| Component | Change |
|---|---|
| `api/k8s/v1/model_types.go` | Add `SGLang` to `Engine` enum |
| `internal/modelcontroller/engine_sglang.go` (NEW) | Implement `Engine` interface |
| `internal/modelcontroller/engine_registry.go` | Register `SGLangEngine` |
| `internal/config/system.go` | Add SGLang image entry to `ModelServers` |
| `charts/kubeai/values.yaml` | Add SGLang default image |
| `test/integration/` | Add SGLang integration test (mock) |

## Implementation Phases

### Phase 1: Basic Serving
* Add `SGLang` enum value and pod builder.
* Support `hf://`, `pvc://`, `s3://` URLs.
* Image configurable via system config.
* Features: `TextGeneration`, `TextEmbedding`.

### Phase 2: LoRA Adapters
* SGLang v0.4+ supports dynamic LoRA via `/add_lora` REST API.
* Implement `SGLangLoraClient` (analogous to `vllmclient`).
* Set `SupportsAdapters() = true`.

### Phase 3: Disaggregation
* SGLang 0.4+ has experimental P/D disaggregation.
* Align with [Prefill/Decode Disaggregation proposal](./prefill-decode-disaggregation.md).

## Relevant Reading

* [SGLang GitHub](https://github.com/sgl-project/sglang)
* [SGLang Server Arguments](https://sgl-project.github.io/references/server_arguments.html)
* [SGLang Radix Attention](https://lmsys.org/blog/2024-01-17-sglang/)
* [KubeAI Issue #413](https://github.com/substratusai/kubeai/issues/413)
