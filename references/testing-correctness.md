# Testing & Correctness — Dimension Reference

**Role:** consumed in **Apply mode**. Load this when a craft question touches testing strategy, error/failure-path testing, test *quality* (not just coverage), flakiness/reproducibility, or "how rigorously should we test this." Synthesize from it — do not quote it at the user.

**Sources:** distilled from the corpus's two testing exemplars, SQLite and FoundationDB, with **Kubernetes as the third witness** that corroborated the core practices (`_extract-kubernetes.md`). Per-practice evidence and provenance live in `_extract-sqlite.md` and `_extract-foundationdb.md`; this file is the operational version. Practice tags (C5–C18) cross-reference those extractions. The core practices have cleared the ≥3 bar and **graduated into `_principles-index.md` as P1–P4**; tags below mark each practice's current status. A few legs remain calibrated refinements at *2/3* by design — notably the deterministic-substrate (C14, §5.1), where Kubernetes is the deliberate counter-point.

---

## Contents
- §1 The master dial — calibrate rigor to blast radius
- §2 The composite principle — seam + deterministic substrate + invariant
- §3 Verifying correctness (the oracle problem)
- §4 Testing failure handling (the hard part)
- §5 Fast & reproducible (protect the inner loop)
- §6 Preventing regression
- §7 Do NOT cargo-cult (anti-pattern ledger)
- §8 Cross-skill links
- §9 Provenance & confidence

---

## §1 The master dial: calibrate rigor to blast radius

**Before recommending any technique below, set the dial.** The single most common failure mode in testing advice is applying a regime built for one blast radius to a system with another. The level of testing investment should scale with **consequence-of-failure × patch latency**:

- *High blast radius, high patch latency* (firmware, an embedded DB on billions of un-updatable devices, a crypto primitive, a billing core) → exhaustive verification, full fault injection, the strictest coverage standard the domain warrants.
- *Low blast radius, low patch latency* (a stateless request handler you can redeploy in minutes) → one good harness, targeted error tests, branch or even statement coverage. More is waste.

SQLite earns its ~590:1 test-to-code ratio and 100% MC/DC standard **because** it is ubiquitous and effectively un-patchable in the field. FoundationDB earns its ~trillion-CPU-hour simulation regime because correctness-under-failure at distributed scale *is* its product. Neither number is a target to copy — they are the *output* of the dial set to an extreme. **Extract the calibration, never the setting.** (Provenance: `_extract-sqlite.md` §4, `_extract-foundationdb.md` §4.)

A corollary the corpus states explicitly: **a coverage number is not a correctness score.** SQLite documents that 100% MC/DC and fuzz-robustness are in direct tension (MC/DC discourages the defensive code that makes fuzzers fail). Know what each metric buys and expect metrics to conflict.

---

## §2 The composite principle: seam + deterministic substrate + invariant assertion

The corpus's most important synthesis — the three practices that keep recurring *together*, at different scopes:

1. **An injection seam** — a place in the production interface where a real resource (allocator, filesystem, clock, network) can be swapped for an instrumented one.
2. **A deterministic substrate** — all nondeterminism (randomness, time, scheduling) routed through seeded/controllable mechanisms, so a run replays exactly.
3. **An invariant assertion** — an executable check, run after the perturbation, that says "this must still hold" (no corruption, transaction atomic, etc.).

Compose them and you get fault injection that is both *thorough* (the seam reaches the failure) and *diagnostic* (determinism makes the failure reproducible, the invariant localizes it). Without the substrate, injection is flaky; without the invariant, it's silent.

- **SQLite** implements this at **component scope**: pluggable malloc/VFS (seam) + seeded fail-at-iteration-N loops (substrate) + `PRAGMA integrity_check` / transaction-atomicity (invariant).
- **FoundationDB** implements it at **system scope**: everything on Flow so all I/O/time/network is swappable (seam) + `deterministicRandom()` + virtual clock (substrate) + in-sim validation (invariant).

Status: the composite's core has **graduated** across SQLite + FoundationDB + Kubernetes — the injection **seam → P2** and **fault-injection-plus-invariant → P3**. The **deterministic-substrate** leg remains a **2/3 refinement** (Kubernetes is the deliberate counter-point: it injects faults on real, non-deterministic clusters and pays the flaky-e2e price — see "How the composite decomposed" in `_principles-index.md`). When advising, recommend the *composite*, sized to the dial — most teams need it around one critical module, with off-the-shelf parts (seeded RNG, fake clock, in-memory fakes), not a bespoke substrate.

---

## §3 Verifying correctness (the oracle problem)

*How do you know the answer is actually right?* Unit tests only encode what the author thinks is correct.

**3.1 Independent redundant verification (C5 · graduated → P1).** Verify with multiple harnesses *developed separately*, so they fail on disjoint blind spots — independence, not volume, is the active ingredient.
- *Apply when:* failure is costly and you can afford a second verification path for at least the critical kernel.
- *Dial down when:* low blast radius — one good harness plus CI is the rational stop. A second *independent* harness is for the crypto/billing/consensus core, not the whole app.
- *Exemplars:* SQLite (four harnesses + multiple independent fuzzers); FoundationDB (simulation + perf + hardware-failure regimes).

**3.2 Differential testing against an oracle (C6 · likely ≥3, confirm).** For behavior with an external spec, run identical inputs through *other mature implementations* and assert equal results.
- *Apply when:* a reference implementation of the same semantics exists (SQL, codecs, protocols, compilers).
- *Skip when:* you are the only implementation of novel logic → fall back to property/invariant assertions instead.
- *Exemplar:* SQLite's SQL Logic Test cross-checks against PostgreSQL/MySQL/SQL Server/Oracle.

**3.3 Executable invariants / assertions (C8 · graduated as P3's invariant leg; standalone 2/3).** Encode every "must be true here" as an assertion run during testing and **compiled out in production** (SQLite: 6,563 `assert()`); fuzzers most often trip these, i.e. the asserts were catching the latent bugs. FoundationDB adds *probabilistic expensive* invariant checks (`EXPENSIVE_VALIDATION`, ~5% under buggify).
- *Hard rule:* assertions must be **side-effect-free** and **disabled in prod** — an assert with a side effect breaks release builds; one left enabled turns a recoverable wrong-state into a hard crash (availability regression).
- *Boundary:* asserts check *programmer* invariants, not *user input* — input validation is real runtime code, never an assert.

**3.4 Mutation testing — test the tests (C13 · 1/3 provisional).** Mutate the source, confirm a test now fails; guards against vacuous tests that execute code (satisfying coverage) without asserting on behavior. Coverage proves a line *ran*, not that a wrong result would be *caught*.
- *Apply when:* test *quality* must be assured on a critical path. *Skip when:* expensive and the path is low-stakes and already covered by differential/property tests.

---

## §4 Testing failure handling (the hard part)

Error/failure paths are the least-exercised, most-bug-prone code. Building something that works on good input and a healthy machine is easy; surviving failure is the discipline.

**4.1 Systematic deterministic fault injection (C7 · graduated → P3).** Don't hope failures are handled — *simulate them deterministically and exhaustively*. The canonical shape: instrument the resource to fail after N operations, run in a loop advancing N until completion, run twice (fail-once and fail-continuously), and assert an invariant after each. Scope ranges from one syscall (SQLite: OOM/IO/crash) to a whole cluster (FoundationDB: machine/network/datacenter failure).
- *Prerequisite:* the injection seam (§2, C10) must exist; retrofitting into code that assumes infallible malloc/IO is the real cost.
- *Dial down when:* a failure just returns 500 and retries — a few targeted error tests suffice; the exhaustive loop is for commit/recovery paths where partial failure corrupts state.

**4.2 Compiled-out in-code failpoints (C16 · 1/3).** Let developers mark hazardous edges *inline* — `if (BUGGIFY) { inject delay / flush early / smaller buffer / rare branch }` — activated only under test, deterministically, per call-site. Reaches intra-component edge states that external injection can't, placed where the author knows the danger is.
- *Hard rule:* must compile/disable to a no-op in production, and needs a deterministic substrate or it just adds flakiness.
- *Transferable form without full simulation:* a test-only chaos hook behind a build flag (cf. failpoints in TiKV/etcd — would corroborate this practice).

**4.3 Adversarial fault design (C17 · likely ≥3).** Uniform-random injection under-samples pathological *orderings*; design structured adversarial sequences deliberately. FoundationDB's "swizzle-clog" (stop a random subset's connections one-by-one, restore in a different random order) finds deep bugs occurring only in the rarest real orderings; Jepsen-style nemeses are the external analogue.
- *Apply when:* large failure-interaction surface (consensus, recovery, replication). *Skip when:* simple stateless service.

**4.4 Same code in test and production (C15 · likely ≥3).** Test the *real* code paths, not a model that drifts. Achieved by writing production code against abstractions that have both real and simulated implementations (Flow / VFS), and reusing the same workload code across simulation and perf testing.
- *Boundary:* requires that dual-implementation substrate; where it's absent, a model may be unavoidable — but its fidelity becomes a standing risk to manage.

---

## §5 Fast & reproducible (protect the inner loop)

**5.1 Determinism: reproducibility from a seed (C14 · 2/3).** Control every source of nondeterminism (randomness via one seeded RNG, time virtual, scheduling cooperative) so any failure replays byte-identically. *Reproducibility is the diagnostic tool*: hit a failure, re-run the seed with added logging, get identical behavior.
- *Calibration:* FoundationDB architects the *whole system* for determinism (bespoke actor language) — entry price justified only when correctness-under-failure is the product. **Most teams should buy the reproducibility benefit cheaply**: seeded RNG + fake clock + in-memory fakes around one module. Do not rewrite everything for determinism.

**5.2 Virtual clock / time compression (C18 · narrowly transferable).** With time injected, collapse idle waits (FoundationDB ~10:1 real-to-simulated) and eliminate timing flakiness. Even without full simulation, any suite using a fake clock instead of real `sleep()` gains speed + determinism.

**5.3 Tiered test depth (C12 · ≥3 trivially).** Stratify by how fast a decision needs feedback: a minutes-long subset that catches *most* errors pre-commit (SQLite's `veryquick`), the full multi-config suite pre-release, soak before ship. *Skip the machinery* until the full suite is actually slow enough to hurt the inner loop.

**5.4 Design-for-testability seams (C10 · graduated → P2).** The first leg of §2's composite, and the enabler of §4.1. Build injection points into the production interface (pluggable allocator/filesystem/clock/network). Note SQLite's seams (VFS) are the *same* ones that give portability — testability and portability share a mechanism.
- *Boundary:* seams add indirection and public surface; apply at boundaries where failure-injection or portability matters, not everywhere ("inject everything" is the DI-everywhere anti-pattern). *(Cross-link: a seam viewed from the extension side is an extension point — meets the extensibility dimension, Envoy.)*

---

## §6 Preventing regression

**6.1 The regression ratchet (C11 · 1/3, SQLite-only so far).** No bug is fixed until a test that would have caught it is added, and that test lives forever; institutionalized for fuzzer finds via a persistent "interesting-cases" corpus rerun on every build. Effect: prevented failures grow monotonically.
- *Cost:* corpus growth — curate to behavior-distinct cases, not every input. Near-universally worth it; low constraint. (Expected to corroborate across Envoy/Redis/Kubernetes — check to lift past 1/3.)

---

## §7 Do NOT cargo-cult (anti-pattern ledger)

The boundaries are the product. Recognize and refuse these:

- **MC/DC (or any max-coverage standard) everywhere** — justified by SQLite's un-patchable ubiquity, not a universal default. Match coverage target to the dial (§1).
- **Matching SQLite's 590:1 ratio or FoundationDB's trillion-CPU-hours** — outputs of an extreme dial, not goals.
- **Treating a coverage number as a correctness score** — metrics conflict (MC/DC vs fuzzing); none captures "correct."
- **Building a bespoke deterministic substrate (à la Flow)** for an ordinary app — catastrophic over-engineering. Get reproducibility from off-the-shelf seeded RNG + fake clock at module scope.
- **In-code failpoints without a compiled-out guard** — injects real flakiness/risk into production.
- **"Inject everything" / DI-everywhere** — seams have a cost; place them at failure/portability boundaries only.

---

## §8 Cross-skill links

Keep the seam with `software-architect` clean:
- **Injection seams (§5.4) ↔ extensibility dimension** (Envoy): the same construct seen from the test side vs. the plugin side.
- **API honesty** (Redis C4 — "don't fake capabilities you can't deliver") **rhymes with** `software-architect`'s consistency-honesty ("effectively-once via idempotence, not exactly-once"): same value, different altitude. The *craft* instance (honest module APIs) lives in `implementation-simplicity.md` §4.1; the *architecture* instance (consistency-model honesty) routes to `software-architect`.
- If a "testing" question is really about *where the service boundaries should be* so they're testable, that's an architecture question — hand off.

---

## §9 Provenance & confidence

- Full evidence, calibrated verbs (Verified/Documented/…), and per-practice counter-cases: `_extract-sqlite.md` (C5–C13), `_extract-foundationdb.md` (C14–C18).
- This dimension was distilled from the two testing exemplars (SQLite + FoundationDB); **Kubernetes later corroborated as the third witness, graduating §2/§3.1/§4.1/§5.4 into `_principles-index.md` as P1–P4** (`_extract-kubernetes.md`). The deterministic-substrate refinement (§5.1, C14) deliberately stays at 2/3 — Kubernetes is its counter-point.
- Confidence tags here are honest snapshots, not guarantees. A practice at *1/3* is "one strong exemplar"; treat its universality as provisional until corroborated.
