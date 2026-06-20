# Extraction: FoundationDB — Deterministic Simulation Testing

Filled from `_extraction-template.md` via Distill mode. Sources: Tier 1 — `documentation/sphinx/source/testing.rst` (the repo's human-authored "Simulation and Testing" doc); Tier 2 — `flow/include/flow/Buggify.h`; Tier 3 — `fdbrpc/sim2.cpp`, `flow/include/flow/DeterministicRandom.h`, `flow/Net2.cpp`, `fdbserver/workloads/*`. Verified against a surgical blobless+sparse re-clone of `github.com/apple/foundationdb` (Apache-2.0) at commit `9bb7086`. Clone-measured counts drift across snapshots (this run sees 159 workload `.cpp` files; an earlier snapshot recorded ~167) and the doc's headline figures (≈1 trillion CPU-hours, ≈10:1 time) are the project's own estimates — treat all as *Documented as of a version*, not fixed constants.

This is the corpus's **second** testing exemplar. Its candidates are C14–C18; its other job (§3a) is to be the *second independent witness* that lifts several SQLite-originated practices to 2/3. Those reached 3/3 and graduated only later, when Kubernetes became the third witness — this file preserves the second-witness evidence; current graduation status lives in `_principles-index.md`.

---

## 0. Identity

- **Repo:** FoundationDB (`apple/foundationdb`)
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `9bb7086`; thesis is testing *discipline*, not decision-evolution, so no full history needed.
- **Language(s):** C++ written in **Flow** — a bespoke actor-based dialect compiled to C++ by the in-repo `flow/actorcompiler/`. The custom language is itself load-bearing for the thesis (see C14).
- **Why it's in the corpus:** The reference example of *deterministic simulation testing*; the second independent witness for the seam / fault-injection / same-code principles (corroborating SQLite); and the corpus's strongest deterministic substrate — which Kubernetes later reveals to be a *calibrated refinement*, not a universal.

## 1. Thesis

**Correctness under failure at distributed scale is achieved by running the *entire system* as a deterministic simulation — one seeded, single-threaded substrate in which every machine, disk, and network link is modeled and can be failed on demand, so any bug replays exactly from its seed.** It is the same shape SQLite uses (seams + injected faults + invariants) lifted from one component to a whole cluster, with **determinism** as the active ingredient that makes whole-system fault injection *diagnosable* instead of flaky.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `documentation/sphinx/source/testing.rst` ("Simulation and Testing") — the canonical methodology doc. States: deterministic single-threaded whole-cluster simulation with "perfect repeatability"; ≈1 trillion CPU-hours run; ≈10:1 real-to-simulated time; network/machine/datacenter fault injection; the "swizzle-clog" nemesis; a combined regime of simulation + live performance (Circus) + hardware-failure testing; Circus perf written in the *same code* as simulation workloads.

**Tier 2 — Design notes / mechanism:**
- `flow/include/flow/Buggify.h` — the `BUGGIFY` in-code failpoint mechanism and `EXPENSIVE_VALIDATION`, with the per-call-site activation logic in source.
- Flow itself (referenced by testing.rst; compiler at `flow/actorcompiler/`) — the actor substrate that makes all I/O/time/network swappable.

**Tier 3 — Code (confirm/falsify):**
- `flow/include/flow/DeterministicRandom.h` + `flow/DeterministicRandom.cpp` — seeded `mt19937_64` RNG behind the `IRandom` interface.
- `fdbrpc/sim2.cpp` — the `Sim2` engine: virtual clock (`now()` returns simulated `time`), network clogging primitives.
- `flow/Net2.cpp` — the *production* network impl, the real twin of `Sim2` behind one interface.
- `fdbserver/workloads/*.cpp` — 159 workloads incl. `Cycle.cpp` (ring-integrity invariant), `MachineAttrition.cpp`, `RandomClogging.cpp`, `ClogTlog.cpp`.

## 3. Candidate practices

### C14. Deterministic substrate — one seeded source of nondeterminism, so every run replays exactly

- Practice: Route *every* source of nondeterminism — randomness, time, scheduling, I/O ordering — through a single seeded, controllable substrate, so a whole run is a pure function of its seed and replays byte-identically. FDB does this at **system** scope: one seeded `DeterministicRandom` for all randomness, a virtual clock, cooperative single-threaded scheduling, and all I/O/network through Flow.
- Evidence: **Verified** — `flow/include/flow/DeterministicRandom.h` (`DeterministicRandom(uint64_t seed)`; `boost::random::mt19937_64` chosen, per its comment, "to get consistent output across different compilers and therefore across different C++ standard library implementations"; `resetSeed`); `fdbrpc/sim2.cpp` (`class Sim2 final : public ISimulator, public INetworkConnections`; `double now() const override { return time; }`). **Documented** — testing.rst: "a *deterministic* simulation of an entire FoundationDB cluster within a single-threaded process. Determinism is crucial in that it allows perfect repeatability of a simulated run, facilitating controlled experiments to home in on issues."
- Why / constraint: Reproducibility *is* the diagnostic tool — hit a failure, re-run the seed with extra logging, get identical behaviour. Whole-system fault injection without determinism yields flaky, un-repeatable failures.
- Counter-case: The entry price is architecting the whole system for determinism (FDB built a custom actor language to get it) — justified only when correctness-under-failure is the product. Most systems should buy the *benefit* cheaply (seeded RNG + fake clock + in-memory fakes around one module), not the substrate. **Kubernetes is the deliberate counter-point** — it injects faults on real, non-deterministic clusters and accepts flaky e2e.
- Convergence: SQLite does the same at *component* scope (seeded fail-at-N loops). **2/3** (SQLite + FDB). A *third deterministic-simulation* repo (TigerBeetle, Antithesis-tested) would graduate it; Kubernetes (the third testing witness) does **not** corroborate — it is the counter-example — so determinism stays a calibrated *refinement* even at three repos (see `_principles-index.md` P3).
- Graduates to: `testing-correctness.md` (as the determinism refinement of the fault-injection principle).

### C15. Same code in test and production — the test drives the real binary, not a model

- Practice: Write production code against abstractions that have both a real and a simulated implementation, so simulation exercises the *actual* production code paths rather than a reimplementation. The performance suite is written in the *same* workload code as the correctness suite.
- Evidence: **Documented** — testing.rst: "Flow … In addition to generating efficient production code, Flow works with Simulation for simulated execution"; and (Circus) "Workloads are specified using the same code as for simulation, which benefits the productivity of our testing." **Verified** — `fdbrpc/sim2.cpp` (`Sim2 : INetworkConnections`) is twinned with `flow/Net2.cpp` (the production network); both satisfy the one `INetwork`/`INetworkConnections` surface routed through `g_network`.
- Why / constraint: A model or mock encodes your *belief* about behaviour and drifts from it; running the real code under instrumentation closes that gap. Sharing workload code across sim and perf means both exercise identical logic.
- Counter-case: Requires the dual-implementation substrate (the seam, below). Where it's absent a model may be unavoidable — but then the model's fidelity to production becomes a standing risk you must actively manage.
- Convergence: SQLite witnesses it (the *real* library runs under instrumented malloc/VFS, not a reimplementation). **2/3** (SQLite + FDB) at this point; Kubernetes later made it 3/3 (envtest runs the *real* apiserver/etcd) → graduated to `_principles-index.md` **P4**.
- Graduates to: `testing-correctness.md` → `_principles-index.md` (P4).

### C16. Compiled-out in-code failpoints (BUGGIFY) — mark hazardous edges inline, fire them only under test

- Practice: Let developers annotate dangerous code edges *inline* — `if (BUGGIFY) { inject a delay / flush early / use a smaller buffer / take a rare branch }`. Each site is enabled only when buggify is on (under simulation, never in production), tracked per call-site, and fired probabilistically through the deterministic RNG so it stays reproducible. Reaches intra-component edge states external fault injection can't, placed where the author knows the danger is.
- Evidence: **Verified** — `flow/include/flow/Buggify.h`: `buggify(prob)` = `isGeneralBuggifyEnabled() && getGeneralSBVar(file,line) && deterministicRandom()->random01() < probability` (gated off in prod, per-`{file,line}` section, deterministic fire; section-activation probability `0.25`); companion `#define EXPENSIVE_VALIDATION (isGeneralBuggifyEnabled() && deterministicRandom()->random01() < P_EXPENSIVE_VALIDATION)` with `P_EXPENSIVE_VALIDATION{ 0.05 }` runs costly invariant checks ≈5% of the time, only under buggify.
- Why / constraint: Some failure-relevant states (a buffer flushed at an awkward point, a rare branch) live *inside* a component and can't be reached by failing an external resource. An inline, author-placed hook reaches them; the disabled-in-prod gate keeps the hazard out of production; deterministic firing keeps it replayable.
- Counter-case: Must compile/disable to a no-op in production, and needs a deterministic substrate or it just adds flakiness. Transferable without full simulation as a test-only chaos hook behind a build flag (cf. failpoints in TiKV / etcd).
- Convergence: **1/3** (FDB only). TiKV / etcd failpoints would corroborate.
- Graduates to: `testing-correctness.md`.

### C17. Adversarial fault design — engineer pathological orderings, don't just sample uniformly

- Practice: Uniform-random injection under-samples the rare *orderings* where the deepest bugs live; deliberately design structured adversarial sequences. FDB's "swizzle-clog": pick a random subset of nodes, clog (stop) each one's connections one-by-one over a few seconds, then unclog them in a *different* random order, one-by-one.
- Evidence: **Documented** — testing.rst: "the reigning champion is called 'swizzle-clogging' … This pattern seems to be particularly good at finding deep issues that only happen in the rarest real-world cases." **Verified** — `fdbrpc/sim2.cpp` connection-clog primitives (`clogPairFor`, `unclogPair`, `clogSendFor`, `clogPairUntil`); workloads `fdbserver/workloads/RandomClogging.cpp`, `MachineAttrition.cpp`, `ClogTlog.cpp`.
- Why / constraint: The interaction space of distributed failures is too large for uniform random to reach its pathological corners; hand-designed nemeses target the orderings most likely to break invariants.
- Counter-case: Worth it for a large failure-interaction surface (consensus, recovery, replication); a stateless service has no such ordering space and uniform error tests suffice. Jepsen-style nemeses are the external analogue for systems you can't simulate.
- Convergence: **1/3** (FDB). A Jepsen-tested or chaos-engineering repo would corroborate.
- Graduates to: `testing-correctness.md`.

### C18. Virtual clock / time compression — inject time, then collapse idle waits

- Practice: With time injected (the simulator owns `now()`), advance simulated time directly across idle waits instead of sleeping in real time, and eliminate timing flakiness because every component reads the same controlled clock. FDB runs ≈10:1 real-to-simulated time.
- Evidence: **Documented** — testing.rst: "steps through time, synchronized across the system, representing a larger amount of real time in a smaller amount of simulated time … about a 10-1 factor of real-to-simulated time." **Verified** — `fdbrpc/sim2.cpp`: `double now() const override { return time; }` (simulated time is a member advanced by the engine, not the wall clock).
- Why / constraint: Real `sleep()`-based waits make suites both slow and timing-flaky; a virtual clock removes both at once and is a prerequisite for deterministic replay of timing-dependent bugs.
- Counter-case: Full time-compression needs the simulation substrate, but the *kernel* transfers cheaply: any suite that injects a fake clock instead of calling real `sleep()` gains speed + determinism for timing-dependent logic.
- Convergence: **1/3** standalone; it is really the *time leg* of the C14 deterministic substrate, with which it folds.
- Graduates to: `testing-correctness.md` (as part of the determinism refinement).

## 3a. Convergence cross-check — FoundationDB as the second witness

FoundationDB is the corpus's **second** independent testing exemplar. With two unrelated repos (SQLite + FDB) several practices reach **2/3** — *not yet graduated*; each needs a third independent witness. (That witness later arrived as Kubernetes — see `_extract-kubernetes.md` §3a and `_principles-index.md`. This table preserves the second-witness state these graduations rest on.)

| Principle | SQLite | FoundationDB | Status at two witnesses |
|---|---|---|---|
| **Independent redundant verification** (C5) | 4 harnesses + independent fuzzers | simulation + Circus performance + hardware-failure regimes (testing.rst) | **2/3** — pending 3rd (→ later P1) |
| **Design for testability / injection seams** (C10) | pluggable malloc + VFS | whole system on Flow; `INetwork` = `Sim2` ∥ `Net2`; `IRandom` = `DeterministicRandom` | **2/3** — pending 3rd (→ later P2) |
| **Fault injection + invariant assertion** (C7 + C8) | OOM/IO/crash loops + `integrity_check` | network/machine/DC faults in sim + `Cycle.cpp` ring-integrity invariant + `EXPENSIVE_VALIDATION` | **2/3** — pending 3rd (→ later P3) |
| **Same code in test and production** (C15) | real library under instrumented malloc | Flow emits prod code *and* drives sim; Circus reuses workload code | **2/3** — pending 3rd (→ later P4) |
| **Executable invariants compiled out** (C8 standalone) | 6,563 `assert()` | `EXPENSIVE_VALIDATION` (≈5% under buggify) + in-sim validation | **2/3** |
| **Deterministic substrate** (C14) | seeded fail-at-N loops | full deterministic single-threaded simulation | **2/3** — and likely to *stay a refinement* (the next robustness witness, Kubernetes, is non-deterministic by design) |

**The nuance to carry forward.** The *base* principle — seam + injected fault + invariant — is on track to graduate (and did, at 3/3). Its *determinism* leg (C14/C18) is the costly part: FDB pays for it with a bespoke actor language *because reproducibility-under-failure is its product*. Whether determinism itself graduates hinges on finding a *third deterministic* system; a non-deterministic third witness (as Kubernetes turned out to be) corroborates the base principle while leaving determinism a calibrated refinement. This is the governing dial (rigor ∝ blast radius) applied to a single technique.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            Building a bespoke deterministic substrate — a custom actor language (Flow)
                   to run the whole system single-threaded and seeded.
- Why not general: Catastrophic over-engineering for an ordinary application. FDB pays this
                   entry price because correctness-under-failure at distributed scale IS its
                   product and a field bug in a distributed database is ruinous. EXTRACT THE
                   BENEFIT (reproducibility from a seed) CHEAPLY — seeded RNG + fake clock +
                   in-memory fakes around one module — NOT the substrate.
```

```
- Item:            ≈1 trillion CPU-hours of simulation; tens of thousands of simulations nightly.
- Why not general: An output of an extreme dial (FDB's blast radius), not a target to hit. The
                   transferable idea is "run many seeded randomized simulations and let volume ×
                   failure-intensity surface rare bugs," sized to your own consequence-of-failure.
```

```
- Item:            Whole-cluster modeling inside one process (drive performance, drive-fill,
                   per-packet network delivery, datacenter topology).
- Why not general: Worth it to test a distributed transactional store; absurd for a stateless
                   service. Model only the failure surface that can actually corrupt your
                   invariants — not the whole world.
```

## 5. Open questions / corpus cross-checks

- **Determinism (C14):** needs a *third deterministic* witness (TigerBeetle, an Antithesis-tested system) to graduate from refinement to principle. Kubernetes — already added as the third testing/robustness witness — is the non-deterministic counter-point, so it corroborates the *base* fault-injection principle (C7) but leaves determinism a calibrated refinement.
- **BUGGIFY (C16):** confirm against TiKV / etcd failpoints to lift past 1/3.
- **Swizzle-clog (C17):** confirm against a Jepsen-tested or chaos-engineering repo.
- **Figures drift:** 159 workload `.cpp` at commit `9bb7086` vs ~167 in an earlier snapshot; "trillion CPU-hours" and "10:1" are the doc's own estimates. Treat every such number as *Documented as of a version*, never a fixed constant — the same discipline the SQLite extraction notes for `testcase()` counts.
