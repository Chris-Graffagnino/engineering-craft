# Extraction: SQLite — Testing & Correctness Discipline

Filled from `_extraction-template.md` via Distill mode. Sources: shallow clone of the GitHub mirror `github.com/sqlite/sqlite` (Tier 3, code), in-source comments (Tier 2), and `sqlite.org/testing.html` + `sqlite.org/th3.html` (Tier 1, canonical philosophy; SQLite docs are public domain). Web-sourced figures are as of SQLite 3.42.0 (2023-05-16) unless noted; clone-measured figures are from the snapshot pulled during this run and differ slightly from the published doc due to version drift (noted where it matters).

---

## 0. Identity

- **Repo:** SQLite (GitHub mirror of the canonical Fossil repo on sqlite.org)
- **Commit / clone depth:** `--depth 1` shallow; thesis is testing *discipline*, not decision-evolution, so no full history needed.
- **Language(s):** C (core ~155.8 KSLOC per testing.html; ~207k raw lines across `src/*.c` in clone, incl. comments/blanks)
- **Why it's in the corpus:** The reference example of correctness engineering. Also supplies the corpus's most important *counter-case* (see §4).

## 1. Thesis

**Correctness is achieved by independent redundant verification plus systematic simulation of every failure mode — and the *level* of that investment is deliberately calibrated to blast radius.** SQLite teaches both the techniques *and* the discipline of knowing when they're worth it.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `sqlite.org/testing.html` ("How SQLite Is Tested") — the canonical methodology doc; its section structure is itself a fault taxonomy.
- `sqlite.org/th3.html` — the proprietary harness + the committed 100% branch/MC/DC standard since 3.6.17 (2009-08-10).
- In-repo: `doc/testrunner.md`, `doc/tcl-extension-testing.md` (harness mechanics).

**Tier 2 — Design notes (in-source comments):**
- `src/sqliteInt.h` — intent comments for `testcase()`, `ALWAYS()`/`NEVER()`, `TESTONLY()`, `VVA_ONLY()`. These explain the *why* of the coverage/defensive macros precisely.

**Tier 3 — Code (confirm/falsify):**
- `src/*.c` macro census (clone): `assert(` ×6,563 · `testcase(` ×917 · `ALWAYS(` ×194 · `NEVER(` ×140 · `VVA_ONLY(` ×30.
- `test/` — 1,283 files. Fault-injection suites by filename convention: io/crash/jrnl/wal ×71, fault ×52, malloc/oom ×33, corrupt ×33, fuzz ×31.

## 3. Candidate practices

### C5. Independent redundant verification — independence is the active ingredient, not volume

- Practice: Verify correctness with **multiple harnesses developed and maintained separately**, so they fail on disjoint bug classes. SQLite runs four: TCL (in-tree, public-domain, primary dev tests, 51,445 cases), TH3 (proprietary, embedded-targeted, 100% branch+MC/DC), SQL Logic Test (differential — see C6), dbsqlfuzz (fuzzing — see C9). It even keeps *multiple independent fuzzers* on purpose, noting that independently-developed testers surface obscure issues others miss.
- Evidence: **Documented** — testing.html §2 (four independently developed harnesses); §4.1.3 (independent fuzzers deliberately kept).
- Why / constraint: A single harness shares the blind spots of its authors. Independence — not raw test count — is what converts effort into disjoint coverage. (Echoes Linus's law: many independent eyes.)
- Counter-case: Four harnesses is enormous fixed cost, justified only when failure is catastrophic and field-unpatchable. For most software, *one* good harness plus CI is the rational stopping point; a second independent harness is warranted only for a critical kernel of code (a crypto primitive, a billing core), not the whole app.
- Convergence: _stub_ — differential/independent verification recurs in compiler testing (Csmith vs GCC/Clang), and the corpus's FoundationDB (simulation) is a second instance of "don't trust one verification path." Plausibly ≥3.
- Graduates to: `testing-correctness.md`

### C6. Differential testing against an independent oracle — solve "how do we know the answer is right?"

- Practice: For behavior with an external spec, check correctness by running the same inputs through **other mature implementations and comparing outputs**. SLT runs 7.2M queries (1.12GB data) against PostgreSQL, MySQL, SQL Server, and Oracle, asserting identical answers.
- Evidence: **Documented** — testing.html §2 (SLT description).
- Why / constraint: The oracle problem — unit tests encode what the author *thinks* is correct; an independent implementation encodes what the *standard* requires. Disagreement localizes a real bug (in either system).
- Counter-case: Requires an independent reference implementation of the same semantics to exist. Perfect for standardized behavior (SQL, HTTP, a codec); impossible for novel/proprietary logic where you *are* the only implementation. There, fall back to property-based testing (assert invariants) rather than oracle comparison.
- Convergence: _stub_ — codecs, compilers, protocol stacks. Likely ≥3.
- Graduates to: `testing-correctness.md`

### C7. Systematic deterministic fault injection — simulate every external failure, in a loop, then assert an invariant

- Practice: Don't hope external failures are handled — **simulate them deterministically and exhaustively**. SQLite's method (identical shape for OOM, I/O error, crash): instrument the resource (a malloc that fails after N allocations; a VFS that errors after N ops or snapshots+damages the file to mimic power loss), run the operation in a loop advancing the failure point by one each iteration until it completes without hitting the fault, and run the loop twice — fail-once and fail-continuously. After each, assert an invariant: `PRAGMA integrity_check` must show no corruption, and a transaction must be either fully committed or fully rolled back (never partial).
- Evidence: **Documented** — testing.html §3.1 (OOM), §3.2 (I/O error + integrity_check), §3.3 (crash via VFS snapshot/damage, atomic-commit invariant), §3.4 (compound failures: I/O error *during* crash recovery). **Verified** — fault-suite file census in clone (§2 above).
- Why / constraint: Error-handling paths are the least-exercised and most-bug-prone code; on embedded devices OOM and I/O failures are common, not theoretical. Deterministic injection makes these paths first-class and repeatable rather than relying on rare real failures.
- Counter-case: The technique *requires injection seams designed in* (see C10) — pluggable allocator, pluggable filesystem. Retrofitting seams into a codebase that assumes infallible malloc/IO is expensive. And the exhaustive fail-at-every-point loop is worth it for a storage engine's commit path; for a stateless request handler where a failure just returns 500, a few targeted error tests suffice.
- Convergence: _stub_ — **strong expected hit** with corpus FoundationDB (deterministic simulation of disk/network faults is its whole identity) and Kubernetes (failure-injection in reconcilers). ≥3 very likely. This is a leading candidate for the principles index.
- Graduates to: `testing-correctness.md`

### C8. Invariants as compiled-out assertions — `assert()` for "can't happen," zero production cost

- Practice: Encode every "this must be true here" invariant as an `assert()` (6,563 of them). They run during all testing and are compiled out in release (`NDEBUG`), so they cost nothing in production. For conditions that are *believed* unreachable but guarded defensively anyway, use `ALWAYS(X)`/`NEVER(X)`: in debug builds these assert if the impossible happens; the macro layer keeps the defensive intent explicit.
- Evidence: **Verified** — `src/sqliteInt.h` macro definitions + counts; **Documented** — testing.html §8.1 notes fuzzers most often trip `assert()`s (i.e. the asserts were catching the latent bugs).
- Why / constraint: Assertions document invariants executably and turn silent corruption into a loud, located failure during testing — at no runtime cost once disabled.
- Counter-case: **Assertions must be side-effect-free and disabled in production.** An `assert()` that mutates state breaks release builds; assertions left enabled in production convert a recoverable wrong-state into a hard crash (an availability regression). Also, asserts check *programmer* invariants, not *user input* — input validation must be real runtime code, never an assert.
- Convergence: _stub_ — near-universal in systems C/C++/Rust (`debug_assert!`). Almost certainly ≥3.
- Graduates to: `testing-correctness.md`

### C9. Profile-guided fuzzing with a persistent regression corpus

- Practice: Run coverage-guided fuzzers continuously (dbsqlfuzz on ~16 cores at all times, mutating **both** SQL and the database file simultaneously via a structure-aware mutator — hundreds of millions of mutations/day). Then **collect historical fuzzer-found cases into a corpus rerun on every `make test`** (`fuzzcheck`) so old bugs can never silently return.
- Evidence: **Documented** — testing.html §4.1.3 (dbsqlfuzz), §4.1.5 (fuzzcheck corpus). **Verified** — fuzz test files in clone.
- Why / constraint: Guided fuzzing discovers behaviors the designers never envisioned; the persistent corpus turns each discovery into a permanent guard (the ratchet, C11, applied to fuzzing).
- Counter-case: Continuous multi-core fuzzing is justified for code parsing untrusted input (SQL, file formats, network bytes). Code that never touches attacker-controlled input gets little from fuzzing and shouldn't pay for it.
- Convergence: _stub_ — OSS-Fuzz ecosystem broadly; pair with FoundationDB. ≥3 likely for "fuzz untrusted-input parsers."
- Graduates to: `testing-correctness.md`

### C10. Design for testability — build in injection seams

- Practice: The fault injection in C7 is only *possible* because the architecture exposes seams: a pluggable allocator (`sqlite3_config(SQLITE_CONFIG_MALLOC, …)`) and a pluggable Virtual File System (VFS). Testability is designed into the production interfaces, not bolted on.
- Evidence: **Documented** — testing.html §3.1–3.3 (instrumented malloc via config interface; alternative VFS for I/O and crash simulation).
- Why / constraint: You cannot exhaustively simulate a failure you have no seam to inject at. The seams that make SQLite portable (VFS) are the same ones that make it testable — testability and portability share a mechanism.
- Counter-case: Seams add indirection and a public surface to maintain. Worth it at the boundaries where failure injection or portability matters (allocator, filesystem, clock, network); over-applying "make everything injectable" is the dependency-injection-everywhere anti-pattern that adds accidental complexity. *(Cross-link: this is where the testing dimension meets Envoy's extensibility dimension — a seam is an extension point viewed from the test side.)*
- Convergence: _stub_ — clock injection, filesystem abstraction recur widely. ≥3 likely.
- Graduates to: `testing-correctness.md` (with cross-link to `extensibility-boundaries.md`)

### C11. The regression ratchet — every bug becomes a permanent test

- Practice: No bug is considered fixed until a test that would have caught it is added; that test lives forever. Applies to hand-found bugs and is institutionalized for fuzzer finds via the fuzzcheck corpus. Effect: the set of prevented failures grows monotonically.
- Evidence: **Documented** — testing.html §5 (Regression Testing) + §4.1.5 (fuzzcheck corpus rerun on every build).
- Why / constraint: Bugs cluster and recur; without a permanent guard, a fixed bug reopens under a later refactor. The ratchet converts every failure into permanent progress.
- Counter-case: Low-constraint — close to universally good. The only cost is corpus growth, managed by keeping only "interesting" (behavior-distinct) cases rather than every input ever tried.
- Convergence: _stub_ — expected in essentially every mature repo (Envoy, Redis, Kubernetes). **Strong principles-index candidate.**
- Graduates to: `testing-correctness.md`

### C12. Tiered test depth matched to feedback latency

- Practice: Stratify the suite by how fast a decision needs feedback. SQLite: `veryquick` (~304.7k cases, minutes) pre-checkin; full multi-harness run (hours, all platforms × compile configs) pre-release; soak (~248.5M tests) before release. Each gate runs the depth its decision warrants.
- Evidence: **Documented** — testing.html §2 (veryquick subset; full run before each release; TH3 soak figure).
- Why / constraint: A multi-hour suite on every commit destroys the inner loop; a minutes-long subset that catches *most* errors preserves it, with exhaustive depth reserved for the release gate where latency doesn't matter.
- Counter-case: Tiering only pays once the full suite is slow enough to hurt. A small project whose entire suite runs in seconds shouldn't add the machinery — premature tiering is wasted complexity.
- Convergence: _stub_ — CI fast/slow lanes are ubiquitous. ≥3 trivially.
- Graduates to: `testing-correctness.md`

### C13. Mutation testing — test that the tests can fail

- Practice: Deliberately mutate the source and confirm some test now fails. Guards against vacuous tests that execute code (satisfying coverage) without actually asserting on its behavior.
- Evidence: **Documented** — testing.html §7.6 (Mutation testing).
- Why / constraint: Coverage proves a line *ran*, not that a wrong result would be *caught*. Mutation testing measures the latter — the quality of the assertions, not just their reach.
- Counter-case: Expensive (recompile + full run per mutant). Reserve for code where test *quality* must be assured; not worth it for low-stakes paths already covered by differential or property tests.
- Convergence: _stub_ — less common; may not reach ≥3 in this corpus. Hold as repo-notable until corroborated.
- Graduates to: `testing-correctness.md` (provisional)

## 4. Rejected / over-engineered / do-NOT-generalize

**This is the most important section in the SQLite extraction.** SQLite's regime is the textbook example of a practice that must *not* be cargo-culted, and SQLite itself documents the limits.

```
- Item:            590:1 test-to-code ratio and a committed 100% MC/DC standard.
- Why not general: Justified by a constraint almost no other project shares — SQLite ships
                   on billions of devices, is embedded, and is effectively un-patchable in
                   the field, so a latent bug is catastrophic and irreversible. The coverage
                   *target* should scale with blast radius × patch latency. Most software has
                   smaller blast radius and can ship a fix in hours, making branch — or even
                   statement — coverage the rational target. Copying MC/DC and a 590:1 ratio
                   into a CRUD web app is pure waste. EXTRACT THE CALIBRATION PRINCIPLE
                   (match rigor to consequence), NOT THE SETTING (MC/DC everywhere).
```

```
- Item:            Maximizing a single coverage metric as a proxy for "correct."
- Why not general: SQLite documents (testing.html §4.1.6) that 100% MC/DC and fuzz-robustness
                   are in DIRECT TENSION: MC/DC discourages defensive code with unreachable
                   branches, but that defensive code is exactly what stops fuzzers reaching bad
                   states. So code at 100% MC/DC is MORE fuzz-vulnerable, and fuzz-hardened code
                   sits well below 100% MC/DC. MC/DC → robust in normal use; fuzzing → robust
                   under attack. Lesson: no single metric captures correctness; know what each
                   one does and does not buy, and expect them to conflict. A coverage number is
                   not a correctness score.
```

```
- Item:            TH3 as a proprietary, closed harness.
- Why not general: A funding/business-model choice (TH3 is sold to support the project and serve
                   regulated/aviation users), not a testing principle. The transferable idea is
                   "independent harness using only the public interface," not "make it closed."
```

## 5. Open questions / corpus cross-checks

- Run the convergence test on C7 (fault injection) and C11 (regression ratchet) against FoundationDB and Kubernetes next — both are the strongest expected corroborators and would promote these two to `_principles-index.md` first.
- Confirm whether C13 (mutation testing) reaches ≥3 across the corpus or stays a repo-notable.
- C10 (injection seams) deliberately bridges to the Envoy extensibility dimension — when Envoy is extracted, link the two from `_principles-index.md` rather than duplicating.
- Numbers drift across snapshots (the published `testcase()` count and the clone's 917 differ; the doc's own dbsqlfuzz rate is quoted as both ~500M/day and ~1B/day). Treat specific counts as *Documented as of a version*, never as fixed constants.

---

## 6. Supplementary extraction — Simplicity cross-witness (added after the Redis distillation)

*Added in a later pass to make SQLite the **second witness** for the implementation-simplicity dimension (Redis was the first). Sourced from a fresh surgical re-clone at commit `3a697a2` (the zero-deps tree structure, the amalgamation tooling, comment density) plus sqlite.org's public-domain docs. SQLite's testing-dimension candidates (C5–C13, above) are unchanged; this section only adds the simplicity candidates SQLite witnesses, lifting several of Redis's (C3, C4, C41–C45 in `_extract-redis.md`) toward 2/3.*

**Independence caveat (honest):** SQLite and Redis are both **single-author-led C systems projects** with a strong readability ethos — genuinely independent (different domains: embedded on-disk DB vs in-memory store; different authors and era) but *less* independent than a cross-language / cross-paradigm pair would be. The convergence is real, but the *third* witness should be something further afield (Go's stdlib, a non-C project) — see §6-bis.

### 6a. Convergence cross-check — SQLite as the 2nd simplicity witness

| Redis candidate | SQLite mechanism | Verdict |
|---|---|---|
| **C3** understandability budget / readable code | Heavily-commented source (**Verified:** `src/btree.c` = 11,600 lines, ~31% comment-*leading*, undercounting trailing/block comments); sqlite.org documents clarity as a goal ("~35% comments"). Clarity is *enabled by* the testing regime (§3–§5) — you can refactor freely because the tests catch regressions. | **1/3 → 2/3** (strong) |
| **C45** minimize the dependency surface | **Zero external dependencies** (**Verified:** no dependency manifests / vendored libs in the tree) shipped as a **single-file amalgamation** (`tool/mksqlite3c.tcl` builds one `sqlite3.c`); sqlite.org headlines "self-contained, zero-dependency." SQLite is the *more extreme* exemplar than Redis. | **1/3 → 2/3** (strong) |
| **C44** special-case compact representations | Compact on-disk record format — variable-length integers (varints) + type-tagged serial values, space-optimized for the common small value (sqlite.org file-format docs). | **1/3 → 2/3** (moderate; Documented) |
| **C42** simplest execution model | SQLite is an **in-process library, not a server** — it builds no concurrency machinery of its own; access is serialized, the engine runs in the caller's thread. A different *flavor* of Redis's single-threaded core (library-serialization vs single-threaded-server), same move: don't build concurrency infrastructure you don't need. | **1/3 → 2/3** (moderate; flavor differs) |
| **C43** avoid intermediate layers | **NOT witnessed — mild counter-case.** SQLite is *deliberately layered* (tokenizer → parser → code-gen → VDBE → B-tree → pager → VFS) because a SQL engine's domain *requires* it. This **confirms C43's own counter-case** ("some domains genuinely need the layer"). SQLite's simplicity is *clean boundaries between* layers, not layer-avoidance. | **stays 1/3** (counter-case) |
| **C4** API honesty | Weak/partial — SQLite is honest about its limits ("appropriate uses" / "when not to use" positioning), but that's product-positioning honesty, not the "don't fake a capability *in the API*" mechanism Redis C4 names. | **stays 1/3** |
| **C41** complexity budget / 95-5 | Already anchored to **P0**, which SQLite witnesses directly (it's one of the three P0 repos) — but as the *rigor* dial, not a distinct "95/5" statement. No new lift. | **unchanged** (P0-anchored) |

**Result:** C3 and C45 reach a clean **2/3** (Redis + SQLite); C44 and C42 reach a *moderate* 2/3; C43 stays 1/3 with SQLite as its **counter-case confirmer**; C4/C41 unchanged. Two witnesses < the ≥3 bar, so nothing graduates into `_principles-index.md`.

### 6-bis. Open questions (simplicity)

- A **third, more-independent** witness would graduate C3/C45: **Go** (gofmt + "less is more" ethos, stdlib self-containment) is the cleanest — non-C, multi-author, different paradigm, sidestepping the "single-author C project" caveat. A language or standard-library project (not another C database) is the right shape.
- C44/C42 are *moderate* corroborations — a storage/codec repo (C44) or another library-not-a-server (C42) would firm them.
- The SQLite **counter-case on C43** (clean layering for an inherently-layered domain) is worth foregrounding in `implementation-simplicity.md`: it sharpens "avoid layers" into "avoid *gratuitous* layers; when the domain demands layers, keep their boundaries clean."
