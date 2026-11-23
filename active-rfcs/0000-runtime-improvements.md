- Start Date: 2026-02-27
- Target Major Version: 10.x

## 0) Context and Motivation

Criticisms:

-   "I can't *prove* the performance envelope."
-   "Debugging across JS<->native is not deterministic enough."
-   "Memory and lifecycle rules feel too implicit."
-   "Tooling doesn't show me where time and memory went."

This RFC proposes a runtime roadmap that directly addresses those
concerns by making boundary costs measurable and cheap, ownership
explicit, and profiling first-class.

------------------------------------------------------------------------

## 1) Goals

### Primary Goals

1.  **Performance Predictability**\
    Reduce tail latency and eliminate random jank spikes.

2.  **Explicit Ownership & Lifecycle**\
    Make object lifetime across JS and native deterministic and
    enforceable.

3.  **Observability-First Runtime**\
    Make it easy to answer: "Why did this hitch?"

### Secondary Goals

4.  **Startup Consistency**
5.  **Safe Migration Path**

### Non-Goals

-   Rewriting the UI paradigm

------------------------------------------------------------------------

## 2) Current Pain Points (Hypotheses)

1.  Boundary overhead is too dynamic (reflection, runtime dispatch).
2.  Ownership between JS and native is ambiguous.
3.  Threading rules are insufficiently enforced.
4.  Tooling does not unify JS and native execution clearly.

------------------------------------------------------------------------

## 3) Principles

1.  Make the fast path the default path.
2.  Pay costs once, not per call.
3.  Ownership must be explicit and enforceable.
4.  Everything must be measurable.

------------------------------------------------------------------------

## 4) Proposal

### 4.1 Codegen-First Bindings

Move core bindings to a code-generated interface layer with static
dispatch.

**Deliverables:** - Binding schema format - JS wrappers + native stubs -
Marshaling fast paths - Tracepoints per binding

**Acceptance Criteria:** - Top 100 APIs use codegen path - Profiler
reports dynamic vs codegen usage

------------------------------------------------------------------------

### 4.2 Explicit Ownership Model

Introduce ownership policies:

-   **Borrowed**
-   **Owned**
-   **Pinned**

Use stable numeric handles with handle tables internally.

**Deliverables:** - Handle table implementation - `dispose()` /
`close()` lifecycle hooks - Ownership annotations in schema - Dev-time
misuse warnings

**Acceptance Criteria:** - Leak snapshots identify retained native
objects - Misuse triggers actionable errors

------------------------------------------------------------------------

### 4.3 Boundary Performance Improvements

-   Zero-copy typed arrays and buffers
-   Packed structs for common value types
-   Batched UI transactions

**Acceptance Criteria:** - Reduced boundary call count - Improved
p95/p99 frame times

------------------------------------------------------------------------

### 4.4 Threading Enforcement

-   Enforce UI-thread-only objects
-   Provide explicit scheduling primitives

**Acceptance Criteria:** - Clear errors for thread misuse

------------------------------------------------------------------------

### 4.5 Observability & Profiling

1.  Unified JS + native timeline tracing
2.  Boundary profiler (top bindings, call counts, costs)
3.  Memory correlation (JS heap + native heap + handles)

**Acceptance Criteria:** - Performance issues diagnosable from captured
traces

------------------------------------------------------------------------

### 4.6 Startup Improvements

-   Bytecode caching / snapshotting
-   Precomputed binding tables
-   Lazy loading of native APIs

**Acceptance Criteria:** - Improved cold-start p95

------------------------------------------------------------------------

## 5) Migration Strategy

-   Phase 1: Ship with npm tag during acceptance cycle
-   Phase 2: Dual-path bindings (codegen + fallback)
-   Phase 3: Tooling for plugin migration
-   Phase 4: Default for new apps

------------------------------------------------------------------------

## 6) Benchmarks & Metrics

Scenarios: - List scroll stress - Startup & first render - Navigation
churn - Animation/gesture pressure - Memory retention cycles

Metrics: - Frame time p50 / p95 / p99 - Boundary calls/sec - Marshaling
allocations/sec - Cold-start p95 - Retained native handles

------------------------------------------------------------------------

## 7) Risks

-   Engineering scope
-   API surface complexity
-   Plugin inertia
-   Instrumentation overhead

------------------------------------------------------------------------

## 8) Phased Plan

### Phase A: Observability + Baseline Suite

### Phase B: Ownership + Handle Tables

### Phase C: Codegen for Hot APIs

### Phase D: Zero-Copy + Batching

### Phase E: Startup Optimization

------------------------------------------------------------------------

## 9) Open Questions

-   Target engine constraints?
-   Current top boundary hot paths?
-   Ecosystem keystone plugins?
-   Schema ergonomics?

------------------------------------------------------------------------

## 10) Predictability

We are making the runtime predictable:

-   Static, measurable boundary calls
-   Explicit ownership
-   Unified profiling
-   Reasonable, enforceable semantics
