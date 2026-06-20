# Extraction: Erlang/OTP — Operational Robustness (Supervision & Restart Mechanics)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/erlang/otp` (Apache-2.0) at commit `f5d1255`. Tier 1 — `system/doc/design_principles/sup_princ.md` ("Supervisor Behaviour") and `gen_server_concepts.md`, plus the `-doc` attributes embedded in the source. Tier 3 — `lib/stdlib/src/supervisor.erl`, `lib/stdlib/src/gen_server.erl`. The "let it crash" philosophy is also codified in Joe Armstrong's 2003 thesis *Making reliable distributed systems in the presence of software errors* (attributed inline; not in-repo).

**Seam discipline:** this extraction stays on the *craft* side of the boundary with `software-architect`. It extracts supervision/restart **mechanics** — how to structure a self-healing component. It does **not** argue "should you adopt the actor model / share-nothing concurrency" as a system architecture — that choice is architecture and routes to the other skill.

Dual purpose: (1) the **second witness** for the operational-robustness dimension — it independently corroborates several Kubernetes candidates (C19–C27), lifting them from 1/3 to **2/3** (see §3a); (2) it contributes three new candidates (C29–C31) for self-healing craft Kubernetes does not teach.

---

## 0. Identity

- **Repo:** Erlang/OTP (`erlang/otp`) — specifically the STDLIB `supervisor`/`gen_server` behaviours and the OTP Design Principles.
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `f5d1255`. Thesis is robustness *mechanics*, not decision-evolution, so no full history.
- **Language(s):** Erlang.
- **Why it's in the corpus:** The canonical example of fault-tolerance-by-supervision, and the **most independent** possible second witness for the reconciliation/retry principles — a different language, era (1980s–90s telecom), and concurrency model (share-nothing actors) than Kubernetes. That independence is exactly what the convergence test (rule 3) values; convergence between two systems this unrelated is strong evidence the practice is a principle, not a Go-ecosystem habit.

## 1. Thesis

**Build self-healing components by isolating failure and delegating recovery to a *separate* supervisor that restarts the failed part from a known-good initial state — rather than trying to anticipate and handle every error inline ("let it crash").** Robustness comes from structured, bounded restart and clear fault-containment, not from defensive code in the worker.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `system/doc/design_principles/sup_princ.md` — "Supervisor Behaviour": the supervision-tree model, restart strategies, restart types, and Maximum Restart Intensity, all stated as the project's own design doctrine.
- `gen_server_concepts.md` — the generic-server behaviour: `init/1` as the state constructor re-run on (re)start.
- Armstrong (2003), *Making reliable distributed systems…* — the "let it crash" / supervised-recovery thesis (web-attributed).

**Tier 2 — Design notes:**
- The `-moduledoc`/`-doc` attributes embedded in `supervisor.erl` (restart-type semantics, intensity/period defaults) — design notes co-located with code.

**Tier 3 — Code (confirm/falsify):**
- `lib/stdlib/src/supervisor.erl` — restart strategies (`one_for_one`/`one_for_all`/`rest_for_one`/`simple_one_for_one`), restart types (`permanent`/`transient`/`temporary`), intensity/period accounting (`add_restart`). No built-in restart backoff (grep: 0) — restarts are immediate; this matters for the C22 cross-check.
- `lib/stdlib/src/gen_server.erl` — the behaviour whose `init/1` rebuilds state on restart.

## 3. Candidate practices (operational-robustness dimension)

### C29. Separate failure-recovery from business logic — "let it crash," supervise the restart

- Practice: Don't scatter defensive error handling through worker code. Let a process *crash* on an unexpected fault, and put recovery in a **separate** supervisor whose only job is to restart the child from its known-good start function. The worker stays focused on the happy path; the recovery policy lives in one place.
- Evidence: **Codified** — `sup_princ.md`: "A supervisor is responsible for starting, stopping, and monitoring its child processes. The basic idea of a supervisor is that it is to keep its child processes alive by restarting them when necessary." **Verified** — `supervisor.erl` implements exactly this; the worker (`gen_server`) carries no restart logic. **Documented** — Armstrong's thesis frames "let it crash" as the reliability strategy.
- Why / constraint: Code that tries to handle every error inline is both bug-prone (error paths are the least-tested code) and tangled. Crashing to a clean, separately-owned recovery point makes the failure path simple and uniform.
- Counter-case: Requires that a restart from known-good state is *cheap and safe* — i.e. state is reconstructible (C20) and a moment of unavailability during restart is acceptable. For a long-lived in-memory computation with no durable source to re-derive from, "let it crash" loses that state; you need checkpointing or persistence first. Also assumes fault-isolation between worker and supervisor (separate processes).
- Convergence: Kubernetes separates the recovery mechanism (the controller's reconcile loop) from the workload it manages, for the same reason — recovery is a distinct concern with its own loop. **2/3** (K8s + OTP). A third (a durable-execution engine, a resilience framework) would graduate it.
- Graduates to: `operational-robustness.md`

### C30. Scope the restart blast radius to fate-sharing (restart strategies)

- Practice: When restarting a failed component, restart exactly the set that shares fate with it — no more, no less. OTP makes this an explicit choice per supervisor: `one_for_one` (restart only the failed child), `one_for_all` (restart all siblings — they depend on each other), `rest_for_one` (restart the failed child and those started after it), `simple_one_for_one` (a dynamic pool of identical children).
- Evidence: **Codified + Verified** — `sup_princ.md` and `supervisor.erl` define the four strategies; the doc ties each to the dependency structure among children.
- Why / constraint: Restarting too little leaves siblings talking to a process that lost its peer's state; restarting too much needlessly interrupts healthy work. The right scope is the *fate-sharing set* — components whose state is coupled.
- Counter-case: Only meaningful when you supervise *multiple* related components. A single isolated worker has a trivial blast radius (itself). Over-coupling (`one_for_all` where children are actually independent) converts one failure into a fleet-wide restart — the anti-pattern.
- Convergence: _1/3_ — Kubernetes does not scope restart blast radius this way (each resource reconciles independently); this is OTP-specific craft so far. A second witness would be another supervision/actor framework (Akka, Elixir) — but those are not *independent* of OTP, so they wouldn't strengthen convergence much.
- Graduates to: `operational-robustness.md`

### C31. Bound restart intensity, then escalate — don't retry forever in place

- Practice: Cap how many restarts may occur in a time window (`intensity`/`period` — default 1 per 5 s); if exceeded, the supervisor gives up, terminates its children, and **fails upward** so a higher level can take a coarser recovery action. Retry is bounded *and* has an escalation path, rather than looping forever at one level.
- Evidence: **Codified** — `sup_princ.md` ("Maximum Restart Intensity"): "If more than `MaxR` number of restarts occur in the last `MaxT` seconds, the supervisor terminates all the child processes and then itself… the next higher-level supervisor takes some action." Stated intent: "prevent a situation where a process repeatedly dies for the same reason, only to be restarted again." **Verified** — `supervisor.erl` intensity/period accounting; defaults `intensity=1`, `period=5`.
- Why / constraint: A component that fails for a deterministic reason will never recover by local retry — retrying just floods logs and burns resources. Bounding the rate and escalating hands the problem to a level that can clear it differently (restart a larger subtree, fail the node).
- Counter-case: The bound is a tunable, not a constant — `sup_princ.md` devotes a whole "Tuning" section to it ("These choices depend a lot on your problem domain"): tolerate bursts vs. cap sustained rate; an embedded system with no operator might accept one restart per minute then escalate, a high-availability path might sustain 1–2/s. Set per blast radius.
- Convergence: The *ceiling* (bound total retry throughput so a persistent failure can't hot-loop) converges strongly with Kubernetes's token-bucket rate limiter — see C22 in §3a. The *escalate-up-a-hierarchy* response is OTP-specific (K8s rate-limits in place rather than failing upward); that escalation refinement is **1/3**.
- Graduates to: `operational-robustness.md`

## 3a. Convergence cross-check — Erlang/OTP as the second operational-robustness witness

The operational-robustness dimension had a single witness (Kubernetes, C19–C27 all 1/3). Erlang/OTP independently witnesses several of those practices *for the same reason* via a completely different mechanism, lifting them to **2/3**. Nothing reaches the ≥3 graduation bar yet — the dimension still needs a third independent witness — so none of these promote to `_principles-index.md`; this is honest 2/3 corroboration.

| Practice (K8s tag) | Kubernetes mechanism | Erlang/OTP mechanism | Verdict |
|---|---|---|---|
| **Re-derive state, don't trust deltas** (C19) | level-triggered: re-read actual state each cycle | restart re-runs `init/1`, rebuilding state from known-good rather than trusting corrupted in-memory state | **1/3 → 2/3** — convergent *reason* (don't trust in-flight/corrupt state); mechanism differs (continuous re-read vs crash-triggered re-init) |
| **Idempotent recovery** (C20) | idempotent reconcile (create-if-missing/update-if-different) | restart-to-clean-init is repeatable; "let it crash" assumes restart is safe to repeat | **1/3 → 2/3** |
| **Bound retry throughput so failures can't hot-loop** (C22, ceiling leg) | token-bucket ceiling across all items | restart intensity (`MaxR`/`MaxT`) → terminate+escalate | **1/3 → 2/3** for the *ceiling* leg. NOT the per-item exponential-backoff leg — OTP restarts immediately (no backoff); and OTP *escalates* where K8s rate-limits in place |
| **Distinguish transient from terminal** (C23) | `TerminalError` absorbed (no requeue); transient → requeue | restart type: `transient` (restart on abnormal exit only), `permanent` (always), `temporary` (never) | **1/3 → 2/3** — the strongest match; both classify failures into retry-vs-absorb |
| **Single-owner guard** (C25) | `IsControlledBy` check; refuse to act on un-owned resources | a supervisor structurally owns exactly its children; a process is terminated by its own supervisor | **1/3 → 2/3** — *structural* single-ownership converges; OTP enforces it by tree topology, K8s by a runtime guard check |
| Queue keys, dedup (C21) | workqueue of de-duplicated identity keys | — (mailboxes are per-process FIFO, not keyed dedup) | **stays 1/3** — not witnessed |
| Optimistic concurrency control (C24) | re-fetch + retry on version conflict, no locks | — **deliberate counter-case:** share-nothing actors have no shared mutable state, so there is no read-modify-write conflict to resolve | **stays 1/3** + counter-case (see §4) |
| Spec/status separation (C26) | `status` subresource + `observedGeneration` | — (no core OTP analogue) | **stays 1/3** — not witnessed |
| Cache-sync gate + periodic resync (C27) | `WaitForCacheSync`; periodic drift resync | weak: children start in dependency order (gate-ish), but no periodic drift resync | **stays 1/3** — at most a partial echo of the start-ordering gate |

**The key result.** Five of the nine robustness candidates (C19, C20, C22-ceiling, C23, C25) reach **2/3** on a maximally-independent witness — strong evidence they are real principles, not Kubernetes idioms. Four (C21, C24, C26, C27) are *not* witnessed and stay honestly at 1/3; C24 earns a sharp **counter-case** (share-nothing is the alternative answer to coordination). The new candidate C29 (separate recovery from logic) also lands at 2/3 immediately. A third robustness witness (a durable-execution engine like Temporal, or a chaos/resilience framework) would be needed to graduate any of these into `_principles-index.md`.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            A deep static supervision tree for everything.
- Why not general: Justified for a long-running system of many interdependent stateful processes
                   that must survive partial failure. For a short script, a stateless request
                   handler, or a linear job, a multi-level supervision hierarchy is ceremony.
                   EXTRACT the principles (isolate failure; restart from known-good; bound and
                   escalate) and apply them as a single bounded retry/restart, not a tree.
```

```
- Item:            "Let it crash" as a universal error strategy.
- Why not general: It depends on cheap, safe restart from reconstructible state and on process-
                   level fault isolation (the BEAM gives both). In a language/runtime without
                   isolation, an uncaught crash can corrupt shared memory or take down the whole
                   process — there, crashing is not a recovery strategy. Adopt the PRINCIPLE
                   (separate recovery, restart to known-good) only where restart is actually safe.
```

```
- Item:            Immediate restart with no backoff (OTP default).
- Why not general: OTP relies on the intensity ceiling (then escalate) instead of per-attempt
                   backoff. For retries against a shared, overloadable downstream (an API, a DB),
                   immediate restart can hammer it — there you want Kubernetes-style exponential
                   backoff (C22's other leg) in addition to a ceiling. The two systems each
                   implement one leg of the same idea; a robust retry usually wants both.
```

## 5. Open questions / corpus cross-checks

- This lifts C19, C20, C22 (ceiling), C23, C25, and C29 to **2/3**. A **third** robustness witness would graduate them: a durable-execution engine (Temporal's retry-policy/idempotency/non-retryable-error mechanics — explicitly robustness craft, not the DDD overlap that kept it out of the testing corpus) is the strongest candidate, and would also likely witness C20/C23 directly.
- C24 (optimistic concurrency) now has a documented **counter-case** (share-nothing actors), not a second witness — it stays 1/3 and the boundary is sharper for it.
- C21 (keyed dedup), C26 (spec/status), C27 (resync) remain Kubernetes-only — look for them in a second reconciler-style system, not in OTP.
- C30 (restart-strategy scoping) and the escalation refinement of C31 are OTP-specific so far; the obvious corroborators (Akka, Elixir) descend from OTP and so add little *independent* weight — note this when tempted to cite them.
