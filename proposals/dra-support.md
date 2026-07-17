# Dynamic Resource Allocation (DRA) Support

## Problem

KubeAI allocates GPUs using the legacy device plugin model: `resources.limits["nvidia.com/gpu": "N"]`. This approach has hard limits that matter more as GPU clusters grow:

* **No MIG awareness**: NVIDIA MIG (Multi-Instance GPU) partitions (e.g., `1g.10gb`) cannot be expressed as simple integer limits. Users must pick the right node via `nodeSelector` manually.
* **No time-slicing control**: Kubernetes device plugins expose time-sliced GPUs as if they were real GPUs; there is no way to express priority or quota within a slice.
* **No cross-vendor composability**: A model that needs "1 GPU of at least 80GB VRAM" must be expressed as a vendor-specific resource name today. DRA enables abstract capacity requests.
* **No structured parameters**: Device plugins have no mechanism to pass per-allocation config (e.g., ECC mode, compute mode) to the driver. DRA `DeviceParameters` enables this.

Kubernetes 1.32 graduated DRA to stable for the `resource.k8s.io/v1beta1` API. Major GPU vendors (NVIDIA, AMD, Intel) are shipping DRA drivers alongside or replacing their device plugins.

This proposal depends on [Engine Interface Refactor](./engine-interface-refactor.md) for clean per-engine gating. It is backwards-compatible: device plugin behavior is the default.

## Solution

Extend `ResourceProfile` in the system config to support a `dra` mode. When a model's `resourceProfile` resolves to a DRA-backed profile, the controller generates `ResourceClaim` objects instead of `resources.limits` entries.

### System Config Change

```yaml
# config.yaml
resourceProfiles:
  # Existing device-plugin profile (unchanged)
  nvidia-gpu-l4:
    imageName: "nvidia-gpu"
    requests:
      cpu: "6"
      memory: "24Gi"
    limits:
      nvidia.com/gpu: "1"
    tolerations:
      - key: nvidia.com/gpu
        value: present
        effect: NoSchedule
    nodeSelector:
      cloud.google.com/gke-accelerator: nvidia-l4

  # New DRA-backed profile
  nvidia-gpu-h100-dra:
    imageName: "nvidia-gpu"
    requests:
      cpu: "12"
      memory: "80Gi"
    dra:
      deviceClassName: "gpu.nvidia.com"
      selectors:
        - cel:
            expression: "device.attributes['memory'].isGreaterThan(quantity('79Gi'))"
      config:
        - opaque:
            driver: "gpu.nvidia.com"
            parameters:
              apiVersion: gpu.nvidia.com/v1alpha1
              kind: GpuConfig
              sharing:
                strategy: TimeSlicing
                timeSlicingConfig:
                  interval: Long

  # MIG profile: request a 3g.40gb partition specifically
  nvidia-mig-3g-40gb:
    imageName: "nvidia-gpu"
    requests:
      cpu: "4"
      memory: "48Gi"
    dra:
      deviceClassName: "mig.nvidia.com"
      selectors:
        - cel:
            expression: "device.attributes['profile'] == '3g.40gb'"
```

### Model CRD (No Change)

The `Model` CRD does not change. Users continue to specify:

```yaml
spec:
  resourceProfile: "nvidia-gpu-h100-dra:1"
```

The `:<count>` multiplier is interpreted as the number of DRA device requests when in DRA mode.

### ResourceClaim Lifecycle

The controller creates one `ResourceClaim` per model Pod (not per model), owned by the Pod:

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  name: llama-3-1-70b-instruct-<pod-hash>-gpu
  ownerReferences:
    - apiVersion: v1
      kind: Pod
      name: model-llama-3-1-70b-instruct-<hash>
spec:
  devices:
    requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
        selectors:
          - cel:
              expression: "device.attributes['memory'].isGreaterThan(quantity('79Gi'))"
    config:
      - opaque:
          driver: gpu.nvidia.com
          parameters: { ... }
```

The Pod references the claim via `spec.resourceClaims`:

```yaml
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: llama-3-1-70b-instruct-<pod-hash>-gpu
  containers:
    - name: server
      resources:
        claims:
          - name: gpu
```

### `ResourceProfile` Go Type Change

```go
// internal/config/system.go

type ResourceProfile struct {
    // Existing fields (unchanged)
    ImageName      string                      `json:"imageName"`
    Requests       corev1.ResourceList         `json:"requests,omitempty"`
    Limits         corev1.ResourceList         `json:"limits,omitempty"`
    NodeSelector   map[string]string           `json:"nodeSelector,omitempty"`
    Affinity       *corev1.Affinity            `json:"affinity,omitempty"`
    Tolerations    []corev1.Toleration         `json:"tolerations,omitempty"`
    RuntimeClass   *string                     `json:"runtimeClassName,omitempty"`
    SchedulerName  string                      `json:"schedulerName,omitempty"`

    // New: DRA mode. Mutually exclusive with Limits["nvidia.com/gpu"].
    DRA *DRAConfig `json:"dra,omitempty"`
}

type DRAConfig struct {
    DeviceClassName string                    `json:"deviceClassName"`
    Selectors       []resourcev1b1.DeviceSelector `json:"selectors,omitempty"`
    Config          []resourcev1b1.DeviceClaimConfiguration `json:"config,omitempty"`
}
```

### Controller Changes

In `pod_plan.go`, after building the pod via the engine builder, apply DRA if configured:

```go
// pod_plan.go
podForModel := engine.PodForModel(model, modelConfig)

if modelConfig.ResourceProfile.DRA != nil && engine.SupportsDRA() {
    if err := r.applyDRAToPod(ctx, model, podForModel, modelConfig); err != nil {
        return nil, fmt.Errorf("applying DRA config: %w", err)
    }
}
```

`applyDRAToPod` creates the `ResourceClaim` and patches `pod.Spec.ResourceClaims` and `container.Resources.Claims`.

### DRA Claim Cleanup

`ResourceClaim` objects owned by Pods are automatically deleted when the Pod is deleted (via `ownerReference` GC). The controller watches `ResourceClaim` objects for its owned Pods to update status if allocation fails.

### Affected Components

| Component | Change |
|---|---|
| `internal/config/system.go` | Add `DRAConfig` struct to `ResourceProfile` |
| `internal/modelcontroller/pod_plan.go` | Apply DRA config after engine builder |
| `internal/modelcontroller/dra.go` (NEW) | `applyDRAToPod`: create `ResourceClaim`, patch Pod spec |
| `internal/modelcontroller/model_controller.go` | Add `Owns(&resourcev1b1.ResourceClaim{})` |
| `internal/modelcontroller/engine.go` | Add `SupportsDRA() bool` to `Engine` interface |
| All engine implementations | Implement `SupportsDRA()` (vLLM: true, Ollama: false, etc.) |
| `charts/kubeai/` | RBAC: add `resourceclaims` get/create/delete permissions |
| `go.mod` | Ensure `k8s.io/api` ≥ 0.32 for `resource.k8s.io/v1beta1` |

## Design Decisions

### Why per-Pod claims, not per-Model?

DRA claims are allocated at Pod scheduling time by the scheduler. A single claim shared by multiple Pods cannot work because the scheduler allocates the claim once. Each Pod needs its own claim to get its own GPU allocation.

### Why not use `ResourceClaimTemplate`?

`ResourceClaimTemplate` + Pod template is the standard pattern for workloads (Deployments, StatefulSets). KubeAI manages Pods directly (not via Deployment), so using a template would require creating a separate `ResourceClaimTemplate` object per model *and* wiring it into each Pod's `resourceClaims[].resourceClaimTemplateName`. Creating the claim directly per Pod is simpler and keeps the ownership chain clean.

### Backwards compatibility

If `DRA` is nil in the profile, behavior is identical to today. No existing models or configs break.

## Design Revision: ResourceClaimTemplate over ResourceClaim

After studying llm-d's implementation, the primary DRA pattern for exclusive GPU access (the common case) is `ResourceClaimTemplate`, not a manually-created `ResourceClaim`. The scheduler creates one `ResourceClaim` per Pod from the template automatically. A pre-created shared `ResourceClaim` is the secondary path, only for MPS/time-slicing pools.

This also means KubeAI does **not** need to create or delete `ResourceClaim` objects in the controller. The lifecycle is handled by the scheduler (template path) or pre-exists (shared path). The controller only needs to patch the Pod spec.

---

## Implementation Plan

This plan replaces PR #642 with the correct abstraction: DRA config lives in the system-level `ResourceProfile`, not in the `Model` CRD. The `Model` author never touches DRA.

### Step 1: `internal/config/system.go`

Add `DRAConfig` to `ResourceProfile` and apply defaults in `DefaultAndValidate()`.

```go
type ResourceProfile struct {
    ImageName        string              `json:"imageName"`
    Requests         corev1.ResourceList `json:"requests,omitempty"`
    Limits           corev1.ResourceList `json:"limits,omitempty"`
    NodeSelector     map[string]string   `json:"nodeSelector,omitempty"`
    Affinity         *corev1.Affinity    `json:"affinity,omitempty"`
    Tolerations      []corev1.Toleration `json:"tolerations,omitempty"`
    SchedulerName    string              `json:"schedulerName,omitempty"`
    RuntimeClassName *string             `json:"runtimeClassName,omitempty"`
    // DRA is mutually exclusive with GPU entries in Limits.
    DRA *DRAConfig `json:"dra,omitempty"`
}

type DRAConfig struct {
    // ResourceClaimTemplateName names a pre-created ResourceClaimTemplate.
    // The scheduler creates one ResourceClaim per Pod (exclusive allocation).
    // Mutually exclusive with ResourceClaimName.
    ResourceClaimTemplateName string `json:"resourceClaimTemplateName,omitempty"`

    // ResourceClaimName names a pre-created shared ResourceClaim.
    // All Pods of a Model share this single claim (MPS / time-slicing pools).
    // Mutually exclusive with ResourceClaimTemplateName.
    ResourceClaimName string `json:"resourceClaimName,omitempty"`

    // ClaimName is the local name for this claim inside the Pod spec.
    // Defaults to "gpu-claim".
    ClaimName string `json:"claimName,omitempty"`

    // ClaimRequest is the request name within the claim.
    // Defaults to "gpu".
    ClaimRequest string `json:"claimRequest,omitempty"`
}
```

In `DefaultAndValidate()`, after the existing `CacheProfiles` loop:

```go
for name, profile := range s.ResourceProfiles {
    if profile.DRA != nil {
        if profile.DRA.ClaimName == "" {
            profile.DRA.ClaimName = "gpu-claim"
        }
        if profile.DRA.ClaimRequest == "" {
            profile.DRA.ClaimRequest = "gpu"
        }
        if profile.DRA.ResourceClaimTemplateName != "" && profile.DRA.ResourceClaimName != "" {
            return fmt.Errorf("resourceProfile %q: dra.resourceClaimTemplateName and dra.resourceClaimName are mutually exclusive", name)
        }
        if profile.DRA.ResourceClaimTemplateName == "" && profile.DRA.ResourceClaimName == "" {
            return fmt.Errorf("resourceProfile %q: dra requires either resourceClaimTemplateName or resourceClaimName", name)
        }
    }
    s.ResourceProfiles[name] = profile
}
```

### Step 2: `api/k8s/v1/model_types.go`

Remove the `ResourceClaimName` field added by PR #642. The `Model` CRD is unchanged from before that PR.

```go
// DELETE this field:
// ResourceClaimName string `json:"resourceClaimName,omitempty"`
```

Regenerate CRD manifests: `make manifests generate`.

### Step 3: `internal/modelcontroller/resource_claims.go`

Replace the current function signature (which takes `*kubeaiv1.Model`) with one that takes `*config.DRAConfig`. Handle both the template and shared-claim paths.

```go
package modelcontroller

import (
    "github.com/kubeai-project/kubeai/internal/config"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/utils/ptr"
)

// applyResourceClaims patches pod to use DRA instead of device plugin limits.
// It is a no-op when dra is nil (device plugin profile).
func applyResourceClaims(pod *corev1.Pod, dra *config.DRAConfig) {
    if dra == nil {
        return
    }

    podClaim := corev1.PodResourceClaim{Name: dra.ClaimName}
    switch {
    case dra.ResourceClaimTemplateName != "":
        podClaim.ResourceClaimTemplateName = ptr.To(dra.ResourceClaimTemplateName)
    case dra.ResourceClaimName != "":
        podClaim.ResourceClaimName = ptr.To(dra.ResourceClaimName)
    }

    // Upsert into pod.Spec.ResourceClaims.
    updated := false
    for i := range pod.Spec.ResourceClaims {
        if pod.Spec.ResourceClaims[i].Name == dra.ClaimName {
            pod.Spec.ResourceClaims[i] = podClaim
            updated = true
            break
        }
    }
    if !updated {
        pod.Spec.ResourceClaims = append(pod.Spec.ResourceClaims, podClaim)
    }

    // Upsert container claim reference on the server container only.
    for i := range pod.Spec.Containers {
        if pod.Spec.Containers[i].Name != serverContainerName {
            continue
        }
        claims := pod.Spec.Containers[i].Resources.Claims
        claimUpdated := false
        for j := range claims {
            if claims[j].Name == dra.ClaimName {
                claims[j].Request = dra.ClaimRequest
                claimUpdated = true
                break
            }
        }
        if !claimUpdated {
            claims = append(claims, corev1.ResourceClaim{
                Name:    dra.ClaimName,
                Request: dra.ClaimRequest,
            })
        }
        pod.Spec.Containers[i].Resources.Claims = claims
    }
}
```

### Step 4: `internal/modelcontroller/pod_plan.go`

One-line change at the call site (currently `applyResourceClaimsForModel(podForModel, model)`):

```go
// Before:
applyResourceClaimsForModel(podForModel, model)

// After:
applyResourceClaims(podForModel, modelConfig.ResourceProfile.DRA)
```

`modelConfig.ResourceProfile` is already populated by `getModelConfig` (line 384 of `model_controller.go`). The `DRA` field rides along for free.

### Step 5: `internal/modelcontroller/resource_claims_test.go`

Update the test to pass `*config.DRAConfig` instead of `*v1.Model`:

```go
func Test_applyResourceClaims_template(t *testing.T) {
    dra := &config.DRAConfig{
        ResourceClaimTemplateName: "nvidia-h100-exclusive",
        ClaimName:                 "gpu-claim",
        ClaimRequest:              "gpu",
    }
    pod := &corev1.Pod{ /* ... server container ... */ }
    applyResourceClaims(pod, dra)

    require.Equal(t, "gpu-claim", pod.Spec.ResourceClaims[0].Name)
    require.Equal(t, "nvidia-h100-exclusive", *pod.Spec.ResourceClaims[0].ResourceClaimTemplateName)
    require.Nil(t, pod.Spec.ResourceClaims[0].ResourceClaimName)
    // container claim ref
    require.Equal(t, "gpu-claim", pod.Spec.Containers[0].Resources.Claims[0].Name)
    require.Equal(t, "gpu", pod.Spec.Containers[0].Resources.Claims[0].Request)
}

func Test_applyResourceClaims_shared(t *testing.T) {
    dra := &config.DRAConfig{
        ResourceClaimName: "nvidia-mps-shared",
        ClaimName:         "gpu-claim",
        ClaimRequest:      "gpu",
    }
    // ... same assertions but ResourceClaimName set, ResourceClaimTemplateName nil
}

func Test_applyResourceClaims_nil(t *testing.T) {
    pod := &corev1.Pod{ /* ... */ }
    applyResourceClaims(pod, nil) // must be no-op
    require.Empty(t, pod.Spec.ResourceClaims)
}
```

### Step 6: `charts/kubeai/values.yaml`

Remove DRA from the model chart; add example DRA profiles to the operator chart:

```yaml
# values.yaml - resourceProfiles section
resourceProfiles:
  # Standard device-plugin profiles (unchanged)
  nvidia-gpu-l4:
    imageName: nvidia-gpu
    requests: { cpu: "6", memory: "24Gi" }
    limits:
      nvidia.com/gpu: "1"

  # DRA profile - exclusive per-Pod via ResourceClaimTemplate (primary path)
  # Mirrors llm-d's Gaudi/XPU pattern. User must pre-create the template.
  nvidia-gpu-h100-dra:
    imageName: nvidia-gpu
    requests: { cpu: "12", memory: "80Gi" }
    tolerations:
      - key: nvidia.com/gpu
        effect: NoSchedule
    dra:
      resourceClaimTemplateName: "nvidia-h100-exclusive"
      # claimName defaults to "gpu-claim"
      # claimRequest defaults to "gpu"

  # DRA profile: shared claim (MPS / time-slicing)
  nvidia-gpu-mps:
    imageName: nvidia-gpu
    requests: { cpu: "4", memory: "20Gi" }
    dra:
      resourceClaimName: "nvidia-mps-shared"
```

Remove from `charts/models/templates/models.yaml` (the per-model helm chart):
```yaml
# DELETE:
# resourceClaimName: {{ .Values.resourceClaimName }}
```

### Step 7: RBAC

No new RBAC needed. KubeAI does not create or delete `ResourceClaim` objects in this design: the scheduler handles it for template-based claims, and shared claims are pre-created by the cluster admin. The existing RBAC is sufficient.

### Execution order

```
1. internal/config/system.go          DRAConfig struct + DefaultAndValidate
2. internal/modelcontroller/          resource_claims.go + resource_claims_test.go
3. internal/modelcontroller/          pod_plan.go (one line)
4. api/k8s/v1/model_types.go          remove ResourceClaimName field
5. make manifests generate            regenerate CRD YAML
6. charts/                            values.yaml + models chart cleanup
```

Steps 1–3 are independent of each other and can be done in any order. Step 4 must come after step 3 (otherwise the old call site breaks). Step 5 must follow step 4.

---

## Implementation Phases

### Phase 1: ResourceProfile-based DRA (this plan)
* `DRAConfig` in system config with defaults and validation.
* `ResourceClaimTemplate` path (exclusive) + `ResourceClaim` path (shared MPS).
* No `Model` CRD changes
* Unit tests covering both paths and the nil no-op.

### Phase 2: Multi-device count
* Respect the `:<N>` multiplier from `resourceProfile: "profile:N"` as the device `count` in the claim.
* Requires the `ResourceClaimTemplate` to be authored without a fixed `count`, relying on KubeAI to set it per-Pod via a generated `ResourceClaim` (switches from template reference to KubeAI-created claim with count).
* More complex; defer to Phase 2.

### Phase 3: MIG / structured device selectors
* Expose `deviceSelectors` (CEL expressions) in `DRAConfig` so cluster admins can express constraints like `device.attributes['memory'].isGreaterThan(quantity('79Gi'))`.
* Allows KubeAI to create `ResourceClaim` objects directly with precise selectors, rather than requiring a pre-created template.

## Relevant Reading

* [Kubernetes DRA Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
* [NVIDIA DRA Driver](https://github.com/NVIDIA/k8s-dra-driver-gpu)
* [Intel Resource Drivers for Kubernetes](https://github.com/intel/intel-resource-drivers-for-kubernetes)
* [llm-d Gaudi DRA example](https://github.com/llm-d/llm-d/blob/main/guides/optimized-baseline/modelserver/hpu/vllm/resource-claim-template.yaml)
* [KubeAI Issue #639](https://github.com/substratusai/kubeai/issues/639)
* [KubeAI PR #642](https://github.com/substratusai/kubeai/pull/642)
* [resource.k8s.io/v1beta1 API Reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/resource-claim-v1beta1/)
