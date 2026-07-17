# Engine Interface Refactor

## Problem

Engine-specific logic is scattered across individual files (`engine_vllm.go`, `engine_ollama.go`, etc.) with selection via a `switch` statement in `pod_plan.go`. There is no formal contract defining what an "engine" must implement. This makes it difficult to:

* Add new engines (e.g., SGLang, TensorRT-LLM)
* Gate engine-specific features (disaggregation is VLLM-only, adapters are VLLM-only)
* Test engines in isolation

## Solution

Formalize engine-specific logic into a Go `interface` and an `EngineRegistry`.

### Engine Interface

```go
// internal/modelcontroller/engine.go
type Engine interface {
    Name() string
    PodForModel(m *v1.Model, c ModelConfig) *corev1.Pod
    SupportsDisaggregation() bool
    SupportsAdapters() bool
    HealthEndpoint() string
    DefaultPort() int
}
```

### Engine Registry

```go
type EngineRegistry struct {
    engines map[string]Engine
}

func NewEngineRegistry(r *ModelReconciler) *EngineRegistry {
    return &EngineRegistry{
        engines: map[string]Engine{
            "VLLM":          &VLLMEngine{reconciler: r},
            "OLlama":        &OLlamaEngine{reconciler: r},
            "FasterWhisper":  &FasterWhisperEngine{reconciler: r},
            "Infinity":      &InfinityEngine{reconciler: r},
        },
    }
}
```

### Current Code (Before)

```go
// pod_plan.go
switch model.Spec.Engine {
case kubeaiv1.OLlamaEngine:
    podForModel = r.oLlamaPodForModel(model, modelConfig)
case kubeaiv1.FasterWhisperEngine:
    podForModel = r.fasterWhisperPodForModel(model, modelConfig)
case kubeaiv1.InfinityEngine:
    podForModel = r.infinityPodForModel(model, modelConfig)
default:
    podForModel = r.vLLMPodForModel(model, modelConfig)
}
```

### Refactored Code (After)

```go
// pod_plan.go
engine, err := r.engineRegistry.Get(model.Spec.Engine)
if err != nil {
    return nil, err
}
podForModel := engine.PodForModel(model, modelConfig)
```

### Affected Components

| Component | Change |
|---|---|
| `internal/modelcontroller/pod_plan.go` | Replace `switch` with `EngineRegistry.Get()` |
| `internal/modelcontroller/engine_vllm.go` | Implement `Engine` interface on `VLLMEngine` struct |
| `internal/modelcontroller/engine_ollama.go` | Implement `Engine` interface on `OLlamaEngine` struct |
| `internal/modelcontroller/engine_fasterwhisper.go` | Implement `Engine` interface on `FasterWhisperEngine` struct |
| `internal/modelcontroller/engine_infinity.go` | Implement `Engine` interface on `InfinityEngine` struct |
| `internal/modelcontroller/model_controller.go` | Add `EngineRegistry` field |
