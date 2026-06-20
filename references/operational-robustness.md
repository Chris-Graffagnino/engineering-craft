# Operational Robustness — Dimension Reference

**Role:** consumed in **Apply mode**. Load this when a craft question touches how a component *keeps working in production* — self-healing/reconciliation, retry and backoff, surviving missed events or crashes, coordinating concurrent writers to shared state, reporting a component's real status, or testing a controller/reconciler. Synthesize from it — do not quote it at the user.

**Sources:** distilled from **three** operational-robustness exemplars spanning three distinct recovery models: **Kubernetes** (reconcile-external-state — `sample-controller`, `client-go`, `controller-runtime`), **Erlang/OTP** (supervised crash-and-restart — `supervisor`/`gen_server`), and **Temporal** (durable execution / event-replay — the Go SDK's retry + determinism contracts). Per-practice evidence: `_extract-kubernetes.md` (C19–C28), `_extract-erlang-otp.md` (C29–C31), `_extract-temporal.md` (C47 + the graduating cross-check).

**Confidence caveat — read it before applying anything here.** **The robustness core has GRADUATED to 3/3** across Kubernetes + Erlang/OTP + Temporal (three different recovery mechanisms — reconcile / restart / replay): self-heal by re-deriving + idempotent (**C19+C20 → P7**), retry safely by classifying + bounding (**C23 + C22-ceiling → P8**), separate recovery from business logic (**C29 → P9**). Lead with those. **At 2/3:** per-item exponential backoff (C22 *backoff* leg — K8s + Temporal; OTP is the no-backoff counter-point), keyed dedup (C21 — K8s + Temporal), single-owner (**C25 — HELD**, see §5.2/§10). **At 1/3:** durable-execution/replay (C47, Temporal-only), spec/status (C26), cache-sync+resync (C27), optimistic concurrency (C24, with OTP's share-nothing as a counter-case). (C28 is graduated via the *testing* corpus — P2/P4.) See §10.

---

## Contents
- §1 The master dial — how much reconciliation machinery to build
- §2 The core pattern — level-triggered idempotent convergence
- §3 Making a component self-heal
- §4 Retrying safely
- §5 Coordinating concurrent writers
- §6 Reporting reality back (resource observability)
- §7 Testing a reconciler
- §8 Do NOT cargo-cult (anti-pattern ledger)
- §9 Cross-skill links
- §10 Provenance & confidence

---

## §1 The master dial: calibrate the machinery to drift risk × blast radius

**Before recommending any pattern below, set the dial.** This dimension's defining failure mode is *over-building* — importing Kubernetes's full controller/informer/workqueue stack into a problem that never needed it. How much reconciliation machinery to build scales with **how unreliable your event stream is × how much undetected drift costs**:

- *High drift risk, high blast radius* — you manage many independently-specified resources that drift on their own and reach you through a lossy, reorderable event stream (the Kubernetes situation) → full level-triggered reconciliation: caches, periodic resync, rate-limited work queues.
- *Low drift risk, low blast radius* — your source of truth is reliable and re-readable on demand, the work is a linear job or a single CRUD endpoint → a single **idempotent retry loop** is the whole pattern; informers and resync are needless load.

**Extract the principles, not the controller.** The durable lessons — level-triggered idempotent convergence, retry with backoff, transient-vs-terminal classification — apply at the scale of *one retry loop*. The *machinery* (informers, listers, rate-limited queues, periodic resync) is a high-cost amortization that only pays off at Kubernetes's scale and event-unreliability. (Provenance: `_extract-kubernetes.md` §4.) This is `_principles-index.md`'s governing dial (P0 — rigor ∝ blast radius × patch latency) applied to robustness machinery.

---

## §2 The core pattern: level-triggered idempotent convergence

The dimension's centerpiece — the single shape every robustness practice here composes into. A component is robust not because it catches every event, but because it **re-derives what to do from observed reality each cycle** and **acts in a way that's safe to repeat**:

1. **Trigger on identity, not contents** — the work item is a stable key (namespace/name), never the event or object payload (C21). A burst of events for one object collapses to one unit of work.
2. **Observe actual state** — on dequeue, read current desired *and* current observed state fresh from the source of truth (or a cache), rather than trusting the delta that woke you (C19, *level-triggered*).
3. **Act only on the difference, idempotently** — create-if-missing, update-if-different; a second run after convergence is a no-op (C20).
4. **Requeue on failure, classified** — transient errors requeue with backoff; terminal/semantic errors are absorbed so they don't hot-loop (C23, C22).

Compose these and the component **self-corrects**: a dropped event, a duplicate, a crash mid-reconcile, or a reordering all resolve on the next cycle, because the next cycle re-reads reality and converges on the diff. Edge-triggered designs — process every event exactly once, in order — have no such safety net and inherit exactly-once/ordering as a hard obligation.

**Sized to the dial (§1):** most components need this as *one idempotent retry loop around a re-read of the source of truth*, not a controller. The full informer/workqueue stack is the same pattern industrialized for fleets of drifting resources. **Status: 1/3** (Kubernetes, the canonical reference implementation). This loop is the candidate centerpiece for graduation into `_principles-index.md` once a second robustness repo (Erlang/OTP supervision, Temporal worker mechanics) corroborates it.

---

## §3 Making a component self-heal

*The component must end up in the right state even though events get dropped, duplicated, reordered, or interrupted by a crash.*

**§3.1 Re-derive state from the source of truth, don't trust deltas (C19 · 3/3 — GRADUATED to P7).** Act on *actual state read from the source of truth*, not on the change event that triggered you. Carry only an identity in the work item, forcing a fresh read; observe deletion as *absence*, not as a signal you must not miss.
- *Apply when:* state can drift out from under you and your event stream is lossy/unordered, **but** current state is re-queryable so the component can always re-derive truth.
- *Skip when:* you have a reliable, exactly-once, ordered event stream with no re-readable backing state (some message-processing pipelines) — then edge-triggered is what you've got, and exactly-once/ordering becomes a hard requirement *you* now own.
- *Calibration:* the cost is a fresh read per cycle, affordable here only because the informer cache makes reads local. If reads are expensive and drift is rare, reconcile less often rather than caching everything.
- *Evidence:* `reconcile.go` contract ("Reconciliation is level-based … driven by actual cluster state"; Request holds only `NamespacedName`, not the event); `sample-controller` `syncHandler` re-Gets the object and returns nil on NotFound. (`_extract-kubernetes.md` C19.)
- *2nd witness (Erlang/OTP):* a supervised process recovers by re-running `init/1` to rebuild state from known-good rather than trusting corrupted in-memory state — crash-triggered rather than continuous. (`_extract-erlang-otp.md` §3a.)
- *3rd witness (Temporal) → GRADUATES:* a workflow reconstructs its state by **replaying the durable event history** — it never trusts volatile memory across a crash. Three recovery models (re-read / restart-reinit / replay), one principle → **P7**. (`_extract-temporal.md` §3a.)

**§3.2 Idempotent recovery — act only on the diff, safe to repeat (C20 · 3/3 — GRADUATED to P7).** Compute desired-minus-observed and act *only on the difference*, so re-running after convergence is a no-op. This is precisely what makes retry, resync, and crash-recovery safe.
- *Apply when:* the action is expressible as idempotent create/update against a queryable resource.
- *Skip / guard when:* the side effect is inherently non-idempotent (send an email, charge a card) — it needs an explicit idempotency key or dedup layer *before* it is safe behind a reconciler. Never drop a raw "charge the card" into a loop that re-runs.
- *Calibration:* idempotency is a property you design in (compare-then-act, conditional writes), not a default you get for free.
- *Evidence:* `sample-controller` `syncHandler` creates the Deployment if NotFound, updates only `if foo.Spec.Replicas != deployment.Spec.Replicas`. (`_extract-kubernetes.md` C20.)
- *2nd witness (Erlang/OTP):* "let it crash" only works *because* restart-to-clean-`init` is repeatable — recovery re-runs the start path. (`_extract-erlang-otp.md` §3a.)
- *3rd witness (Temporal) → GRADUATES:* Temporal activities run **at-least-once** (they're retried), so they **must be idempotent** — the same constraint, made explicit. Co-graduates with C19 as **P7** (re-derive + idempotent are the two legs of the self-healing core). (`_extract-temporal.md` §3a.)

**§3.3 Cache-sync gate + periodic resync as a drift safety-net (C27 · 1/3).** Block on cache sync before processing so you never act on cold/partial data; schedule a periodic full re-reconcile to catch drift and any events the event-path missed — belt-and-suspenders *behind* event triggering, not instead of it.
- *Apply when:* your cache/watch can miss events or start cold, and undetected drift is costly.
- *Skip when:* your source of truth is reliable and re-readable on demand — blanket periodic resync of *everything* is then pure load (see §8).
- *Calibration:* the resync period trades drift-correction latency against load — too frequent is churn, too rare lets drift linger. Size it to actual drift risk, not by imitation of Kubernetes's defaults.
- *Evidence:* `sample-controller` `Run` calls `WaitForCacheSync` before starting workers; comment: "Periodic resync will send update events for all known Deployments." (`_extract-kubernetes.md` C27.)

**§3.4 Separate recovery from business logic (C29 · 3/3 — GRADUATED to P9).** Don't thread defensive error handling through worker code; let a component fail cleanly and put recovery in a *separate* mechanism — a supervisor, a reconcile loop, a durable-execution platform — that restarts/retries/replays from a known-good point. Recovery policy lives in one place; the worker stays on the happy path.
- *Apply when:* restart-to-known-good is cheap and safe (state is reconstructible, §3.2) and a brief unavailability during restart is acceptable.
- *Skip when:* a long-lived in-memory computation with no durable/re-derivable source — a crash loses irrecoverable state; add checkpointing before "let it crash" is safe. Also needs real fault isolation between worker and recovery.
- *Calibration:* size the restart unit so that losing and rebuilding its state is cheap.
- *Evidence:* OTP `supervisor` restarts the worker (`gen_server`), which carries no restart logic; K8s's reconcile loop *is* recovery, distinct from the workload; **Temporal's durable-execution platform** handles retry/replay/recovery while the activity is just the business work. Three mechanisms, one principle → **P9**. (`_extract-kubernetes.md`, `_extract-erlang-otp.md` C29, `_extract-temporal.md` §3a.)

**§3.5 Scope the restart blast radius to fate-sharing (C30 · 1/3).** When you must restart, restart exactly the set that shares fate with the failed part. OTP names the choice: `one_for_one` (just the failed child), `one_for_all` (all siblings — coupled), `rest_for_one` (the failed child and those started after it).
- *Apply when:* you supervise multiple related components with differing coupling.
- *Skip when:* a single isolated worker — its blast radius is trivially itself.
- *Calibration:* restart too little and siblings talk to a peer that lost its state; too much and you interrupt healthy work. Over-coupling (`one_for_all` for independent children) turns one failure into a fleet-wide restart (see §8).
- *Evidence:* OTP restart strategies (codified + verified); the Kubernetes-side analogue is weak (resources reconcile independently), so this stays 1/3. (`_extract-erlang-otp.md` C30.)

**§3.6 Durable execution — persist an event log, replay to recover (C47 · 1/3).** For long-running state that must survive crashes exactly, record every state-changing step to a durable log and reconstruct state after a failure by *replaying* it, rather than checkpointing live memory. For replay to be faithful, the replayed code must be **deterministic** (route time/randomness/I/O through the platform, which records each result once).
- *Apply when:* long-lived orchestration/state must survive process loss *exactly* (sagas, multi-step workflows, money movement) and resume precisely where it was.
- *Skip when:* short, stateless, retryable work — a plain bounded retry (§4) is far cheaper than a durable-execution runtime + the determinism tax.
- *Calibration:* the determinism constraint (no wall-clock/rand/direct-I/O in the replayed path) is real friction — the robustness-altitude echo of the testing dimension's determinism (C14): a calibrated upgrade, not a default.
- *Evidence:* Temporal — `workflow/doc.go` ("Execution must be deterministic"; state reconstructed by replay; `SideEffect` records a result once so it isn't re-run on replay). (`_extract-temporal.md` C47.) *Seam: adopting durable execution / event sourcing as the system model is `software-architect`'s; the replay/determinism mechanics are here.*

---

## §4 Retrying safely

*A failure must not turn into a hot-loop that melts a shared downstream, and a recovered item must become responsive again.*

**§4.1 Classify transient vs terminal errors (C23 · 3/3 — GRADUATED to P8).** Requeue *transient* failures (network, version conflict) with backoff; *absorb* terminal/non-retryable failures (malformed input, not-owned) by returning success so they don't hot-loop — then wait for the next change event to retry them. **The classification is the craft.**
- *Apply when:* a reconcile/handler can fail for both retryable and permanently-bad reasons (almost always true).
- *Skip when:* genuinely *every* failure is retryable — but verify that before assuming it; a single misclassified permanent error hot-loops forever.
- *Calibration:* misclassification cuts both ways — transient-as-terminal silently drops real work; terminal-as-transient hot-loops. This demands a deliberate error taxonomy, not a catch-all `return err`.
- *Evidence:* `sample-controller` returns `nil` (absorbs) on a missing spec field and on not-controlled-by, returns the error (→ requeue) on transient Get/Create/Update; `reconcile.go` formalizes `TerminalError` → no requeue. (`_extract-kubernetes.md` C23.)
- *2nd witness (Erlang/OTP):* child restart *type* — `transient` (restart only on abnormal exit), `permanent` (always), `temporary` (never restart). (`_extract-erlang-otp.md` §3a.)
- *3rd witness (Temporal) → GRADUATES:* `NewNonRetryableApplicationError` / `NonRetryableErrorTypes` in the retry policy — mark errors non-retryable so they stop instead of looping. Three explicit error-taxonomies → **P8** (with the retry-ceiling, §4.2). (`_extract-temporal.md` §3a.)

**§4.2 Bound the retry rate so failures can't hot-loop (C22 · *ceiling* 3/3 GRADUATED to P8; *backoff* 2/3 refinement).** Combine **per-item exponential backoff** (a repeatedly-failing item backs off, capped) with an **overall ceiling** (caps total retry throughput); on success, *forget* the item to reset its backoff.
- *Apply when:* many items retry independently and a mass-failure could flood a shared downstream (an API server, a database).
- *Skip / simplify when:* a single retry stream with no shared-downstream blast radius — plain capped exponential backoff is enough; you don't need the global bucket.
- *Calibration:* the caps and bucket sizes are **tunables, not constants to copy** — latency-sensitive user-facing retries want smaller caps; batch reconcilers tolerate larger. The *pattern* (per-item backoff + global ceiling + reset-on-success) transfers; the numbers don't.
- *Evidence:* `DefaultControllerRateLimiter = MaxOf(ItemExponentialFailureRateLimiter(5ms→1000s), BucketRateLimiter(10qps,100))`; `processNextWorkItem` Forgets on success, AddRateLimited on error, Done in defer. (`_extract-kubernetes.md` C22.)
- *2nd witness (Erlang/OTP, ceiling only):* supervisor **restart intensity** (`MaxR` per `MaxT` s → terminate + escalate; default 1/5) is the same ceiling — but OTP has *no* per-item backoff (immediate restart). (`_extract-erlang-otp.md` §3a.)
- *3rd witness (Temporal) → GRADUATES the ceiling:* `RetryPolicy` `MaximumAttempts` / `MaximumInterval` is the ceiling; **and Temporal has `BackoffCoefficient`** — so the *backoff* leg now has 2 witnesses (K8s + Temporal), reaching **2/3**, with OTP the no-backoff counter-point. The ceiling graduates to **P8**; per-item exponential backoff is its calibrated 2/3 refinement (add it against a shared, overloadable downstream). (`_extract-temporal.md` §3a.)

**§4.3 Queue keys, not objects — dedup at the door (C21 · 2/3).** Enqueue a stable *identity*, not the object payload; dequeue, then read fresh state. Identical keys collapse to one item, so a burst of events for one object causes one reconcile — and the work always runs against current state.
- *Apply when:* one entity can be touched by many rapid events and you want coalescing + freshness for free.
- *Skip when:* each work item is genuinely unique with no re-readable backing state (a true job queue of distinct payloads) — then the payload *is* the work and there's nothing to re-read.
- *Calibration:* keys must be stable identities, and the re-read must be cheap (here, the informer cache makes it local). If re-reading is expensive, the dedup benefit may not pay for it.
- *Evidence:* `sample-controller` workqueue typed on `cache.ObjectName`; `reconcile.go`: "The workqueue then de-duplicates identical requests." (`_extract-kubernetes.md` C21.)
- *2nd witness (Temporal · 2/3):* task queues + **workflow-ID dedup** (`WorkflowIDReusePolicy`) — starting with an existing identity is deduplicated. The dedup-by-stable-identity aspect converges. (`_extract-temporal.md` §3a.)

**§4.4 Bound retries, then escalate — don't retry forever in place (C31 · 1/3 for the escalation).** Cap retries in a window (the §4.2 ceiling); when exceeded, *give up locally and hand the failure to a higher authority* that can take a coarser action (restart a larger subtree, fail the node, page a human) rather than looping at one level.
- *Apply when:* a deterministic/persistent failure won't clear by local retry **and** a higher level can do something different.
- *Skip when:* there is no meaningful higher authority, or local retry genuinely is the only recovery — then just bound and alert.
- *Calibration:* OTP escalates up the supervision tree on intensity overflow (1 per 5 s default); K8s rate-limits in place without escalating. The *bound* is shared craft (§4.2, C22 · 2/3); the *escalate-upward* response is the OTP-specific refinement (1/3).
- *Evidence:* OTP "Maximum Restart Intensity" → terminate children + fail upward. (`_extract-erlang-otp.md` C31.)

---

## §5 Coordinating concurrent writers

*Multiple actors may touch the same resource; they must neither corrupt it nor fight over it.*

**§5.1 Optimistic concurrency control — detect-and-retry, never lock (C24 · 1/3).** For concurrent writers to one resource: re-fetch, modify, attempt the update, and on a version-conflict re-fetch *again* (your copy is now stale) and retry under backoff-with-jitter. No locks are held.
- *Apply when:* contention is low-to-moderate — the common path is uncontended, so it stays lock-free and only pays on an actual conflict.
- *Skip when:* a *hot* object under heavy write contention — the retry storm is then worse than serializing; queue or shard the writes instead.
- *Calibration:* refetch-per-attempt is mandatory (a conflict means your copy is stale); jitter on the backoff prevents synchronized retry waves.
- *Evidence:* `retry.RetryOnConflict` (= `OnError(backoff, errors.IsConflict, fn)`), doc requires refetch-per-try; `DefaultRetry`/`DefaultBackoff` carry `Jitter: 0.1`. (`_extract-kubernetes.md` C24.) *Cross-skill: choosing the consistency model is `software-architect`'s call; the retry-on-conflict mechanics are craft and live here — see §9.*
- *Counter-case, not a witness (Erlang/OTP):* share-nothing actors sidestep the problem — with no shared mutable state there is no read-modify-write conflict to resolve. Optimistic CC is one answer to coordination; serializing through a single owning process is another. So C24 **stays 1/3** and its boundary is sharper for the contrast. (`_extract-erlang-otp.md` §3a/§4.)

**§5.2 Ownership guard — refuse to mutate what you don't own (C25 · 2/3 — HELD, not graduated).** Stamp created resources with an owner reference and check `IsControlledBy` before acting; if a resource isn't yours, refuse and surface a warning. Stops two controllers clobbering each other and enables safe cascading garbage collection.
- *Apply when:* multiple controllers/actors operate in the same resource space and could collide.
- *Skip when:* a single, exclusive owner by construction — the guard is redundant ceremony.
- *Calibration:* this assumes a *single-owner* model. Genuinely shared resources with several legitimate writers don't fit and need a different coordination scheme (lease, lock, or partition).
- *Evidence:* `sample-controller` emits a warning + returns an error when the Deployment exists but isn't controlled by the Foo; sets owner refs on created objects. (`_extract-kubernetes.md` C25.)
- *2nd witness (Erlang/OTP):* a supervisor structurally owns exactly its children — enforced by tree *topology* (vs K8s's runtime *guard check*). (`_extract-erlang-otp.md` §3a.)
- *3rd instance, but HELD (Temporal):* one current run owns a workflow's history (single-writer *by construction*, for log-consistency). All three establish single-ownership, but for **different purposes** (K8s anti-clobber/GC · OTP recovery topology · Temporal log-consistency) and mechanisms — convergence on a *shape*, not cleanly "the same practice for the same reason." So **C25 stays 2/3**; a clean anti-clobber-guard witness would graduate it. *(The disciplined hold — cf. the determinism refinement. See §10.)* (`_extract-temporal.md` §3a.)

---

## §6 Reporting reality back (resource observability)

*Consumers need to know what the component actually did, kept distinct from what was asked.*

**§6.1 Separate desired (spec) from observed (status) (C26 · 1/3).** Keep intent (`spec`) and observed reality (`status`) in distinct fields; the component writes `status` to reflect the world through a path that *cannot* change spec; an `observedGeneration`-style field tells consumers whether the component has caught up to the latest intent.
- *Apply when:* a component acts on declared intent and its progress/result matters to other readers — the spec/status split *is* the resource's observability.
- *Skip when:* a fire-and-forget action with no standing resource whose status anyone reads.
- *Calibration:* status is *eventually consistent* and may lag — consumers must not treat it as authoritative-now; `observedGeneration` is how they tell "caught up" from "stale."
- *Evidence:* `sample-controller` `updateFooStatus` writes status "to reflect the current state of the world"; the status subresource cannot change Spec. (`_extract-kubernetes.md` C26.) *(This is observability craft, not API-model architecture — see §9.)*

---

## §7 Testing a reconciler

**§7.1 Two tiers: fake for the inner loop, real control plane for integration (C28 · cross-validated — see note).** Unit-test against an in-memory fake of the dependency for speed; integration-test against a *real* local instance of it (Kubernetes: `envtest`'s real etcd + kube-apiserver) so you exercise actual semantics — admission, defaulting, validation, watch — instead of a mock's approximation.
- *Apply when:* the dependency has rich semantics a mock would only approximate (any real datastore/API), and those semantics can hide bugs.
- *Skip / reserve when:* the real instance is heavy/slow — keep it for the integration tier and keep the inner loop on the fake. Don't pay `envtest`'s startup on every unit test.
- *Calibration:* a mock encodes your *belief* about the dependency's behavior; the real instance encodes its *actual* behavior. Match the tier to how much that gap can hide bugs.
- *Evidence:* `controller-runtime/pkg/envtest` starts a real local control plane (etcd + kube-apiserver); `client-go/kubernetes/fake` provides the unit-test fake. (`_extract-kubernetes.md` C28.)
- **Confidence note:** unlike the robustness practices here (which top out at 2/3), this one is already **graduated at 3/3**. It is the Kubernetes witness of P2 (design-for-testability via injection seams) and P4 (same code in test and production), via SQLite + FoundationDB + Kubernetes. See `_principles-index.md` (P2/P4) and `testing-correctness.md` §4.4 / §5.4.

---

## §8 Do NOT cargo-cult (anti-pattern ledger)

The boundaries are the product. Recognize and refuse these:

- **Standing up a full controller/informer/workqueue stack for a one-shot script, a CRUD endpoint, or a linear job** — heavy over-engineering. Extract the *principles* (level-triggered idempotent convergence, retry-with-backoff, transient-vs-terminal) and apply them as a single retry loop. Build the machinery only at fleet scale with an unreliable event stream (§1).
- **Blanket periodic resync when your source of truth is reliable and re-readable** — a drift safety-net sized to Kubernetes's unreliable-watch reality, not a universal default. Tune resync to actual drift risk; with a reliable source, frequent full resync is pure load (C27).
- **Treating real-cluster, non-deterministic e2e as the default correctness mechanism** — Kubernetes *deliberately* accepts flaky e2e because its blast-radius/patch profile differs from an embedded DB. A correctness-critical, hard-to-patch system should prefer a **deterministic** fault-injection substrate (`testing-correctness.md` §2 / §5.1; principle P3's refinement) despite its cost. Choose per the dial, not by imitation.
- **Putting a non-idempotent side effect (charge, email, irreversible call) directly behind a reconciler/retry loop** — re-runs will double-act. Add an idempotency key or dedup layer first (C20).
- **Optimistic-retry on a hot, heavily-contended object** — the retry storm beats you; serialize, queue, or shard instead (C24).
- **Copying the rate-limiter constants** (5ms→1000s, 10qps/100) — those are Kubernetes's tunables. Transfer the *pattern* (per-item backoff + global bucket + reset-on-success); set the numbers to your own latency/throughput profile (C22).
- **Immediate restart with no backoff against a shared, overloadable downstream** — OTP's default (restart at once, rely on the intensity ceiling) is fine for isolated processes but will hammer a shared API/DB on a mass failure. When retries hit a shared dependency, add Kubernetes-style exponential backoff *in addition to* the ceiling (C22 has both legs for a reason; C31).
- **Over-coupling the restart scope** — `one_for_all` (or any "restart the whole subtree") when the children are actually independent turns a single failure into a fleet-wide restart. Scope restart to the genuine fate-sharing set (C30, §3.5).
- **"Let it crash" without reconstructible state** — crashing is only a recovery strategy when restart rebuilds state from a durable/re-derivable source. With irreplaceable in-memory state and no checkpoint, a crash destroys data (C29, §3.4).

---

## §9 Cross-skill links

Keep the seam with `software-architect` clean — this dimension owns reconciliation/retry/observability **mechanics**, not the system-shape decisions around them:

- **"Should the system be declarative / built around reconciliation at all?"** → `software-architect`. The *mechanics* of a reconcile loop (C19–C21) live here; the architectural choice to adopt that model is system-altitude.
- **"Should we adopt an actor / supervision (or share-nothing) architecture?"** → `software-architect`. The supervision/restart *mechanics* (C29–C31, "let it crash," restart strategies, bound-and-escalate) are craft and live here; choosing the actor model as the system's concurrency architecture is system-altitude.
- **Consistency-model / isolation selection** → `software-architect`. The optimistic-concurrency *retry mechanics* (C24, §5.1) are craft and live here; *which* consistency model the system should offer is architecture. (C24 also rhymes with Redis/SQLite "don't fake atomicity — expose the trade-off": same value, different altitude.)
- **Spec/status separation (§6)** is *resource-observability craft*, not API-model *architecture* — but if the question becomes "what should our API's resource model be," that's `software-architect`.
- **Testing seams (§7) ↔ testing-correctness dimension** — C28 is the same construct as that dimension's injection seams (P2) and same-code-in-test-and-prod (P4). Load `testing-correctness.md` when the question is about test *strategy* rather than controller robustness.

---

## §10 Provenance & confidence

- Full evidence and calibrated verbs: `_extract-kubernetes.md` (C19–C28), `_extract-erlang-otp.md` (C29–C31), `_extract-temporal.md` (C47 + the graduating §3a). Three witnesses chosen for *mechanistic distinctness*: Kubernetes reconcile-external-state, Erlang/OTP supervised-restart, Temporal durable-replay.
- **The robustness core has GRADUATED to 3/3 — P7, P8, P9** (the second dimension after testing to graduate principles):
  - **P7** = re-derive from a durable source + idempotent recovery (**C19 + C20**) — three mechanisms: re-read / restart-reinit / event-replay.
  - **P8** = retry safely: classify transient vs terminal + bound the rate (**C23 + C22-ceiling**). *Per-item exponential backoff* is a **2/3 refinement** (K8s + Temporal; OTP restarts immediately — the no-backoff counter-point).
  - **P9** = separate the recovery mechanism from business logic (**C29**) — reconcile loop / supervisor / durable-execution platform.
- **At 2/3 (pending a 3rd):** C21 keyed dedup (K8s + Temporal), the C22-backoff refinement, and **C25 single-owner — HELD**: three instances establish single-ownership but for *different purposes* (anti-clobber / recovery-topology / log-consistency), so it's convergence on a *shape*, not cleanly the same reason. Holding it is the bar working (cf. the determinism refinement, P3).
- **At 1/3:** C47 (durable execution — Temporal; bridges testing-determinism C14, different purpose), C26 (spec/status), C27 (cache-sync+resync), C24 (optimistic CC, with OTP share-nothing as a counter-case), C30/C31 (OTP-only). (C28 is graduated via the *testing* corpus — P2/P4.)
- Confidence tags are honest snapshots. The graduations rest on *three mechanistically-distinct* recovery models — strong independence (stronger than a same-ecosystem trio). A 4th would only harden; the held C25 and the 1/3 set show the bar is still being enforced.
