# InferenceClass

## Problem

KubeAI currently conflates two distinct personas in a single `Model` manifest:

**Platform engineer** — knows the cluster's hardware topology: which GPU types exist, their node selectors and tolerations, how much memory they have, which DRA templates are available, what engine image versions are approved.

**Application developer** — knows the model they want to serve: the Hugging Face repo, which features it needs (text generation, embeddings), whether it uses LoRA adapters, how much traffic to expect.

Today a developer writing a `Model` must specify `engine: VLLM`, `resourceProfile: nvidia-gpu-l4:1`, tolerations, and engine args — all cluster-specific plumbing they should not need to know. When the platform team upgrades a GPU type or changes the DRA setup, every `Model` manifest must be updated.

This mirrors a solved problem in Kubernetes storage: `StorageClass` decouples the application developer (who writes a `PersistentVolumeClaim` with `storageClassName: fast`) from the storage admin (who writes the `StorageClass` with provisioner details). The same pattern applies to inference hardware.

## Solution

Introduce an `InferenceClass` cluster-scoped CRD owned by platform engineers. It encapsulates the full hardware and engine configuration for a serving shape. The `Model` CRD is simplified: developers specify only what they know (model ID, features, scaling bounds) and reference an `InferenceClass` by name.

### InferenceClass CRD

```yaml
apiVersion: kubeai.org/v1
kind: InferenceClass
metadata:
  name: nvidia-l4-single
spec:
  # Hardware constraints — cluster admin territory
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-l4
  tolerations:
    - key: nvidia.com/gpu
      effect: NoSchedule
  resources:
    requests:
      cpu: "6"
      memory: "24Gi"
    limits:
      nvidia.com/gpu: "1"
  # OR DRA (mutually exclusive with GPU entries in resources.limits)
  # dra:
  #   resourceClaimTemplateName: nvidia-l4-exclusive

  # Engine defaults — platform team picks the approved image and baseline args
  engine:
    type: VLLM
    image: "vllm/vllm-openai:v0.8.3"
    args:
      - "--gpu-memory-utilization=0.95"
      - "--enable-prefix-caching"
    env:
      VLLM_WORKER_MULTIPROC_METHOD: "spawn"

  # Scaling defaults — platform team sets sensible bounds for this hardware shape
  scaling:
    minReplicas: 0
    maxReplicas: 8
    targetRequests: 100
    scaleDownDelaySeconds: 30

---
# Multi-GPU shape: platform team names the shape, not the user
apiVersion: kubeai.org/v1
kind: InferenceClass
metadata:
  name: nvidia-h100-4gpu
spec:
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-h100-80gb
  tolerations:
    - key: nvidia.com/gpu
      effect: NoSchedule
  resources:
    requests:
      cpu: "24"
      memory: "320Gi"
    limits:
      nvidia.com/gpu: "4"
  engine:
    type: VLLM
    image: "vllm/vllm-openai:v0.8.3"
    args:
      - "--tensor-parallel-size=4"
      - "--gpu-memory-utilization=0.95"
  scaling:
    minReplicas: 1
    maxReplicas: 4
    targetRequests: 50

---
# CPU-only shape for embeddings
apiVersion: kubeai.org/v1
kind: InferenceClass
metadata:
  name: cpu-embedding
spec:
  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
  engine:
    type: Infinity
    image: "michaelf/infinity:latest"
  scaling:
    minReplicas: 1
    maxReplicas: 16
    targetRequests: 200
```

### Simplified Model CRD

The developer writes only what they own:

```yaml
apiVersion: kubeai.org/v1
kind: Model
metadata:
  name: qwen3-8b
spec:
  url: "hf://Qwen/Qwen3-8B"
  inferenceClass: nvidia-l4-single
  features: [TextGeneration]

---
apiVersion: kubeai.org/v1
kind: Model
metadata:
  name: llama-3-70b
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-70B-Instruct"
  inferenceClass: nvidia-h100-4gpu
  features: [TextGeneration]
  # Developer overrides only the scaling upper bound — hardware is not their concern
  maxReplicas: 2

---
apiVersion: kubeai.org/v1
kind: Model
metadata:
  name: bge-embed
spec:
  url: "hf://BAAI/bge-large-en-v1.5"
  inferenceClass: cpu-embedding
  features: [TextEmbedding]
```

### What the developer no longer writes

| Before | After |
|---|---|
| `engine: VLLM` | Comes from InferenceClass |
| `resourceProfile: nvidia-gpu-l4:1` | `inferenceClass: nvidia-l4-single` |
| `args: [--gpu-memory-utilization=0.95]` | Default in InferenceClass (overridable) |
| Tolerations, nodeSelector | Comes from InferenceClass |
| `targetRequests: 100` | Default in InferenceClass (overridable) |
| `scaleDownDelaySeconds: 30` | Default in InferenceClass (overridable) |

### Override Semantics

The Model can override a subset of InferenceClass fields. Platform-controlled fields (hardware, DRA, engine image) cannot be overridden by the Model:

| Field | Overridable by Model? | Rationale |
|---|---|---|
| `nodeSelector`, `tolerations` | No | Hardware placement is platform-controlled |
| `resources.limits` (GPU) | No | Hardware allocation is platform-controlled |
| `dra` | No | DRA setup is platform-controlled |
| `engine.type` | No | Platform chooses the approved engine |
| `engine.image` | Yes (via `model.spec.image`) | Allows pinning a specific version |
| `engine.args` | Yes (appended via `model.spec.args`) | Developer adds model-specific flags |
| `engine.env` | Yes (merged via `model.spec.env`) | Developer adds model-specific env |
| `scaling.minReplicas` | Yes | Developer controls scale-to-zero |
| `scaling.maxReplicas` | Yes (bounded by InferenceClass max) | Developer sets their ceiling |
| `scaling.targetRequests` | Yes | Developer tunes for their workload |
| `scaling.scaleDownDelaySeconds` | Yes | Developer tunes for their workload |

Model `args` are **appended** to InferenceClass `engine.args`, not replaced. This allows the platform to enforce required flags (e.g., `--gpu-memory-utilization`) while developers add model-specific ones (e.g., `--max-model-len=8192`).

Model `env` entries are **merged** with InferenceClass `engine.env`, with Model values taking precedence on key collision.

### InferenceClass Go Type

```go
// api/k8s/v1/inferenceclass_types.go

// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:subresource:status
type InferenceClass struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   InferenceClassSpec   `json:"spec,omitempty"`
    Status InferenceClassStatus `json:"status,omitempty"`
}

type InferenceClassSpec struct {
    // NodeSelector constrains which nodes Pods may be scheduled on.
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`
    // Tolerations applied to Pods.
    Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
    // Affinity rules for Pod scheduling.
    Affinity *corev1.Affinity `json:"affinity,omitempty"`
    // Resources for the server container.
    // GPU limits here are mutually exclusive with DRA.
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`
    // DRA configures Dynamic Resource Allocation.
    // Mutually exclusive with GPU entries in Resources.Limits.
    DRA *DRAConfig `json:"dra,omitempty"`
    // RuntimeClassName for the Pod.
    RuntimeClassName *string `json:"runtimeClassName,omitempty"`
    // SchedulerName for the Pod.
    SchedulerName string `json:"schedulerName,omitempty"`
    // Engine configures the inference engine defaults.
    Engine InferenceClassEngine `json:"engine"`
    // Scaling configures default autoscaling parameters.
    Scaling InferenceClassScaling `json:"scaling,omitempty"`
}

type InferenceClassEngine struct {
    // Type is the engine to use. Required.
    // +kubebuilder:validation:Enum=OLlama;VLLM;FasterWhisper;Infinity;SGLang
    Type string `json:"type"`
    // Image overrides the default engine image for this class.
    Image string `json:"image,omitempty"`
    // Args are baseline engine arguments prepended before Model args.
    Args []string `json:"args,omitempty"`
    // Env are baseline environment variables. Model env takes precedence on collision.
    Env map[string]string `json:"env,omitempty"`
}

type InferenceClassScaling struct {
    // MinReplicas default. Model may override.
    // +kubebuilder:default=0
    MinReplicas int32 `json:"minReplicas"`
    // MaxReplicas default. Model may override downward but not upward.
    MaxReplicas *int32 `json:"maxReplicas,omitempty"`
    // TargetRequests default. Model may override.
    // +kubebuilder:default=100
    TargetRequests int32 `json:"targetRequests"`
    // ScaleDownDelaySeconds default. Model may override.
    // +kubebuilder:default=30
    ScaleDownDelaySeconds int64 `json:"scaleDownDelaySeconds"`
}

type InferenceClassStatus struct {
    // ModelsCount is the number of Models currently referencing this class.
    ModelsCount int32 `json:"modelsCount,omitempty"`
    Conditions  []metav1.Condition `json:"conditions,omitempty"`
}
```

### Model CRD Changes

`inferenceClass` replaces `resourceProfile` and `engine`. `resourceProfile` is kept as a deprecated alias during the migration window:

```go
type ModelSpec struct {
    URL             string      `json:"url"`
    Features        []ModelFeature `json:"features"`

    // InferenceClass references the cluster-scoped InferenceClass to use.
    // Mutually exclusive with resourceProfile.
    // +kubebuilder:validation:Optional
    InferenceClass string `json:"inferenceClass,omitempty"`

    // Engine is set from the referenced InferenceClass when inferenceClass is used.
    // Required when resourceProfile is used (legacy path).
    // +kubebuilder:validation:Enum=OLlama;VLLM;FasterWhisper;Infinity;SGLang
    Engine string `json:"engine,omitempty"`

    // ResourceProfile is the legacy hardware selector. Use inferenceClass instead.
    // +kubebuilder:validation:Optional
    // Deprecated: use inferenceClass.
    ResourceProfile string `json:"resourceProfile,omitempty"`

    // Overrides (apply on top of InferenceClass defaults):
    Image             string              `json:"image,omitempty"`
    Args              []string            `json:"args,omitempty"`
    Env               map[string]string   `json:"env,omitempty"`
    EnvFrom           []corev1.EnvFromSource `json:"envFrom,omitempty"`
    MinReplicas        int32               `json:"minReplicas"`
    MaxReplicas        *int32              `json:"maxReplicas,omitempty"`
    TargetRequests     *int32              `json:"targetRequests"`
    ScaleDownDelaySeconds *int64           `json:"scaleDownDelaySeconds"`
    // ... adapters, files, loadBalancing, priorityClassName unchanged
}
```

CEL validation:
```go
// +kubebuilder:validation:XValidation:rule="has(self.inferenceClass) != (has(self.resourceProfile) && self.resourceProfile != '')",message="exactly one of inferenceClass or resourceProfile must be set."
// +kubebuilder:validation:XValidation:rule="!has(self.inferenceClass) || !has(self.engine) || self.engine == ''",message="engine must not be set when inferenceClass is used."
```

### Controller Changes

`getModelConfig` resolves either path into the same `ModelConfig`:

```go
func (r *ModelReconciler) getModelConfig(model *kubeaiv1.Model) (ModelConfig, error) {
    if model.Spec.InferenceClass != "" {
        return r.getModelConfigFromInferenceClass(model)
    }
    // Legacy path: resourceProfile + engine (unchanged)
    return r.getModelConfigFromResourceProfile(model)
}

func (r *ModelReconciler) getModelConfigFromInferenceClass(model *kubeaiv1.Model) (ModelConfig, error) {
    var ic kubeaiv1.InferenceClass
    if err := r.Get(ctx, types.NamespacedName{Name: model.Spec.InferenceClass}, &ic); err != nil {
        return ModelConfig{}, fmt.Errorf("inferenceClass %q not found: %w", model.Spec.InferenceClass, err)
    }
    // Merge: InferenceClass base + Model overrides
    return mergeInferenceClass(&ic, model), nil
}
```

The `mergeInferenceClass` function builds a `ModelConfig` applying the override rules from the table above.

The `ModelReconciler` watches `InferenceClass` objects and re-queues all Models referencing them when they change:

```go
func (r *ModelReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&kubeaiv1.Model{}).
        Owns(&corev1.Pod{}).
        Owns(&corev1.Service{}).
        Owns(&corev1.PersistentVolumeClaim{}).
        Owns(&batchv1.Job{}).
        Watches(&kubeaiv1.InferenceClass{},
            handler.EnqueueRequestsFromMapFunc(r.modelsForInferenceClass)).
        Complete(r)
}
```

### Backwards Compatibility

The `resourceProfile` + `engine` path continues to work unchanged. No existing Models break. The migration path:

1. Platform team creates `InferenceClass` objects mirroring their existing `ResourceProfile` entries.
2. Developers optionally migrate `Model` manifests to use `inferenceClass`.
3. In a future major version, `resourceProfile` is removed.

The `config.yaml` `ResourceProfile` entries can be auto-converted to `InferenceClass` objects at startup via a migration controller, or left as-is for the legacy path.

### Affected Components

| Component | Change |
|---|---|
| `api/k8s/v1/inferenceclass_types.go` (NEW) | `InferenceClass` + `InferenceClassList` types |
| `api/k8s/v1/model_types.go` | Add `InferenceClass` field; mark `ResourceProfile`/`Engine` optional |
| `internal/modelcontroller/model_controller.go` | `getModelConfig` branches on `inferenceClass` vs `resourceProfile` |
| `internal/modelcontroller/inferenceclass.go` (NEW) | `mergeInferenceClass`, `modelsForInferenceClass` watch mapper |
| `internal/modelcontroller/model_controller.go` | `SetupWithManager` watches `InferenceClass` |
| `manifests/crds/` | New `InferenceClass` CRD; updated `Model` CRD |
| `charts/kubeai/` | Add `InferenceClass` RBAC; add example `InferenceClass` values |

## Relationships to Other Proposals

### DRA Support (`dra-support.md`)

DRA support is implemented in two phases across these two proposals:

- **Phase 1 (dra-support.md, already in progress)**: `DRAConfig` added to `ResourceProfile` in system config. The `Model` author never touches DRA; the platform engineer configures it in `config.yaml`. This is the near-term implementation.
- **Phase 2 (this proposal)**: When `InferenceClass` replaces `ResourceProfile`, `DRAConfig` migrates from `ResourceProfile.dra` to `InferenceClass.spec.dra`. The struct definition is identical; only the container changes.

The `DRAConfig` type referenced in `InferenceClassSpec` is the same struct defined in `dra-support.md` and implemented in `internal/config/system.go`. The migration is mechanical: copy the `DRA` field from each `ResourceProfile` entry to the corresponding `InferenceClass` object.

The `DRA` and `Resources.Limits` mutual-exclusion rule from the DRA proposal carries over unchanged.

### Engine Interface Refactor (`engine-interface-refactor.md`)

The Engine Interface Refactor makes the engine dispatcher a proper registry (`EngineRegistry.Get(engineType)`). InferenceClass feeds `InferenceClass.spec.engine.type` into that registry: when `getModelConfigFromInferenceClass` populates `ModelConfig.Engine`, the rest of the reconcile loop is unaffected.

InferenceClass and the Engine Interface Refactor are **independent work** that can be implemented in any order. The reconcile loop is already abstracted on `ModelConfig`; InferenceClass changes only where that config is populated.

### SGLang Engine (`sglang-engine.md`)

The SGLang proposal adds `SGLang` as an enum value on `Model.Spec.Engine`. In the InferenceClass world, the engine type moves to `InferenceClass.Spec.Engine.Type`. The InferenceClass `Engine.Type` enum already includes `SGLang`. The SGLang implementation work (the engine builder, port defaults, image defaults) is unchanged — only the field that selects it shifts from Model to InferenceClass.

When both proposals are complete, a developer serving an SGLang model writes:
```yaml
inferenceClass: sglang-h100-single   # platform team configured SGLang engine here
```
instead of:
```yaml
engine: SGLang
resourceProfile: nvidia-gpu-h100:1
```

### Multi-Node Serving (`multi-node-serving.md`)

The multi-node proposal adds `model.spec.multiNode.workerResourceProfile` to select hardware for worker Pods separately from the leader. When InferenceClass replaces ResourceProfile, this field would be renamed to `workerInferenceClass`. The semantics are identical; the source of hardware config shifts from a system config map entry to a cluster-scoped CRD object.

This rename should be made atomically: implement multi-node with `workerResourceProfile` first, then rename to `workerInferenceClass` in the same commit that adds InferenceClass.

### Gateway API Inference Extension (`gateway-api-inference-extension.md`)

No conflict. InferenceClass is a compute/hardware abstraction; the Gateway API Bridge is a traffic-routing abstraction. They are orthogonal: a Model can use both `inferenceClass: nvidia-l4-single` and be served via an `InferencePool` when `proxy.mode=external` is set.

## Design Decisions

### Why cluster-scoped?

`InferenceClass` follows the same reasoning as `StorageClass` and `RuntimeClass`: hardware topology is a cluster-level concern. A GPU node pool exists at the cluster level, not per-namespace. Cluster-scope also means developers in any namespace can reference a class without the platform team duplicating it across namespaces.

For multi-tenant isolation, RBAC on `Model` objects (not `InferenceClass`) is the right lever — restrict which classes a namespace's service account may reference via `ResourceQuota` or admission webhooks.

### Why not just rename ResourceProfile to InferenceClass?

`ResourceProfile` lives in `config.yaml` (a Helm values file). Moving it to a CRD gives:
- **RBAC**: `kubectl auth can-i get inferenceclass` works; config file entries don't have RBAC
- **GitOps**: Platform teams manage `InferenceClass` CRs in git, not Helm values
- **Discoverability**: `kubectl get inferenceclasses` shows what's available in a cluster
- **Status**: `InferenceClass.status.modelsCount` tells platform teams which classes are in use
- **Events**: Referencing a deleted `InferenceClass` produces a proper Kubernetes event on the Model

### MaxReplicas override boundary

The Model's `maxReplicas` override is bounded by the InferenceClass `maxReplicas` — a developer cannot scale beyond what the platform has approved for that hardware shape. If the InferenceClass has `maxReplicas: 8` and the Model sets `maxReplicas: 20`, the controller uses 8. This gives platform teams cost control without blocking developer autonomy.

### Args are appended, not replaced

Platform-required flags (e.g., `--gpu-memory-utilization=0.95`, security-relevant args) cannot be silently dropped by a developer. Model `args` are always appended after InferenceClass `args`. If a developer needs to override a platform arg, they use `env` overrides or a custom `image` — both are deliberate escapes.

## Implementation Phases

**Prerequisite**: `dra-support.md` Phase 1 must be complete (DRA in `ResourceProfile`). This is already in progress in the `feat/dra-resource-profile` branch.

### Phase 1: InferenceClass CRD + Controller
* New `InferenceClass` CRD with the spec above.
* `getModelConfigFromInferenceClass` and `mergeInferenceClass`.
* `InferenceClass` watch in `SetupWithManager`.
* `resourceProfile` + `engine` legacy path unchanged.
* Unit tests for merge semantics (nil dra, template dra, shared dra, scaling override bounds).
* Example `InferenceClass` manifests for each GKE/EKS/AKS GPU shape in `charts/kubeai/`.

### Phase 2: Migration Tooling
* Auto-generation of `InferenceClass` objects from `config.yaml` `ResourceProfile` entries on operator startup (one-shot migration, idempotent).
* Deprecation warning logged when `model.spec.resourceProfile` is used.
* `kubectl kubeai migrate inferenceclass` command that bulk-patches existing Models in a namespace.

### Phase 3: ResourceProfile Removal
* Remove `resourceProfile` and `engine` from `Model` CRD (major version bump).
* Remove `ResourceProfile` map from `config.yaml` / system config.
* `InferenceClass` becomes the sole hardware abstraction.
* Rename `multiNode.workerResourceProfile` → `multiNode.workerInferenceClass` (see multi-node-serving.md).

## Relevant Reading

* [Kubernetes StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* [Kubernetes RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
* [Gateway API InferencePool](https://gateway-api-inference-extension.sigs.k8s.io/api-types/inferencepool/) — complementary: InferenceClass is compute-focused, InferencePool is traffic-focused
* [llm-d modelserver recipes](https://github.com/llm-d/llm-d/tree/main/guides/recipes/modelserver) — llm-d uses Kustomize overlays for the same purpose; InferenceClass is the CRD-native equivalent
* [dra-support.md](./dra-support.md) — Phase 1 DRA implementation; `DRAConfig` struct reused by InferenceClass
* [engine-interface-refactor.md](./engine-interface-refactor.md) — Engine registry that `InferenceClass.spec.engine.type` feeds into
* [multi-node-serving.md](./multi-node-serving.md) — `workerResourceProfile` → `workerInferenceClass` rename in Phase 3
* [sglang-engine.md](./sglang-engine.md) — SGLang is an InferenceClass engine type; `model.spec.engine` removal happens in Phase 3
