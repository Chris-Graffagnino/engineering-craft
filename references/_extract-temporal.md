# Extraction: Temporal — Operational Robustness (Durable Execution)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/temporalio/sdk-go` (MIT) at commit `d726053`. Tier 2/3 — `workflow/doc.go` (the determinism/replay model), `temporal/error.go` (retryable vs non-retryable classification), `internal/client.go` + `internal/activity.go` (the `RetryPolicy`: `BackoffCoefficient`, `MaximumAttempts`, `MaximumInterval`). The durable-execution model is also heavily documented at docs.temporal.io (attributed inline).

**Seam discipline:** stays on the *craft* side of the boundary with `software-architect`. It extracts Temporal's **retry / recovery / durability mechanics**. It does **not** decide whether your system *should* be built on durable execution / workflow orchestration, nor whether to adopt event sourcing as a *system architecture* — those are architecture and route to the other skill. (Temporal was kept out of the *testing* corpus for DDD overlap; its robustness mechanics carry no such overlap.)

Dual purpose: (1) the **third operational-robustness witness** — a durable-execution engine, mechanistically unlike Kubernetes (reconcile-external-state) and Erlang/OTP (crash-and-restart), so the trio spans three genuinely different recovery models; (2) it **graduates the robustness core** — C19+C20, C23+C22-ceiling, C29 reach 3/3 → new **P7, P8, P9** in `_principles-index.md`. It lifts C21 and the C22 *backoff* leg to 2/3, **holds C25** (a disciplined non-graduation), and contributes one new candidate, C47. See §3a.

---

## 0. Identity

- **Repo:** Temporal Go SDK (`temporalio/sdk-go`) — the developer-facing durable-execution contracts.
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `d726053`.
- **Language(s):** Go (the SDK); the model is language-agnostic across Temporal SDKs.
- **Why it's in the corpus:** The reference example of *durable execution* — recovery as a platform guarantee via event-log replay — and the third, mechanistically-distinct robustness witness needed to graduate the reconcile/retry core.

## 1. Thesis

**Make recovery a property of the platform, not the business code: persist every state-changing step to a durable log, reconstruct state after any failure by replaying that log, and retry the side-effecting work under an explicit, bounded, classified retry policy.** Temporal teaches the same robustness core as Kubernetes and OTP — re-derive from a durable source, act idempotently, retry safely, separate recovery from logic — reached from a third direction: durable execution.

## 2. Three-tier source inventory

**Tier 1 — Codified model:** docs.temporal.io (durable execution; at-least-once activities → idempotency; deterministic workflows; retry policies) — web-attributed.

**Tier 2 — Design notes (SDK package docs):**
- `workflow/doc.go` — "Execution must be deterministic"; workflow state is reconstructed by *replay*; `SideEffect` exists precisely because arbitrary nondeterministic code "would re-execute upon replay."
- `activity/doc.go` — activities are the side-effecting unit, retried under a policy.

**Tier 3 — Code (confirm/falsify):**
- `temporal/error.go` — `NewApplicationError()` / `NewNonRetryableApplicationError()`: the retryable-vs-terminal classification.
- `internal/client.go`, `internal/activity.go` — the `RetryPolicy`: `InitialInterval`, `BackoffCoefficient` (default 2.0), `MaximumInterval` (100× initial), `MaximumAttempts` (0 = unlimited; 1 = no retries), `NonRetryableErrorTypes`.

## 3. Candidate practices (operational-robustness dimension)

### C47. Durable execution — persist an event log, replay to recover; constrain the replayed code to be deterministic

- Practice: Record every state-changing step to a durable log, and reconstruct state after a crash by **replaying** that log rather than by checkpointing live memory. For replay to reproduce state faithfully, the replayed code **must be deterministic** — route all nondeterminism (time, randomness, I/O) through the platform, which records each result once and returns the recorded value on replay.
- Evidence: **Verified** — `workflow/doc.go`: "Execution must be deterministic"; "deterministic and repeatable within an execution context"; `SideEffect` returns "the recorded result" on replay because arbitrary code "would re-execute upon replay." **Documented** — docs.temporal.io (durable execution; the deterministic-workflow constraint).
- Why / constraint: Replay-from-a-log makes recovery automatic and exact — the component resumes precisely where it was — but it imposes the determinism tax on all replayed code (no wall-clock, no `rand`, no direct I/O in the workflow path).
- Counter-case: The determinism constraint is real friction; it's worth it when long-running state must survive crashes exactly (sagas, orchestrations, money movement). For short, stateless, retryable work, a plain bounded retry (P8) is far cheaper than adopting a durable-execution runtime. *(Seam: adopting durable execution / event sourcing as the system model is `software-architect`'s call; the replay/determinism mechanics are here.)*
- Convergence: **1/3** (Temporal). **Cross-dimension bridge:** this is the *same determinism technique* as the testing dimension's deterministic substrate (C14, FoundationDB) — but applied to **production recovery-replay**, not **test-reproducibility**. It therefore does **NOT** graduate C14 (whose gate is a 3rd *deterministic-testing* witness) — same mechanism, different purpose. A 2nd durable-execution / event-sourcing engine would corroborate C47.
- Graduates to: `operational-robustness.md`

## 3a. Convergence cross-check — Temporal as the third robustness witness (three graduations, one disciplined hold)

Temporal independently witnesses the robustness core via durable execution — a third mechanism distinct from K8s's reconcile-loop and OTP's supervised-restart. With three unrelated repos these clear the **≥3 bar and graduate**.

| Practice (tags) | Kubernetes | Erlang/OTP | Temporal | Verdict |
|---|---|---|---|---|
| **Re-derive from a durable source + idempotent recovery** (C19 + C20) | level-triggered re-read of actual state; idempotent reconcile | restart re-runs `init/1` from known-good; reinit is repeatable | **replay the durable event history** to reconstruct state; activities are at-least-once → **must be idempotent** | **GRADUATES → P7** |
| **Classify transient vs terminal, then bound the retry rate** (C23 + C22-ceiling) | `TerminalError`; token-bucket ceiling | `permanent`/`transient`/`temporary`; restart-intensity ceiling | `NewNonRetryableApplicationError`; `MaximumAttempts`/`MaximumInterval` | **GRADUATES → P8** |
| **Separate the recovery mechanism from business logic** (C29) | reconcile loop ≠ workload | supervisor ≠ worker | **durable-execution platform** handles retry/replay ≠ the activity (business work) | **GRADUATES → P9** |
| Per-item *exponential backoff* (C22 backoff leg) | `ItemExponentialFailureRateLimiter` | — (restarts immediately) | `BackoffCoefficient` (2.0) | **1/3 → 2/3** — refinement of P8 (OTP is the no-backoff counter-point) |
| Queue stable identities / dedup (C21) | workqueue of de-duplicated keys | — | task queues + workflow-ID dedup (`WorkflowIDReusePolicy`) | **1/3 → 2/3** (the *dedup-by-identity* aspect) |
| **Single-owner of mutable state (C25) — HELD** | owner-ref + `IsControlledBy` guard (anti-clobber/GC) | supervisor owns its children (recovery topology) | one current run owns the workflow history (log-consistency, *by construction*) | **stays 2/3 — NOT graduated** |
| Optimistic concurrency (C24) · spec/status (C26) · cache-sync+resync (C27) | (K8s) | — | not a documented Temporal craft lesson | unchanged 1/3 |

**The disciplined hold (record it).** All three instances of C25 establish "a single owner of mutable state," but for **different purposes** — Kubernetes to stop two controllers clobbering one object (and enable GC), OTP to fix recovery responsibility, Temporal to keep a single writer to the event log. The *mechanisms* also diverge (runtime guard-check / tree topology / platform invariant). That's convergence on a *shape* but not cleanly "the same practice for the same reason," so **C25 stays 2/3**. A cleaner third witness — another system with an explicit *ownership-guard-before-mutate* check for the same anti-clobber reason — would graduate it. Holding it is the bar working, not failing (cf. the determinism refinement that stayed 2/3).

**The P8 backoff refinement.** P8 graduates on *classification + a retry ceiling* (all three witnesses). Its *per-item exponential backoff* leg reaches only 2/3 (Kubernetes + Temporal; OTP restarts immediately) — so backoff is a **calibrated refinement** of P8 (add it when retries hit a shared, overloadable downstream), exactly parallel to P3's determinism refinement. Record it in the index, don't over-state it.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            Adopting a durable-execution runtime (Temporal) for ordinary retryable work.
- Why not general: Durable execution earns its weight when long-running state must survive crashes
                   exactly (orchestrations, sagas, money movement). For short, stateless, retryable
                   tasks it is heavy infra + the determinism tax. EXTRACT the principles (re-derive
                   from a durable source P7; retry safely P8; separate recovery P9) and apply them
                   as a bounded retry loop — don't stand up a workflow engine.
```

```
- Item:            The deterministic-workflow constraint (no wall-clock/rand/IO in the path).
- Why not general: A real tax justified ONLY by replay-based recovery. A component that doesn't
                   recover by replay should NOT contort its code to be deterministic — that's the
                   testing dimension's determinism debate (C14) re-run at the robustness altitude:
                   determinism is a calibrated upgrade, not a default.
```

## 5. Open questions / corpus cross-checks

- **Graduations done:** C19+C20 → **P7**, C23+C22-ceiling → **P8**, C29 → **P9** (3/3 across Kubernetes, Erlang/OTP, Temporal — three distinct recovery models). Move them from the tracked-candidates ledger into the validated core.
- **Now 2/3, awaiting a 3rd:** C22-backoff (K8s + Temporal), C21-dedup (K8s + Temporal), and **C25** (held — needs a clean anti-clobber-guard witness).
- **C47** (durable execution) is 1/3 and bridges to the testing dimension's determinism (C14): same technique, different purpose (recovery vs reproducibility) — a 2nd event-sourcing engine would corroborate C47 *without* affecting C14.
- Robustness is now the corpus's most-validated dimension after testing — P7/P8/P9 join P1–P4. Remaining robustness candidates (C24/C26/C27/C30/C31) are single-witness and may be genuinely repo-specific.
