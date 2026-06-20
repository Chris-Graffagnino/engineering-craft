# Extraction: Kubernetes — Operational Robustness (Reconciliation Mechanics)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical clones of the canonical reference controller and the reusable machinery — `kubernetes/sample-controller` (the annotated reference controller), `kubernetes/client-go` (`util/workqueue`, `util/retry`, `kubernetes/fake`), and `kubernetes-sigs/controller-runtime` (`pkg/reconcile` contract, `pkg/envtest`). Test-strategy facts (tiers, `[Disruptive]`) are attributed inline to the Kubernetes e2e / Sonobuoy testing docs. **kubernetes/kubernetes itself was deliberately not cloned** — it is enormous and mostly irrelevant to the craft; the reusable patterns live in the repos above.

**Seam discipline:** this extraction stays on the *craft* side of the boundary with `software-architect`. It extracts reconciliation/retry/observability **mechanics**. It does **not** address "should you design your system around declarative reconciliation," service granularity, or API design philosophy — those are architecture and route to the other skill.

Dual purpose: (1) the founding repo for the **operational-robustness** dimension; (2) the **third witness** that graduates several testing principles — see §3a.

---

## 0. Identity

- **Repo(s):** kubernetes/sample-controller, kubernetes/client-go (sparse), kubernetes-sigs/controller-runtime (sparse)
- **Clone:** sample-controller full `--depth 1`; the other two `--filter=blob:none --sparse`. Thesis is mechanics, not decision-evolution.
- **Language(s):** Go
- **Why it's in the corpus:** Reference example of operational-robustness craft (convergent reconciliation), and the pivotal third witness for C5/C7/C10/C15.

## 1. Thesis

**Build components that converge on desired state by repeatedly observing actual state and acting on the difference — so they self-heal through missed events, conflicts, and crashes.** Robustness comes from idempotent, level-triggered reconciliation plus disciplined retry/backoff, not from getting every event exactly once.

## 2. Three-tier source inventory

**Tier 1 — Codified contract:**
- `controller-runtime/pkg/reconcile/reconcile.go` — the Reconciler contract: level-based reconciliation, requeue/backoff semantics, terminal vs retriable errors.
- `client-go/util/retry/util.go` — the optimistic-concurrency retry contract (`RetryOnConflict`).

**Tier 2 — Reference implementation + machinery (annotated):**
- `kubernetes/sample-controller/controller.go` — the canonical controller, with explanatory comments on the workqueue/retry pattern and the reconcile (`syncHandler`).
- `client-go/util/workqueue/default_rate_limiters.go` — the default two-layer rate limiter.

**Tier 3 — Test seams (corroboration):**
- `controller-runtime/pkg/envtest/` — integration testing against a real local control plane (etcd + kube-apiserver).
- `client-go/kubernetes/fake` — in-memory fake clientset for unit tests.

## 3. Candidate practices (operational-robustness dimension)

### C19. Level-triggered reconciliation — act on observed-vs-desired, re-derived each cycle

- Practice: Drive action from *actual state read from the source of truth (or a cache)*, not from individual change events. The work item carries only an identity (namespace/name), never the event or object contents, forcing a fresh read — so missed, duplicated, or reordered events can't corrupt behavior; the controller simply observes reality and converges. Deletion is *observed as absence*, not signaled.
- Evidence: **Verified** — `reconcile.go` contract ("Reconciliation is level-based … driven by actual cluster state read from the apiserver or a local cache"; Request holds only NamespacedName, "does NOT contain information about any specific Event"); `sample-controller` `syncHandler` re-Gets the Foo and returns nil on NotFound.
- Why / constraint: Edge-triggered designs must process every event exactly once and in order — fragile under the inevitable dropped/duplicated event. Level-triggering is self-correcting.
- Counter-case: Requires a *requeryable source of truth* for current state. For pure event streams with no re-readable state (some message-processing pipelines), edge-triggered may be all that's available — but then exactly-once/ordering handling becomes a hard requirement you own.
- Convergence: _stub_ — the robustness-dimension analogue of SQLite/FDB's "re-derive rather than trust deltas." Operational-robustness has only K8s so far; revisit if a 2nd robustness repo enters.
- Graduates to: `operational-robustness.md`

### C20. Idempotent reconcile — converge on the diff, safe to repeat

- Practice: Compute the difference between desired and observed and act *only on the difference* (create-if-missing, update-if-different); a second run after convergence is a no-op. This is what makes retries and periodic resyncs safe.
- Evidence: **Verified** — `sample-controller` `syncHandler`: creates the Deployment if NotFound; updates only `if foo.Spec.Replicas != deployment.Spec.Replicas`.
- Why / constraint: Retry, resync, and crash-recovery all re-invoke reconcile; non-idempotent logic would double-act.
- Counter-case: Requires operations expressible as idempotent create/update. Inherently non-idempotent side effects (send an email, charge a card) need an explicit idempotency key or dedup layer before they're safe to put behind a reconciler.
- Convergence: _stub_ — pairs with SQLite/FDB idempotent-compensation discipline; robustness-dimension singleton for now.
- Graduates to: `operational-robustness.md`

### C21. Work queue of keys (not objects), deduplicating

- Practice: Enqueue a stable *identity* (namespace/name), not the object; dequeue, then read fresh state. Identical keys collapse to a single item, so a burst of events for one object causes one reconcile.
- Evidence: **Verified** — `sample-controller` workqueue typed on `cache.ObjectName`; `enqueueFoo` Adds the key; `reconcile.go` notes "The workqueue then de-duplicates identical requests."
- Why / constraint: Queuing objects races against staleness and floods the queue; queuing identities + re-reading is both fresher and naturally rate-limiting.
- Counter-case: Keys must be stable identities, and the re-read cost must be acceptable — affordable here only because the informer cache makes reads local/cheap.
- Convergence: _stub_ — robustness-dimension singleton.
- Graduates to: `operational-robustness.md`

### C22. Two-layer rate-limited retry, reset on success

- Practice: Combine **per-item exponential backoff** (a repeatedly-failing item backs off, capped) with an **overall token-bucket ceiling** (caps total retry throughput), taking the larger delay; on success, `Forget` the item to reset its backoff. On error, requeue via `AddRateLimited`.
- Evidence: **Verified** — `default_rate_limiters.go` `DefaultControllerRateLimiter` = `MaxOf(ItemExponentialFailureRateLimiter(5ms→1000s), BucketRateLimiter(10qps,100))`; `sample-controller` `processNextWorkItem`: `Forget` on success, `AddRateLimited` on error, `Done` in defer.
- Why / constraint: Per-item backoff stops one bad object from hot-looping; the global bucket stops a mass-failure from melting the API; reset-on-success keeps a recovered item responsive.
- Counter-case: Caps and bucket sizes are tunables — latency-sensitive user-facing retries want smaller caps; batch reconcilers tolerate larger. The pattern, not the constants, transfers.
- Convergence: _stub_ — strong general retry craft; one corpus witness so far (cf. resilience libraries broadly).
- Graduates to: `operational-robustness.md`

### C23. Distinguish transient from terminal errors

- Practice: Requeue *transient* failures (network, version conflict) with backoff; *absorb* terminal/semantic failures (malformed input, not-owned) by returning success so they don't hot-loop — and wait for the next change event to retry.
- Evidence: **Verified** — `sample-controller` returns `nil` (absorbs) on missing spec field and on not-controlled-by, but returns the error (→ requeue) on transient Get/Create/Update failures; `reconcile.go` formalizes this as `TerminalError` → no requeue.
- Why / constraint: Requeuing a permanently-bad input wastes the queue forever; absorbing a transient error silently drops real work. The classification is the craft.
- Counter-case: Misclassification cuts both ways — mark transient-as-terminal and you lose work; mark terminal-as-transient and you hot-loop. Demands deliberate error taxonomy.
- Convergence: _stub_ — robustness-dimension singleton; broadly applicable.
- Graduates to: `operational-robustness.md`

### C24. Optimistic concurrency control — detect-and-retry, never lock

- Practice: For concurrent writers to one resource: re-fetch, modify, attempt update, and on a version-conflict (409) wait per a backoff (with jitter) and retry — re-fetching *every* attempt because a conflict means your copy is stale. No locks held.
- Evidence: **Verified** — `retry.RetryOnConflict` (= `OnError(backoff, errors.IsConflict, fn)`); the doc explicitly requires refetch-per-try; `DefaultRetry`/`DefaultBackoff` carry `Jitter: 0.1`.
- Why / constraint: Locking across a distributed API is expensive and deadlock-prone; optimistic CC keeps the common (uncontended) path lock-free and only pays on actual conflict.
- Counter-case: Wins under low-to-moderate contention. On a *hot* object with heavy write contention, the retry storm is worse than serializing — then queue/shard the writes instead. *(Cross-skill: choosing a consistency model is `software-architect`'s lens; the retry-on-conflict mechanics live here. Also rhymes with Redis/SQLite "don't fake atomicity — expose the trade-off.")*
- Convergence: _stub_ — extremely common pattern; one corpus witness so far.
- Graduates to: `operational-robustness.md`

### C25. Ownership guard — refuse to mutate what you don't own

- Practice: Stamp created resources with owner references, and check `IsControlledBy` before acting on a resource; if it isn't yours, refuse and surface a warning. Enables safe cascading garbage collection and stops two controllers fighting over one object.
- Evidence: **Verified** — `sample-controller` emits a warning + returns error when the Deployment exists but isn't controlled by the Foo; sets owner refs on created objects.
- Why / constraint: Without an ownership check, controllers silently clobber each other's resources; with it, ownership is explicit and GC can cascade safely.
- Counter-case: Assumes a single-owner model. Genuinely shared resources with multiple legitimate writers don't fit and need a different coordination scheme.
- Convergence: _stub_ — robustness-dimension singleton.
- Graduates to: `operational-robustness.md`

### C26. Separate desired (spec) from observed (status); report reality back

- Practice: Keep intent (`spec`) and observed reality (`status`) distinct; the controller writes `status` to reflect the world, via a status subresource that *cannot* change spec; an `observedGeneration`-style field tells consumers whether the controller has caught up to the latest spec.
- Evidence: **Verified** — `sample-controller` `updateFooStatus` writes status "to reflect the current state of the world"; the comment notes `UpdateStatus` can't change Spec.
- Why / constraint: Conflating the two lets a controller (or user) overwrite intent with observation, or vice versa; separation makes "what we want" vs "what is" legible and is the basis of observability for the resource.
- Counter-case: Status is eventually-consistent and may lag; consumers must not treat it as authoritative-now. (This is operational-robustness/observability craft, not API *architecture*.)
- Convergence: _stub_ — robustness-dimension singleton.
- Graduates to: `operational-robustness.md`

### C27. Wait for cache sync before acting; resync periodically as a drift safety-net

- Practice: Block on `WaitForCacheSync` before processing so you never act on a cold/partial cache; schedule a periodic resync that re-reconciles everything, catching drift and any missed events — belt-and-suspenders with event-driven triggering.
- Evidence: **Verified** — `sample-controller` `Run` calls `WaitForCacheSync` before starting workers; comment: "Periodic resync will send update events for all known Deployments."
- Why / constraint: Acting on an unsynced cache makes wrong decisions from incomplete data; resync bounds how long any missed-event drift can persist.
- Counter-case: Resync period trades drift-correction latency against load — too frequent is needless churn; too rare lets drift linger.
- Convergence: _stub_ — the resync safety-net rhymes with SQLite's redundant checks; robustness-dimension singleton for now.
- Graduates to: `operational-robustness.md`

### C28. Integration-test against a real control plane (envtest), unit-test against a fake

- Practice: Test controllers at two tiers — fast unit tests against an in-memory fake clientset, and integration tests against a *real* local etcd + kube-apiserver (envtest) so you exercise real API semantics (admission, defaulting, validation, watch) rather than a mock's approximation.
- Evidence: **Verified** — `controller-runtime/pkg/envtest` starts a local control plane (etcd + kube-apiserver); `client-go/kubernetes/fake` provides the unit-test fake.
- Why / constraint: A mock encodes your *belief* about API behavior; a real apiserver encodes the *actual* behavior, closing the gap that hides bugs.
- Counter-case: envtest is heavier/slower (real binaries) → reserve for integration; keep the inner loop on the fake. (This is the K8s instance of the testing-dimension's C10/C15.)
- Graduates to: `operational-robustness.md` (with cross-link to `testing-correctness.md`)

## 3a. Convergence cross-check — the graduations

Kubernetes is the **third independent witness** for several testing-dimension principles. With three unrelated repos, these meet the ≥3 bar and graduate into `_principles-index.md` (created this run).

| Principle | SQLite | FoundationDB | Kubernetes | Verdict |
|---|---|---|---|---|
| **Independent redundant verification** (C5) | 4 harnesses + multiple fuzzers | Simulation + Circus perf + hardware-failure | unit + integration(envtest) + e2e + conformance tiers; e2e exists "when unit and integration are insufficient" (k8s e2e docs) | **3/3 — GRADUATES (P1)** |
| **Design for testability / injection seams** (C10) | pluggable malloc + VFS | whole system on Flow | fake clientset (unit) + envtest real control plane (integration) | **3/3 — GRADUATES (P2)** |
| **Test under systematically injected faults + check invariants** (C7 base, + C8) | OOM/IO/crash loops + integrity_check | sim machine/network/DC faults + validation | `[Disruptive]` e2e: nodes taken down, API server made unresponsive, fault-tolerance/DR tested (k8s e2e / Sonobuoy docs) | **3/3 — GRADUATES (P3)** |
| **Same code in test and production** (C15) | real library under instrumented malloc | same code in sim + prod | envtest runs the *real* apiserver/etcd; conformance runs on real clusters | **3/3 — GRADUATES (P4)** |
| **Deterministic substrate for reproducibility** (refinement of P3, C14) | seeded loops | full deterministic simulation | **NO — K8s e2e is real-cluster, non-deterministic** | **stays 2/3** — K8s is the *counter-point* |
| **Regression ratchet** (C11) | every bug → permanent test; fuzzcheck corpus | not evidenced | growing permanent conformance/e2e suite (suggestive), but strict "bug→test" policy not confirmed here | **1/3 + suggestive** |

**The key nuance — record it in the index.** The *base* principle (inject faults systematically behind a seam, assert invariants) graduates at 3/3. Its *determinism refinement* does not: Kubernetes deliberately runs faults on real, non-deterministic clusters and accepts the well-known cost — flaky e2e. That flakiness is exactly the tax FoundationDB's deterministic substrate was built to avoid. So determinism is a **calibrated refinement** (worth its high price when reproducibility dominates, as in an embedded DB or a distributed transaction engine), not a universal requirement. This *is* the governing dial (calibrate rigor to blast radius) applied to a specific technique.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            The full controller/informer/workqueue/lister machinery.
- Why not general: Justified for managing many declaratively-specified resources that drift and
                   emit unreliable event streams. For a one-shot script, a CRUD endpoint, or a
                   linear job, standing up informers + caches + rate-limited queues is heavy
                   over-engineering. EXTRACT the principles (level-triggered idempotent
                   convergence; retry with backoff; transient-vs-terminal) and apply them at the
                   scale of a single retry loop — don't build a controller.
```

```
- Item:            Periodic full resync of all objects.
- Why not general: A drift safety-net sized to Kubernetes's unreliable-watch reality. If your
                   source of truth is reliable and re-readable on demand, frequent blanket
                   resync is needless load. Tune to the actual drift risk.
```

```
- Item:            Real-cluster, non-deterministic e2e as a primary correctness mechanism.
- Why not general: This is a *deliberate calibration*, not a model to copy blindly: K8s accepts
                   flaky e2e because its blast-radius/patch profile differs from an embedded DB.
                   A correctness-critical, hard-to-patch system should prefer the deterministic
                   substrate (P3 refinement) despite its cost. Choose per the dial, not by
                   imitation.
```

## 5. Open questions / corpus cross-checks

- The operational-robustness dimension currently has a single witness (Kubernetes) for C19–C27, so its practices are **1/3** — strong and well-evidenced, but not yet cross-validated. A natural second robustness witness: an Erlang/OTP supervision-tree codebase, or Temporal's worker/retry internals (note: Temporal was dropped from the testing-side corpus for DDD overlap, but its *retry/durability mechanics* are robustness-craft and would not overlap `software-architect`). Consider adding one to validate the reconciliation-dimension principles.
- Re-check the **regression ratchet** (C11) against the Kubernetes contributor testing policy to confirm/deny the strict "every fix ships with a test" rule → would lift it past 1/3.
- C24 (optimistic concurrency), C22 (two-layer retry), C12 (tiered depth) are each strong but single-corpus-witness on the robustness side; flag for corroboration.
