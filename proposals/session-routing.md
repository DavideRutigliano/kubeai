# Session-Based Routing

## Problem

KubeAI's two load balancing strategies — `LeastLoad` (route to least-loaded Pod) and `PrefixHash` (CHWBL based on prompt prefix) — are stateless. Every request is routed independently.

This is suboptimal for two classes of workloads (see [Issue #509](https://github.com/substratusai/kubeai/issues/509)):

1. **LoRA adapter affinity**: A user is chatting with a fine-tuned adapter (e.g., `sarcasm`). vLLM loads adapters lazily; the first request to a Pod that hasn't loaded the adapter pays a loading penalty (seconds to minutes for large adapters). If subsequent requests in the same conversation route to a different Pod, the penalty is paid again. Today CHWBL routes by prompt prefix, which is not stable across turns (the prefix grows each turn).

2. **KV-cache reuse for multi-turn conversations**: Modern LLMs benefit from KV-cache reuse for long conversations. vLLM's prefix caching can reuse the entire conversation history if the same Pod handles all turns. CHWBL achieves this for *identical* prefixes but not for *growing* prefixes (each turn adds tokens, changing the hash).

Neither case is served by `PrefixHash` (which re-hashes each turn's full prefix) or `LeastLoad` (which ignores history entirely).

## Solution

Add a `Session` load balancing strategy. When selected, the proxy extracts a session key from the request and maintains a sticky mapping of `session_key → Pod endpoint`. The mapping has a configurable TTL and degrades gracefully when the target Pod dies or becomes overloaded.

### Model CRD Change

```go
// api/k8s/v1/model_types.go

// +kubebuilder:validation:Enum=LeastLoad;PrefixHash;Session
type LoadBalancingStrategy string

const (
    LeastLoadStrategy  LoadBalancingStrategy = "LeastLoad"
    PrefixHashStrategy LoadBalancingStrategy = "PrefixHash"
    SessionStrategy    LoadBalancingStrategy = "Session"  // NEW
)

type LoadBalancing struct {
    // +kubebuilder:default=LeastLoad
    Strategy LoadBalancingStrategy `json:"strategy,omitempty"`

    PrefixHash PrefixHash `json:"prefixHash,omitempty"`

    Session Session `json:"session,omitempty"` // NEW
}

type Session struct {
    // HeaderName is the HTTP header used as the session key.
    // Default: "X-Session-Id"
    // +kubebuilder:default="X-Session-Id"
    HeaderName string `json:"headerName,omitempty"`

    // TTL is how long a session mapping is kept without activity.
    // After TTL expires without a matching request, the mapping is removed
    // and the next request is routed by LeastLoad.
    // +kubebuilder:default="5m"
    TTL metav1.Duration `json:"ttl,omitempty"`

    // MaxLoadFactor is the fraction above mean load at which the session
    // target is abandoned in favor of LeastLoad routing.
    // 0 disables the overload check (strict affinity).
    // +kubebuilder:default=200
    // +kubebuilder:validation:Minimum=0
    // +kubebuilder:validation:Maximum=1000
    MaxLoadFactor int32 `json:"maxLoadFactor,omitempty"`
}
```

### Example Models

```yaml
# LoRA model with session routing for adapter affinity
kind: Model
metadata:
  name: llama-3-1-8b-finetuned
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-8B-Instruct"
  engine: VLLM
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-l4:1"
  adapters:
    - name: sarcasm
      url: "hf://myrepo/sarcasm-lora"
    - name: formal
      url: "hf://myrepo/formal-lora"
  loadBalancing:
    strategy: Session
    session:
      headerName: "X-Session-Id"
      ttl: "10m"
      maxLoadFactor: 150  # abandon stickiness if target Pod is 150% above mean

# Multi-turn conversation model
kind: Model
metadata:
  name: gpt-4o-compatible
spec:
  url: "hf://meta-llama/Meta-Llama-3.1-70B-Instruct"
  engine: VLLM
  features: [TextGeneration]
  resourceProfile: "nvidia-gpu-h100:2"
  loadBalancing:
    strategy: Session
    session:
      headerName: "X-User-Id"
      ttl: "30m"
```

### Client Usage

```bash
# First request creates a session binding
curl http://kubeai:8000/openai/v1/chat/completions \
  -H "X-Session-Id: user-abc-conv-1" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama-3-1-8b-finetuned", "messages": [...], "model_suffix": "sarcasm"}'

# Subsequent requests reuse the same Pod
curl http://kubeai:8000/openai/v1/chat/completions \
  -H "X-Session-Id: user-abc-conv-1" \
  -d '{"model": "llama-3-1-8b-finetuned", "messages": [...], "model_suffix": "sarcasm"}'
```

### Session Table Design

The session table lives in the `group` struct (per model), protected by the existing `endpointsMtx`:

```go
// internal/loadbalancer/group.go

type group struct {
    // ... existing fields ...

    // sessions maps session key → endpoint address.
    // Entries expire after session.TTL of inactivity.
    sessions map[string]*sessionEntry
}

type sessionEntry struct {
    endpoint    string
    lastUsed    time.Time
}
```

Session table is **in-memory only** — it is not persisted across KubeAI restarts. On restart, the next request for each session key routes by LeastLoad and re-establishes affinity. This is acceptable because:
- The penalty is a single re-routing event, not a correctness failure.
- Persisting session state adds a dependency (Redis/etcd) that conflicts with KubeAI's zero-dependency philosophy.

### Routing Algorithm

```
PickEndpoint(req) for Session strategy:

1. Extract session key from req.Header.Get(model.Spec.LoadBalancing.Session.HeaderName)
2. If key is empty → fall back to LeastLoad for this request (no session created)
3. Look up sessions[key]:
   a. Entry found, endpoint is alive:
      - Check endpoint load vs mean load × maxLoadFactor:
        - If within threshold: return endpoint, update lastUsed
        - If overloaded: evict entry, fall through to step 4
   b. Entry not found (new session or TTL expired):
      - Fall through to step 4
4. Route by LeastLoad → selected endpoint
5. Record sessions[key] = {endpoint, time.Now()}
6. Return endpoint
```

### Session Expiry

A background goroutine in `group` runs every `min(TTL/2, 1m)` and removes entries where `time.Since(lastUsed) > TTL`. This keeps the session table bounded.

The goroutine is started in `group.init()` and stopped via the group's context when the model is deleted.

### Interaction with Adapter Routing

When `Session` strategy is active and the request contains a `model_suffix` (adapter name), the session routing provides a secondary benefit: the chosen Pod (sticky for the session) is also the Pod where the adapter was last loaded. The adapter management in `adapters.go` already labels Pods with `kubeai.org/adapter-<name>: "true"` when an adapter is loaded. Session routing does not use these labels directly but the natural consequence of stickiness is that the same Pod handles all requests for a given session, and thus adapters stay hot on that Pod.

### Graceful Degradation

Session routing is **best-effort**, not strict. It degrades in three cases:

| Situation | Behavior |
|---|---|
| Target Pod dies | Endpoint removed from group; next request for this session routes by LeastLoad |
| Target Pod overloaded (> maxLoadFactor) | Entry evicted; routes by LeastLoad for this and future requests until Pod recovers |
| No session key in request | Routes by LeastLoad (no affinity) |
| KubeAI restart | All session entries lost; next request re-establishes via LeastLoad |

### Affected Components

| Component | Change |
|---|---|
| `api/k8s/v1/model_types.go` | Add `Session` strategy and `Session` config struct |
| `internal/loadbalancer/group.go` | Add `sessions` map, `sessionEntry`, expiry goroutine |
| `internal/loadbalancer/balance_session.go` (NEW) | `pickSession`: routing algorithm described above |
| `internal/loadbalancer/load_balancer.go` | Dispatch to `pickSession` when strategy is `Session` |
| `internal/loadbalancer/group_test.go` | Unit tests for session creation, expiry, overload eviction |

## Design Decisions

### Why a header, not a cookie?

HTTP headers are simpler for API/gRPC clients. Cookies require browser context. The header name is configurable so operators can use whatever their API gateway passes (e.g., `X-User-Id`, `Authorization` header hash, `X-Request-Id`).

### Why not make session key derivation configurable (e.g., from body)?

Extracting from the request body is problematic for streaming requests (body may only be readable once). Headers are read before the body is forwarded. Body-based keys can be added in Phase 2 if needed (the request is already buffered for model-name extraction).

### Why not use Kubernetes Session Affinity on the Service?

Kubernetes `Service.spec.sessionAffinity: ClientIP` only works for TCP connections, not individual HTTP requests within a connection. It also doesn't support TTL or overload fallback.

## Implementation Phases

### Phase 1: Header-based Session Key
* `Session` strategy with `HeaderName` + `TTL` + `MaxLoadFactor`.
* In-memory session table with background expiry.
* Graceful degradation on pod failure and overload.
* Unit tests.

### Phase 2: Body-derived Session Key
* Option to derive session key from request body field (e.g., `$.user` in the OpenAI request body).
* Useful when clients cannot add custom headers.
* Requires buffering the first N bytes of the body.

### Phase 3: Adapter-aware Session Routing
* When `Session` strategy is active, prefer Pods that already have the requested adapter loaded.
* Combines session stickiness with adapter label selection.
* Falls back to any Pod if adapter-loaded Pods are all overloaded.

## Relevant Reading

* [KubeAI Issue #509 — Session Based Routing](https://github.com/substratusai/kubeai/issues/509)
* [vLLM Dynamic LoRA Serving](https://docs.vllm.ai/en/latest/features/lora/#dynamically-serving-lora-adapters)
* [Consistent Hashing with Bounded Loads](https://ai.googleblog.com/2017/04/consistent-hashing-with-bounded-loads.html)
