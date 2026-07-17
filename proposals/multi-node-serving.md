# Multi-Node Serving (LeaderWorkerSet)

## Problem

KubeAI currently runs one Pod per replica. Each Pod can use multiple GPUs on a single node via tensor parallelism (`--tensor-parallel-size N`), but the model must fit within the GPU memory of a single machine.

For large models (Llama-3 405B, Qwen-2 72B at full precision, large MoE models), a single node with 4–8 GPUs is often insufficient. Users who need multi-node inference today must:

1. Provision very large instances (e.g., 8×H100 nodes) that are expensive and scarce.
2. Use external orchestration (Ray Serve, vLLM's `--pipeline-parallel-size`) entirely outside KubeAI.

[LeaderWorkerSet (LWS)](https://github.com/kubernetes-sigs/lws) is a Kubernetes-native API for grouped Pod sets. It creates one "group" per replica: a leader Pod (handles inference traffic) plus N worker Pods (participate in computation). LWS is now a CNCF project and has reached v0.4. llm-d uses it as its primary multi-node primitive.

## Solution

Extend the `Model` CRD with an optional `multiNode` field. When set, the `ModelReconciler` creates `LeaderWorkerSet` objects instead of bare Pods.

### Model CRD Change

```yaml
# api/k8s/v1/model_types.go (new fields)
type ModelSpec struct {
    # ... existing fields ...

    # MultiNode configures multi-node distributed serving.
    # When set, a LeaderWorkerSet is created instead of bare Pods.
    # Requires engine: VLLM. Requires LWS controller installed in the cluster.
    # +kubebuilder:validation:Optional
    MultiNode *MultiNodeConfig `json:"multiNode,omitempty"`
}

type MultiNodeConfig struct {
    # WorkerCount is the number of worker Pods per replica group (not counting the leader).
    # Total Pods per group = 1 (leader) + WorkerCount.
    # +kubebuilder:validation:Minimum=1
    # +kubebuilder:validation:Maximum=15
    WorkerCount int32 `json:"workerCount"`

    # WorkerResourceProfile is the resource profile for worker Pods.
    # If omitted, workers use the same profile as the leader (model.spec.resourceProfile).
    # +kubebuilder:validation:Optional
    WorkerResourceProfile string `json:"workerResourceProfile,omitempty"`

    # PipelineParallelSize sets --pipeline-parallel-size on vLLM.
    # Defaults to WorkerCount + 1 (one stage per node).
    # +kubebuilder:validation:Optional
    PipelineParallelSize *int32 `json:"pipelineParallelSize,omitempty"`
}
```

CEL validation rule (prevent multi-node on unsupported engines):

```go
// +kubebuilder:validation:XValidation:rule="!has(self.multiNode) || self.engine == \"VLLM\"",message="multiNode is only supported with the VLLM engine."
```

### Example: Llama-3 405B on 2×8-GPU Nodes

```yaml
kind: Model
metadata:
  name: llama-3-1-405b-instruct
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-405B-Instruct"
  engine: VLLM
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-h100:8"     # 8 GPUs on leader node
  minReplicas: 1
  maxReplicas: 3
  multiNode:
    workerCount: 1                          # 1 leader + 1 worker = 2 nodes
    workerResourceProfile: "nvidia-gpu-h100:8"
    pipelineParallelSize: 2
```

KubeAI creates 1 `LeaderWorkerSet` with `replicas: 1`, each group containing a leader + 1 worker. Total GPU count: 16 (2 nodes × 8 GPUs).

### What KubeAI Creates

For the above Model, KubeAI creates:

```yaml
apiVersion: leaderworkerset.x-k8s.io/v1
kind: LeaderWorkerSet
metadata:
  name: llama-3-1-405b-instruct
  ownerReferences:
    - apiVersion: kubeai.org/v1
      kind: Model
      name: llama-3-1-405b-instruct
spec:
  replicas: 1                               # = model.spec.replicas
  leaderWorkerTemplate:
    size: 2                                 # 1 leader + workerCount workers
    leaderTemplate:
      metadata:
        labels:
          kubeai.org/model: llama-3-1-405b-instruct
          kubeai.org/role: leader
      spec:
        # vLLM args for leader:
        # --model, --served-model-name,
        # --tensor-parallel-size=8,
        # --pipeline-parallel-size=2,
        # --distributed-executor-backend=ray (or mp)
        containers: [ ... ]
    workerTemplate:
      metadata:
        labels:
          kubeai.org/model: llama-3-1-405b-instruct
          kubeai.org/role: worker
      spec:
        containers: [ ... ]
  rolloutStrategy:
    type: RollingUpdate
    rollingUpdateConfiguration:
      maxUnavailable: 1
      maxSurge: 1
```

### Request Routing

In internal proxy mode, the load balancer continues to route requests to **leader Pod IPs**. Worker Pods are not in the endpoint pool. The `LoadBalancer` reconciler already filters by `kubeai.org/model` label — adding `kubeai.org/role: leader` to the filter excludes workers automatically.

In external proxy mode, the headless Service selector includes `kubeai.org/role: leader` to expose only leader Pods.

### Autoscaling

The autoscaler patches `model.spec.replicas`, which the controller translates to `LeaderWorkerSet.spec.replicas`. The scale unit is a group (1 leader + N workers), not individual Pods. This is correct: adding a group adds a full replica capable of serving a request.

### Rollout

LWS has built-in rolling update support. KubeAI passes the Pod spec hash as a LWS label to detect when a new Pod template is needed, same as with bare Pods today.

### Affected Components

| Component | Change |
|---|---|
| `api/k8s/v1/model_types.go` | Add `MultiNode` / `MultiNodeConfig` fields, CEL validation |
| `internal/modelcontroller/model_controller.go` | Branch reconcile path on `model.Spec.MultiNode != nil` |
| `internal/modelcontroller/lws.go` (NEW) | `reconcileLWS`: create/update/delete LeaderWorkerSet |
| `internal/modelcontroller/engine_vllm.go` | Generate leader + worker Pod specs with PP/TP flags |
| `internal/modelcontroller/engine.go` | Add `SupportsMultiNode() bool` to `Engine` interface |
| `internal/loadbalancer/load_balancer.go` | Filter endpoint pods by `kubeai.org/role: leader` when LWS is in use |
| `internal/modelcontroller/model_controller.go` | `SetupWithManager`: `Owns(&lwsv1.LeaderWorkerSet{})` |
| `charts/kubeai/` | RBAC for `leaderworkersets`; LWS as optional CRD dependency |
| `go.mod` | Add `sigs.k8s.io/lws` dependency |

## Design Decisions

### Why LWS instead of bare Pods + headless Service for worker discovery?

vLLM's pipeline parallelism requires the leader to know all worker addresses before startup (`MASTER_ADDR`, `MASTER_PORT`, worker ranks). LWS automatically injects these via environment variables (`LWS_LEADER_ADDRESS`, pod index). Replicating this with bare Pods requires a Init-container + service discovery hack. LWS is the right primitive.

### Why not use Ray?

Ray Serve is a valid option but introduces a heavy dependency (Ray cluster, separate operator). LWS is purpose-built for this pattern and requires only the LWS controller (a single small Deployment). KubeAI's positioning as a "zero external dependencies" operator makes LWS the better fit.

### Backwards compatibility

`multiNode` is optional. All existing Models without this field behave identically. The new LWS code path is completely separate from the existing Pod management path.

### LWS as an optional dependency

LWS CRD and controller are not required unless a Model sets `multiNode`. If a Model with `multiNode` is submitted to a cluster without LWS installed, the reconciler returns an error event on the Model, not a panic.

## Implementation Phases

### Phase 1: Basic Multi-Node (Pipeline Parallelism)
* `MultiNodeConfig` in CRD.
* `reconcileLWS` creating LeaderWorkerSet with fixed `replicas`.
* vLLM leader/worker Pod specs with `--pipeline-parallel-size` and `--tensor-parallel-size`.
* Leader-only endpoint routing in load balancer.
* Integration test with envtest (LWS CRDs can be registered in envtest without a real LWS controller).

### Phase 2: Autoscaling
* Patch `LeaderWorkerSet.spec.replicas` via the autoscaler.
* Autoscaler must count active requests from leader Pods only.
* Scale-from-zero: LWS supports `replicas: 0`; proxy queuing behavior unchanged.

### Phase 3: Heterogeneous Leaders and Workers
* `workerResourceProfile` allows workers to have different GPU types than the leader.
* Example: H100 leader (prefill-heavy) with H100 workers (all decode).
* Interaction with [Prefill/Decode Disaggregation](./prefill-decode-disaggregation.md): a multi-node disaggregated model would use LWS for multi-node within each phase.

## Relevant Reading

* [LeaderWorkerSet GitHub](https://github.com/kubernetes-sigs/lws)
* [vLLM Distributed Inference](https://docs.vllm.ai/en/latest/serving/distributed_serving.html)
* [llm-d LWS Usage](https://github.com/llm-d/llm-d/blob/main/guides/multi-node.md)
* [LWS vLLM Example](https://github.com/kubernetes-sigs/lws/tree/main/docs/examples/vllm)
