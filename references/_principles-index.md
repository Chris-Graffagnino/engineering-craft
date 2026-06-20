# Principles Index — Convergence-Validated Core

The capstone reference. This file holds **only** practices witnessed independently in **≥3 unrelated corpus repos** — the high-confidence, cross-cutting core of the skill. Everything below earned its place by convergence, not by any single repo's authority. Candidates with fewer witnesses live in the *Tracked candidates* ledger at the bottom and in the per-dimension references; they are **not** promoted here until a third independent witness appears.

**How Apply mode uses this:** when a craft question is in scope, these are the principles to lead with — they're the ones the corpus most strongly supports. Always pair each with its boundary and the governing dial; never deliver a principle as a bare imperative.

**Corpus behind the graduations: all four dimensions have now graduated principles — the v1 core is complete.** Testing — the **P0** dial + **P1–P4** — across SQLite, FoundationDB, Kubernetes. Simplicity — **P5** (understandability), **P6** (minimal deps) — across Redis, SQLite, Go (Go the cross-paradigm anchor over the Redis↔SQLite correlation). Robustness — **P7** (re-derive + idempotent), **P8** (retry safely), **P9** (separate recovery) — across Kubernetes, Erlang/OTP, Temporal (reconcile / supervised-restart / durable-replay). Extensibility — **P10** (narrow versioned contract + registration), **P11** (declared maturity/posture), **P12** (governed lifecycle) — across Envoy, Linux, VS Code (in-tree filters / in-tree modules / third-party marketplace). **P0 surfaces as the dial in every dimension**; **P2** also surfaces in **P10** (the extension-side seam — not double-counted). Each graduation carries an explicit independence note where its witnesses are correlated (see P5/P6, P10). Provenance: `_extract-sqlite.md`, `_extract-foundationdb.md`, `_extract-kubernetes.md`, `_extract-erlang-otp.md`, `_extract-envoy.md`, `_extract-linux.md`, `_extract-redis.md`, `_extract-go.md`, `_extract-temporal.md`, `_extract-vscode.md`.

---

## P0 — The governing dial: calibrate rigor to blast radius × patch latency

*Witnessed by all three, and the meta-principle that governs how much of P1–P4 to apply.*

The level of engineering investment in correctness should scale with **consequence-of-failure × how slowly you can ship a fix**. This is not itself a technique — it sets the dial on every technique below.

- SQLite runs ~590:1 test-to-code and 100% MC/DC **because** it's ubiquitous and effectively un-patchable in the field.
- FoundationDB pays for whole-system deterministic simulation **because** correctness-under-failure at distributed scale is its product.
- Kubernetes deliberately accepts *non-deterministic, flaky* e2e **because** its blast-radius/patch profile differs from an embedded DB — a different dial setting, consciously chosen.

**Operational rule:** extract the *calibration*, never the *setting*. Copying SQLite's coverage standard or FoundationDB's simulation regime into an ordinary, fast-to-patch service is waste; under-testing a hard-to-patch critical kernel is negligence. Set the dial first, then apply P1–P4 to depth.

Corollary (also witnessed): **a coverage/scale number is not a correctness score.** Metrics conflict (SQLite documents MC/DC vs fuzzing tension); know what each buys.

---

## P1 — Independent redundant verification

**Verify correctness through multiple harnesses developed and maintained separately, so they fail on disjoint blind spots. Independence — not test volume — is the active ingredient.**

- **Witnesses:** SQLite — four independent harnesses + multiple independent fuzzers. FoundationDB — simulation + Circus performance + hardware-failure regimes. Kubernetes — unit + integration (envtest) + e2e + conformance tiers, with e2e explicitly for "hard-to-test bugs when unit and integration are insufficient."
- **Boundary / dial:** cost scales with each added path. Under low blast radius, one good harness + CI is the rational stop; a *second independent* path is for the critical kernel (a crypto primitive, a billing core, a consensus layer), not the whole app.
- **Cross-skill:** pure craft; no `software-architect` handoff.
- Source: C5 (`_extract-sqlite.md`, `_extract-foundationdb.md`, `_extract-kubernetes.md`).

## P2 — Design for testability via injection seams

**Build seams into the production interface where a real resource (allocator, filesystem, clock, network, API client) can be swapped for an instrumented one. You cannot test a failure you have no seam to inject at.**

- **Witnesses:** SQLite — pluggable `malloc` + VFS. FoundationDB — the entire system on Flow, so all I/O/time/network is swappable. Kubernetes — fake clientset (unit) + envtest's real local control plane (integration).
- **Boundary / dial:** seams add indirection and public surface — place them at *failure-injection and portability boundaries*, not everywhere. "Inject everything" is the DI-everywhere anti-pattern (accidental complexity). Note SQLite's testability seams (VFS) are the same ones that give portability — the mechanisms often coincide.
- **Cross-skill:** a seam viewed from the *other* side is an extension point — now drafted as `extensibility-boundaries.md` (§2/§3.1, sourced from Envoy's filter contract C32), where this same narrow-interface construct is extracted from the extension side. P2 owns the shared mechanism (don't double-count it as a separate graduation); keep the test-side instance here and route plugin-architecture questions to that dimension / `software-architect`.
- Source: C10 (all three).

## P3 — Test under systematically injected faults, then assert an invariant

**Don't hope failure paths work — simulate failures systematically (not just happy-path inputs) and assert, after each, that a correctness invariant still holds. The canonical shape: instrument a resource to fail after N operations, loop advancing N until completion, and check the invariant each time.**

- **Witnesses:** SQLite — OOM / I/O-error / crash injected in deterministic loops, `PRAGMA integrity_check` + transaction-atomicity asserted after. FoundationDB — machine / network / datacenter faults in whole-cluster simulation, in-sim validation. Kubernetes — `[Disruptive]` e2e takes down nodes and makes the API server unresponsive to test fault-tolerance and disaster-recovery.
- **Two legs that always co-occur:** the **injection seam** (P2) reaches the failure, and an **executable invariant** (assertion / integrity check) catches the corruption. Without the invariant the injection is silent; without the seam it can't reach the path.
- **Boundary / dial:** depth scales with blast radius (P0). Exhaustive fail-at-every-point loops are for commit/recovery/consensus paths where partial failure corrupts state; a stateless handler whose worst case is "return 500 and retry" needs only a few targeted error tests.
- **REFINEMENT (not graduated — 2/3): make the substrate deterministic for reproducibility.** SQLite (seeded loops) and FoundationDB (full deterministic simulation) add a seeded, controllable substrate so any failure *replays exactly*. **Kubernetes is the counter-point** — it injects faults on real, non-deterministic clusters and pays the well-known price: flaky e2e. Determinism buys reproducibility (replay a bug from a seed, add logging, re-run identically) at the cost of architecting the whole system for it. **Worth its price when reproducibility dominates** (embedded DB, distributed transaction engine); deliberately skipped when it doesn't. Promote this refinement to a full principle when a third deterministic-simulation repo enters the corpus.
- **Cross-skill:** pure craft.
- Source: C7 base + C8 (all three); C14 refinement (SQLite, FDB).

## P4 — Same code in test and production

**Test the real code paths, not a model or mock that can drift. Achieve it by writing production code against abstractions that have both real and simulated implementations, and by testing against the real dependency where feasible.**

- **Witnesses:** SQLite — the real library runs under instrumented `malloc`/VFS, not a reimplementation. FoundationDB — Flow emits production binaries *and* drives simulation; workloads are shared across simulation and perf. Kubernetes — envtest runs the *real* etcd + kube-apiserver; conformance runs on real clusters.
- **Boundary / dial:** requires the dual-implementation substrate (P2). Where it's absent, a model may be unavoidable — but then the model's *fidelity to production* becomes a standing risk you must actively manage. A mock encodes your belief about behavior; the real dependency encodes the actual behavior.
- **Cross-skill:** pure craft.
- Source: C15 (all three).

## P5 — Hold an understandability budget; complexity is lock-in

**Make "a bounded reader can understand the code" an explicit, defended constraint — and treat complexity as a form of lock-in: code only its author understands cannot be safely changed by anyone else, regardless of license. The cheapest complexity to manage is the complexity you never create.**

- **Witnesses:** Redis — MANIFESTO #6 ("remain understandable… a single programmer… reading the source code for a couple of weeks"; "Complexity is also a form of lock-in"; "the best way to fight complexity is by not creating it at all"). SQLite — heavily-commented source (`btree.c` ~31% comment-leading) whose clarity is *bought by* the test regime (refactor freely; the tests catch regressions). Go — a ~42k-word *whole-language* spec, `gofmt` (one canonical format), the Proverb "Clear is better than clever," features omitted to stay small.
- **Boundary / dial:** the budget *scales* — "read the whole system in two weeks" doesn't survive a kernel or compiler, where the unit of comprehensibility becomes the *module*, not the whole. P0 governs *how much* simplicity to spend.
- **Independence note (read this):** Redis and SQLite share a single-author-C-systems profile, so two of the three witnesses are correlated; **Go (a multi-author language/toolchain) is the cross-paradigm independence anchor** that makes the trio credible. This is comparable independence texture to P1–P4 (whose witnesses are 2 databases + 1 orchestrator) — graduated to the same standard. A 4th witness from a non-C, non-systems paradigm would harden it further.
- **Cross-skill:** pure craft — and the *meta-value* of this skill's own methodology (extract the calibration not the setting; the boundaries are the product). Lives in `implementation-simplicity.md` §2.
- Source: C3 (`_extract-redis.md`, `_extract-sqlite.md` §6, `_extract-go.md`).

## P6 — Minimize the dependency surface; prefer self-contained code you understand

**Every dependency is code you don't understand, a supply-chain and attack surface, and a constraint on your own evolution. Default to small, self-contained implementations you fully grasp; reach for an external library only when it is mature, self-contained, and genuinely better than what you'd write.**

- **Witnesses:** Redis — MANIFESTO #5 ("write smaller stories that fit into other code"; few deps). SQLite — *zero external dependencies*, shipped as a single-file amalgamation (the extreme exemplar). Go — "A little copying is better than a little dependency" (Proverb); static self-contained binaries; a self-contained batteries-included stdlib.
- **Boundary / dial:** **NOT "no deps" — *minimal, understood* deps.** Reinventing crypto/TLS/compression is far worse than depending on a vetted library (all three use mature libraries where warranted). Calibrate the build-vs-depend call to blast radius (P0): own the small, well-understood piece; depend on the hard, security-critical one.
- **Independence note:** same caveat as P5 — Go is the cross-paradigm anchor over the Redis↔SQLite correlation.
- **Cross-skill:** the *craft* instance lives here (`implementation-simplicity.md` §3.4); **Envoy's dependency *governance*** (C37, extensibility) and `software-architect`'s supply-chain concerns are the same value at the surface/system altitudes — route "how should we govern dependency policy" there.
- Source: C45 (`_extract-redis.md`, `_extract-sqlite.md` §6, `_extract-go.md`).

## P7 — Self-heal by re-deriving state from a durable source, converging idempotently

**Recover by reconstructing state from an authoritative/durable source rather than trusting volatile in-flight state — and act idempotently, so re-running recovery is a no-op. The two legs co-occur: re-derivation is only safe to repeat because the action is idempotent.**

- **Witnesses (three distinct recovery models):** Kubernetes — level-triggered reconcile re-reads actual state from the apiserver each cycle + idempotent reconcile (create-if-missing / update-if-different). Erlang/OTP — restart re-runs `init/1` to rebuild from known-good + reinit is repeatable. Temporal — **replay the durable event history** to reconstruct workflow state + activities run at-least-once and so **must be idempotent**.
- **Boundary / dial:** requires a re-queryable or durable source of truth; a pure event stream with no re-readable/replayable state instead makes exactly-once + ordering a hard requirement you own. Non-idempotent side effects (charge a card, send mail) need an idempotency key before they're safe behind any recovery loop. P0 governs how much recovery machinery to build.
- **Cross-skill:** pure craft. *(Temporal's mechanism — durable-log replay — requires deterministic replayed code, candidate C47, which bridges to the testing dimension's determinism: same technique, recovery vs reproducibility.)*
- Source: C19 + C20 (`_extract-kubernetes.md`, `_extract-erlang-otp.md`, `_extract-temporal.md`).

## P8 — Retry safely: classify transient vs terminal, then bound the retry rate

**Retry *transient* failures under a bounded policy; *absorb* terminal/non-retryable ones (return success / stop) so they don't hot-loop. Classification is the craft; the bound stops one persistent failure from melting a shared downstream.**

- **Witnesses:** Kubernetes — `TerminalError` (no requeue) + a token-bucket ceiling. Erlang/OTP — `permanent`/`transient`/`temporary` restart types + a restart-intensity ceiling (then escalate). Temporal — `NewNonRetryableApplicationError` / `NonRetryableErrorTypes` + `MaximumAttempts` / `MaximumInterval`.
- **REFINEMENT (2/3, not graduated): per-item exponential backoff.** Kubernetes (`ItemExponentialFailureRateLimiter`) and Temporal (`BackoffCoefficient`) back off per item; **OTP restarts immediately** — the no-backoff counter-point. So backoff is a *calibrated* refinement (add it when retries hit a shared, overloadable downstream; skip it when immediate retry + a global ceiling suffices), structurally identical to P3's determinism refinement.
- **Boundary / dial:** caps / backoff / bucket sizes are tunables, not constants to copy. Misclassification cuts both ways — transient-as-terminal silently drops work; terminal-as-transient hot-loops. Demands a deliberate error taxonomy.
- **Cross-skill:** pure craft.
- Source: C23 + C22-ceiling (all three); C22-backoff (Kubernetes, Temporal — 2/3 refinement).

## P9 — Separate the recovery mechanism from the business logic

**Don't thread recovery/retry logic through the business code; delegate it to a separate mechanism — a supervisor, a reconcile loop, a durable-execution platform — that restarts / retries / replays the work from a known-good point. The worker stays on the happy path; recovery policy lives in one place.**

- **Witnesses:** Kubernetes — the reconcile loop *is* the recovery mechanism, distinct from the workload. Erlang/OTP — a supervisor restarts the worker, which carries no restart logic ("let it crash"). Temporal — the durable-execution platform handles retry/replay/recovery; the activity is just the business work.
- **Boundary / dial:** requires that recovery-from-a-known-point is cheap and safe — state reconstructible (P7) — and that the work and its recovery mechanism are fault-isolated. With irreplaceable in-memory state and no checkpoint, "let it crash" destroys data.
- **Cross-skill:** pure craft.
- Source: C29 (`_extract-kubernetes.md`, `_extract-erlang-otp.md`, `_extract-temporal.md`).

## P10 — Extend through a narrow, versioned contract with decoupled registration — never into the core

**Let extensions attach only through a small, explicitly-versioned contract, registered declaratively (by name / manifest, not hardwired into core) and signalling back through a fixed protocol — never reaching into core internals. Adding an extension is then purely additive, and core can evolve its internals freely behind the contract.**

- **Witnesses (three extension models):** Envoy — filter interface + `FactoryRegistry` (named registration) + typed proto config. Linux — `EXPORT_SYMBOL` (the only surface a module may call) + `module_init` registration. VS Code — the 21k-line typed `vscode.d.ts` + declarative `contributes`/`activationEvents` + `engines.vscode` version-compat.
- **The graduated boundary — version the contract *iff* you can't fix your consumers.** Linux holds extensions *in-tree* (the breaker fixes every driver), so it keeps the *internal* API deliberately unstable; VS Code's extensions are *third-party/out-of-tree* (unfixable), so it *freezes* the contract and stages all change behind proposed-API; Envoy sits between (in-tree extensions, versioned proto for external operators). Calibrate by whether you can fix your consumers — not a flat "always version."
- **Cross-skill:** the bare narrow-interface leg is the **extension-side face of P2** (the injection seam from the test side) — P2 owns that shared mechanism; P10 graduates the *extension-specific* additions (declarative registration, versioned config, the one-directional boundary). Don't double-count. The architectural choice to *be* extensible → `software-architect`.
- Source: C32 + C33 + C34 (`_extract-envoy.md`, `_extract-linux.md`, `_extract-vscode.md`).

## P11 — Declare each extension's maturity and trust posture as machine-readable metadata

**Tag every extension with an explicit, schema-checked maturity level and (where there's a trust boundary) a security/capability posture — and make "unvetted" a forcing, visible value, not an absence. The metadata lets consumers, tooling, and security reviewers reason about what enabling an extension costs them.**

- **Witnesses:** Envoy — `status` (stable/alpha/wip) + `security_posture` (robust-to-untrusted / requires-trusted / unknown), schema-checked, `unknown` the largest bucket *on purpose*. Linux — `MAINTAINERS` `S:` status (Supported/Maintained/Orphan/Obsolete) + `EXPORT_SYMBOL_GPL` license posture + kernel taint. VS Code — `preview` (maturity) + `capabilities`/`untrustedWorkspaces`/`virtualWorkspaces` (trust posture).
- **Boundary / dial:** worth it at surface scale and where a trust boundary exists; keep implementation maturity orthogonal to contract-API maturity; the honest move is that the "unvetted/unmaintained" value is *visible and common*, surfacing risk instead of implying safety.
- **Cross-skill:** pure craft (the security-team reasoning it enables borders `software-architect`'s supply-chain / threat-model concerns).
- Source: C35 (`_extract-envoy.md`, `_extract-linux.md`, `_extract-vscode.md`).

## P12 — Govern the extension lifecycle — stage and graduate new surface, deprecate on a timer

**Treat the extension surface as having a lifecycle. Add new surface deliberately (ideally staged behind an opt-in until proven), and remove it on a governed timeline (deprecation window / graduation pipeline) — never break consumers abruptly, never let unused surface linger untested.**

- **Witnesses:** Envoy — governed extension-point addition + a 6-month timed deprecation (factory emits a warning). Linux — `drivers/staging/` → mainline graduation; unused interfaces deleted. VS Code — proposed-API (175 staged files) → stable promotion; API deprecation.
- **REFINEMENT (2/3): stage new surface behind an explicit opt-in, then promote (C48).** VS Code (proposed-API) + Linux (staging) gate experimental surface behind opt-in and graduate it; Rust's nightly feature-gates would be the 3rd. A calibrated refinement — worth it when third-party consumers need stability while the surface evolves.
- **Boundary / dial:** the deprecation window / graduation bar trades migration runway against carrying cost — size to how widely the surface is used. Extension *points* are long-lived contracts; add them deliberately because removing them is expensive.
- **Cross-skill:** pure craft.
- Source: C38 (all three); C48 refinement (VS Code, Linux).

---

## How the "composite" decomposed

Earlier extractions flagged a recurring composite — *injection seam + deterministic substrate + invariant assertion*. Convergence resolved it cleanly: the **seam (P2)** and the **invariant (a leg of P3)** graduate at 3/3, and **fault injection (P3)** graduates as the act that uses them. The **deterministic substrate** is the one leg that did **not** reach 3/3 — it's a calibrated refinement (P3, above), with Kubernetes as the explicit counter-example. So the durable, universal core is *seam + injected fault + invariant*; determinism is the high-cost upgrade you add when reproducibility is worth it.

---

## Tracked candidates (NOT yet graduated)

Honest ledger. These are real and useful but under-witnessed in the current corpus; they stay out of the validated core until a third independent witness appears. Promote by adding the repo that corroborates.

| Candidate | Witnesses | Status | What would graduate it |
|---|---|---|---|
| Deterministic substrate (P3 refinement) | SQLite, FoundationDB | **2/3** | A 3rd deterministic-simulation repo (TigerBeetle, Antithesis-tested systems) |
| Tiered test depth by feedback latency (C12) | SQLite (veryquick/full/soak), Kubernetes (unit/integration/e2e) | **2/3** | One more repo with explicit speed-tiered suites |
| Regression ratchet — every bug → permanent test (C11) | SQLite (confirmed); Kubernetes (suggestive via growing conformance suite) | **1/3 + suggestive** | Confirm K8s contributor "fix ships with a test" policy; check Envoy/Redis |
| Differential testing against an oracle (C6) | SQLite | **1/3** | A compiler/codec/protocol repo (very likely corroborates) |
| Adversarial fault design (C17, "swizzle-clog") | FoundationDB | **1/3** | A Jepsen-tested or chaos-engineering repo |
| Compiled-out in-code failpoints (C16, BUGGIFY) | FoundationDB | **1/3** | TiKV / etcd (failpoints) |
| Optimistic concurrency control (C24) | Kubernetes (+ Erlang/OTP share-nothing **counter-case**) | **1/3** | Any DB/coordination repo using version-conflict retry |
| **Single-owner of mutable state (C25) — HELD at 2/3** | Kubernetes (anti-clobber guard), Erlang/OTP (recovery topology), Temporal (single-writer by construction) | **2/3** — convergence on a *shape*, but three different *purposes*, so not cleanly same-reason | A clean *ownership-guard-before-mutate* witness with the anti-clobber purpose |
| Per-item exponential backoff (C22 *backoff* leg — a refinement of P8) | Kubernetes, Temporal | **2/3** | A 3rd witness with per-item backoff (OTP is the no-backoff counter-point) |
| Keyed dedup by stable identity (C21) | Kubernetes, Temporal (workflow-ID dedup) | **2/3** | A 3rd queue/dispatch system that dedups by identity |
| Spec/status separation (C26) · cache-sync + resync (C27) | Kubernetes | **1/3** | A 2nd reconciler-style system (not witnessed in OTP/Temporal) |
| Durable execution — log + deterministic replay (C47) | Temporal | **1/3** | A 2nd event-sourcing / durable-execution engine (bridges testing-determinism C14 — *different purpose*) |
| Restart-blast-radius scoping (C30) · escalate-on-intensity refinement (C31) | Erlang/OTP | **1/3** | OTP-only so far; Akka/Elixir descend from OTP so add little independent weight |
| Executable invariants compiled out in prod (C8, standalone) | SQLite, FoundationDB | **2/3** | One more systems repo using assert-heavy invariants (near-certain) |
| Avoid *gratuitous* layers / small composable interfaces (C43) | Redis, Go | **2/3** (capped — SQLite is the counter-case: an inherently-layered SQL engine) | A non-layered-domain 3rd witness; may be inherently calibrated, not universal |
| API honesty — don't fake capabilities (C4) | Redis, Go | **2/3** | A 3rd repo with explicit failure contracts (a DB/protocol library) |
| Simplest execution model (C42) | Redis + SQLite (*avoid* concurrency) ; Go (*simplify* concurrency — counter-point) | **2/3, two valid forms** | Not a single graduate — it's a dial: avoid the machinery, or make it cheap, per workload |
| Special-case compact representations (C44) | Redis, SQLite (Go neutral/mild-counter) | **2/3 moderate** | A storage/codec repo |
| Mechanically enforce one canonical format (C46) | Go (gofmt) | **1/3** | Envoy's mandated `clang-format` (in-corpus, would lift to 2/3); rustfmt/Prettier/Black broadly |
| Structural ownership of extensions (C1) — HELD | Envoy (`CODEOWNERS`), Linux (`MAINTAINERS`), VS Code (marketplace `publisher` — *weaker*) | **2/3** | A clean *in-repo path→owner* 3rd witness — VS Code's is marketplace-level, not a graduating instance |
| In-tree extension model (C39) | Envoy, Linux | **2/3** — VS Code is the *out-of-tree counter-pole* (completes the P10 in-tree/out-of-tree refinement) | Inherently calibrated by whether you can fix your consumers; may not be a universal |
| Stage-behind-opt-in maturation pipeline (C48, refinement of P12) | VS Code (proposed-API), Linux (`staging`) | **2/3** | Rust nightly feature-gates (the obvious 3rd; rhymes with Envoy `v3alpha`) |
| Dependency-hygiene specifics (C37) · prune-unused-surface (C40) | Envoy (C37) / Linux (C40) | **1/3 each** | A 2nd security-critical extension surface (C37); a 2nd self-pruning codebase (C40) |

---

## Provenance & maintenance

- Graduation rule: **≥3 unrelated corpus repos**, each evidenced with a calibrated verb (Verified / Documented / …) in the source extraction. No exceptions — the rule is the credibility mechanism.
- When a new repo is distilled, re-run the convergence cross-check (§3a of each extraction), move any candidate that reaches 3/3 up into the validated core, and update the ledger.
- Every validated principle here is reachable from a per-dimension reference — **all four now drafted**: `testing-correctness.md`, `operational-robustness.md`, `extensibility-boundaries.md`, `implementation-simplicity.md`. This index is the cross-cutting view, the dimension references are the problem-indexed views, and the `_extract-*.md` files are the evidence. **P0 surfaces in every dimension as its dial** — rigor (testing), machinery (robustness), governance tiers (extensibility/C36), and the complexity budget (simplicity/C41); **P2 also surfaces in `extensibility-boundaries.md`** (the extension-side seam). These are the same graduated principles viewed at different altitudes, not new graduations.
