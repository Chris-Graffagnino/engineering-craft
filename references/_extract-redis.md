# Extraction: Redis — Implementation Simplicity & Understandability

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/redis/redis` (branch `unstable`) at commit `44a6a9f`. Tier 1 — `MANIFESTO` (Salvatore Sanfilippo / antirez), `CONTRIBUTING.md`. Tier 3 — `src/ae.c` ("A simple event-driven programming library"), `src/listpack.c`, `src/sds.h`, and the flat `src/` layout (125 `.c` files directly under `src/`). The MANIFESTO is the load-bearing source; the code confirms it.

**Seam discipline:** stays on the *craft* side of the boundary with `software-architect`. Extracts the **mechanics and discipline** of keeping an implementation simple. It does **not** decide the system's concurrency model or topology (single-node vs Cluster vs Proxy) — those are architecture and route to the other skill. Where the MANIFESTO touches an architectural choice (e.g. shared-nothing), this extract takes only the *craft lesson* ("don't add concurrency complexity speculatively") and names the seam.

This **founds the implementation-simplicity dimension** (the fourth and final) with Redis as its sole witness — so candidates are **1/3**. But simplicity is the corpus's *meta-value*: the dial **C41** is **P0** seen from the complexity side, and several candidates rhyme across dimensions (shared-nothing ↔ Erlang/OTP, minimal surface ↔ Linux C40, minimal deps ↔ Envoy C37) — see §3a. C3, C4 were seeded earlier in `_extraction-template.md`; this file formalizes them and adds C41–C45.

---

## 0. Identity

- **Repo:** Redis (`redis/redis`).
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `44a6a9f` (branch `unstable`). Thesis is simplicity *discipline*, not decision-evolution.
- **Language(s):** C (a deliberately flat, readable `src/`).
- **Why it's in the corpus:** The reference example of *implementation simplicity as a first-class, defended goal* — and the corpus's clearest statement that fighting complexity is the core design activity. Also the deliberate **counter-case** to Envoy's govern-a-large-extension-surface (Redis keeps a small core and resists surface instead).

## 1. Thesis

**Simplicity is the default and complexity must be *earned* by a real (not hypothetical) use case — because the payoff, a codebase one programmer can hold in their head, is itself a form of freedom (the opposite of lock-in).** Redis teaches three moves: fight complexity by *not creating it*; when an abstraction can't hold cleanly, *expose the trade-off* rather than fake it; and *optimize for the use cases you actually have*.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `MANIFESTO` — the project's stated values. Load-bearing points: #3 (avoid intermediate API layers so complexity is *obvious*), #5 (self-contained code; few deps), #6 (against complexity; understandability budget; complexity = lock-in), #7 (mostly single-threaded / shared-nothing; parallelize only the measured low-hanging fruit), #8 (don't fake capabilities you can't deliver; expose trade-offs), #10 (opportunistic programming — "solve 95% of the problem with 5% of the code").
- `CONTRIBUTING.md` — contribution process.

**Tier 3 — Code (confirm/falsify):**
- `src/ae.c` — the event loop, self-described "A simple event-driven programming library" (511 lines, reused from a Tcl interpreter) — the single-threaded core in practice.
- `src/listpack.c`, `src/sds.h` — compact special-cased representations (listpack fast-paths the common small entry; sds = "simple dynamic strings").
- Flat `src/` (125 top-level `.c` files) — the "read the whole source" claim made structural.

## 3. Candidate practices (implementation-simplicity dimension)

### C3. Hold an understandability budget; treat complexity as lock-in

- Practice: Make "one programmer can understand the whole system by reading the source for a bounded time" an explicit, defended goal. Reject features whose complexity outweighs their value; *the best way to fight complexity is to not create it at all*.
- Evidence: **Codified** — `MANIFESTO` #6 ("We're against complexity… we'll try hard to recognize when a small feature is not worth 1000s of lines of code. Most of the time the best way to fight complexity is by not creating it at all. Complexity is also a form of lock-in… remain understandable, enough for a single programmer to have a clear idea of how it works in detail just reading the source code for a couple of weeks"). **Verified** — flat `src/` (125 `.c` files), readable self-contained modules (`ae.c`).
- Why / constraint: Code only one author understands can't be safely modified by others *regardless of license*; comprehensibility is a hard maintainability constraint, not an aesthetic.
- Counter-case: A bounded "read it all in two weeks" budget doesn't scale to genuinely large-domain systems (a compiler, a kernel) where irreducible essential complexity exceeds any single-reader budget — there the unit of comprehensibility becomes the *module*, not the whole. The principle survives; the scope of "the whole" shrinks.
- Convergence: **1/3** (Redis). SQLite's single-file/single-reader clarity is the obvious corroborator (in-corpus but not extracted for this — its extraction is testing-focused); Go's simplicity ethos is another. Likely 2/3 with a formal SQLite-clarity note.
- Graduates to: `implementation-simplicity.md`

### C4. Don't fake capabilities you can't deliver in all cases — expose the trade-off

- Practice: Refuse an API that *appears* to work uniformly but silently fails outside a happy path. Where a clean abstraction can't hold (e.g. multi-key ops across a partitioned system), expose the seam and the trade-off to the user rather than papering it with magic.
- Evidence: **Codified** — `MANIFESTO` #8 ("there's no way to make the more complex multi-keys API distributed in an opaque way without violating our other principles. We don't want to provide the illusion of something that will work magically when actually it can't in all cases… expose the trade-offs to the user").
- Why / constraint: A leaky abstraction that hides its leaks produces correctness surprises at the worst time; honest seams are debuggable.
- Counter-case: In tension with ergonomics — for *generic/supporting* surfaces a slightly-leaky convenience layer may be the right call; the honesty rule earns its keep on *core* correctness-sensitive paths.
- Convergence: **1/3** (Redis). Strong **cross-skill rhyme**: `software-architect`'s consistency-honesty ("effectively-once via idempotence, not exactly-once") is the same value one altitude up — keep the *craft* instance (honest module APIs) here; route the architecture instance there.
- Graduates to: `implementation-simplicity.md` (cross-link noted in `_principles-index.md`)

### C41. The complexity budget — earn complexity with a real use case ("95% with 5%")

- Practice: Treat complexity as a budget spent only against *real, present* use cases, not hypothetical generality. "Opportunistic programming": get the most for the user with minimal complexity increase — solve 95% of the problem with 5% of the code when that's acceptable, and follow the flow of actual user requests rather than a fixed feature schedule.
- Evidence: **Codified** — `MANIFESTO` #10 ("opportunistic programming: trying to get the most for the user with minimal increases in complexity… Solve 95% of the problem with 5% of the code when it is acceptable") and #6 ("a small feature is not worth 1000s of lines of code").
- Why / constraint: Generality you don't yet need is complexity you pay for now and may never use; deferring it keeps the code small and lets later changes "reach a critical point" that makes the feature cheap.
- Counter-case: The 95% must genuinely be acceptable — for a *correctness*-critical path the missing 5% may be the catastrophic case (a billing rounding error, a security edge). 95/5 is for features where bounded incompleteness is tolerable, not for invariants. (This is where the simplicity dial meets P0's blast-radius dial — they're the same dial.)
- Convergence: **This is P0 seen from the complexity side** — "calibrate investment to real consequence" applied to complexity rather than to test rigor. Inherits P0's corpus-wide (3/3) grounding; the Redis-specific "95/5" framing is **1/3**.
- Graduates to: `implementation-simplicity.md` (anchored to `_principles-index.md` P0)

### C42. Prefer the simplest execution model your use cases allow; add concurrency only at measured low-hanging fruit

- Practice: Default to the simplest execution model that covers your real workload — for Redis, a (mostly) single-threaded, shared-nothing core — because it removes whole categories of concurrency complexity. Add parallelism *only* where measurement shows a bounded, high-value win (Redis: threaded I/O), never speculatively.
- Evidence: **Codified** — `MANIFESTO` #7 ("an efficient (mostly) single threaded Redis core… A shared nothing approach is not just much simpler… we may explore parallelism only for I/O, which is the low hanging fruit: minimal complexity could provide an improved single process experience"). **Verified** — `ae.c`, the small single-threaded event loop.
- Why / constraint: Shared-mutable-state concurrency is among the costliest complexity sources (races, locks, nondeterminism); avoiding it buys simplicity *and* predictability. When you must add it, targeting the measured low-hanging fruit caps the complexity.
- Counter-case: Bounded by the workload — a CPU-bound or massively-parallel problem cannot be served single-threaded; there the simpler model doesn't fit and the complexity is essential, not speculative. *(Seam: the **choice** of concurrency model / topology — single-node vs Cluster — is `software-architect`'s; the craft lesson "don't add concurrency complexity speculatively" is here.)*
- Convergence: **1/3** (Redis), with a strong **cross-dimension rhyme**: Erlang/OTP's share-nothing actor model (extracted as the C24 counter-case) chooses shared-nothing for the same simplicity/isolation reason. Note the rhyme; a formal second simplicity-framed witness would lift it to 2/3.
- Graduates to: `implementation-simplicity.md`

### C43. Avoid intermediate abstraction layers — make complexity obvious, compose from primitives

- Practice: Expose fundamental operations directly and let complex behavior be *the sum of basic operations*, rather than wrapping them in opaque high-level layers. Keep "just the layers of abstraction that are really needed," so the cost of an operation is visible to the caller.
- Evidence: **Codified** — `MANIFESTO` #3 ("Redis will avoid intermediate layers in API, so that the complexity is obvious and more complex operations can be performed as the sum of the basic operations") and #4 ("Faster code having just the layers of abstractions that are really needed").
- Why / constraint: Each abstraction layer hides cost and adds surface; a thin API over fundamental structures keeps time/space complexity legible and the implementation small. (Redis even makes complexity *part of the data-type contract* — see MANIFESTO #1.)
- Counter-case: Some domains genuinely need a higher-level abstraction to be usable at all (a query planner over raw storage); there an intermediate layer is essential, not accidental. The rule is "no *gratuitous* layer," not "never abstract."
- Convergence: **1/3** (Redis). Inverse rhyme with the extensibility dimension: where Envoy adds a *narrow extension layer* (C32), Redis *resists* layering and exposes primitives — two answers calibrated to different goals (open surface vs minimal core).
- Graduates to: `implementation-simplicity.md`

### C44. Special-case the common case with a compact, self-contained representation

- Practice: Don't reach for one general data structure; implement small, special-cased representations that fit the common (often small) case cheaply, and keep each one self-contained and readable. Switch representations by size/threshold when it pays.
- Evidence: **Verified** — `src/listpack.c` (compact encoding, "Fast path: single byte (most common for small entries <= 127 bytes)"), `src/sds.h` ("simple dynamic strings"). **Codified** — `MANIFESTO` #4 ("incrementally build more and more memory efficient data structures") and #1 (the data type *is* its space/time complexity).
- Why / constraint: A single general structure pays worst-case cost everywhere; small special-cased representations keep the common path cheap and the code each one needs small enough to understand in isolation.
- Counter-case: Proliferating special cases is itself complexity — each representation + the switching logic is code to maintain. Justified when the common case is hot and the representation is self-contained; *not* when you're speculatively optimizing a cold path (that violates C41).
- Convergence: **1/3** (Redis). Common in performance-critical systems; a second witness (a codec/DB storage layer) would corroborate.
- Graduates to: `implementation-simplicity.md`

### C45. Minimize the dependency surface — prefer self-contained code you understand

- Practice: Default to writing small, self-contained implementations you fully understand rather than pulling in external code; use external libraries only when they are genuinely self-contained and warranted ("beautiful self contained libraries when needed").
- Evidence: **Codified** — `MANIFESTO` #5 ("we'll be happy to use beautiful self contained libraries when needed. At the same time, when writing the Redis story we're trying to write smaller stories that will fit in to other code").
- Why / constraint: Every dependency is code you don't understand, a supply-chain/attack surface, and a constraint on your own evolution; for a small core, writing it yourself can be *simpler* than vendoring and tracking an external dep.
- Counter-case: Strongly bounded — "write it yourself" is correct for small, well-understood pieces, and *dangerous* for things you'd implement worse than a mature library (crypto, compression, TLS). The rule is "minimal, understood deps," not "no deps" (Redis itself vendors jemalloc, Lua).
- Convergence: **1/3** (Redis) for the *write-self-contained* framing. **Cross-dimension rhyme:** Envoy governs its dependency surface (C37) and Linux is largely self-contained — three different answers to "control the dependency surface." The shared value is real; the practices differ.
- Graduates to: `implementation-simplicity.md`

## 3a. Convergence / cross-corpus position

Implementation-simplicity is a **new dimension with Redis as its sole witness → every candidate is 1/3.** No graduations. But simplicity is the corpus's *meta-value*, so the dimension is densely connected:

| Candidate | Connects to | Relationship |
|---|---|---|
| **C41** complexity budget / 95-5 | **P0** — calibrate investment to real consequence | C41 *is* P0 seen from the complexity side; the simplicity dial and the rigor dial are one dial |
| **C42** shared-nothing simpler model | **Erlang/OTP** share-nothing (C24 counter-case) | both choose shared-nothing to avoid concurrency complexity — rhyme, reasons overlap (simplicity ∩ isolation) |
| **C45** minimal deps | **Envoy** C37 (govern deps) · **Linux** (self-contained) | three answers to "control the dependency surface"; shared value, different practices |
| **C3 / C41** "don't create complexity" | **Linux** C40 (prune unused surface) | "minimize the surface" — Redis by *prevention*, Linux by *removal*; same value across two dimensions → arguably **2/3 for the value** |
| **C4** API honesty | `software-architect` consistency-honesty | same honesty value one altitude up (cross-skill, not a corpus witness) |
| **C43** resist layering / expose primitives | **Envoy** C32 (add a narrow extension layer) | *inverse* — open surface vs minimal core, calibrated to opposite goals |

The headline: Redis is the deliberate **counter-pole to the extensibility dimension**. Envoy/Linux govern a *large* surface; Redis's answer to the same decay pressure is to *not have one*. Both are valid — calibrated to whether an open extension surface earns its keep. That pairing is the sharpest cross-dimension result and belongs in both references.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            "Solve 95% of the problem with 5% of the code."
- Why not general: A feature heuristic, NOT a correctness heuristic. For a correctness- or
                   security-critical path the missing 5% may BE the catastrophic case. 95/5 applies
                   where bounded incompleteness is tolerable; it must never be pointed at an
                   invariant. (This is exactly P0: calibrate to blast radius.)
```

```
- Item:            Mostly-single-threaded core.
- Why not general: A simplicity WIN given Redis's in-memory, per-op-fast workload and its shared-
                   nothing scaling story (Cluster). It is NOT a universal — a CPU-bound or
                   embarrassingly-parallel workload needs real parallelism. Extract "prefer the
                   simplest model your workload allows; parallelize the measured hot spot," not
                   "be single-threaded."
```

```
- Item:            "Write small self-contained stories" / few dependencies.
- Why not general: Right for small, well-understood pieces; DANGEROUS for things a mature library
                   does far better (crypto, TLS, compression) — Redis itself vendors jemalloc and
                   Lua. The principle is "minimal, understood deps," not "reinvent everything."
```

## 5. Open questions / corpus cross-checks

- Every candidate is **1/3** (Redis-only). The cleanest second witness for the *understandability* core (C3) is a **formal SQLite-clarity extraction** (in-corpus already; its testing extract didn't cover readability) — would lift C3 to 2/3 immediately. **Go** (gofmt/simplicity ethos) is another.
- **C42** (shared-nothing/simplest model) and the "minimize the surface" value (C3/C41 ↔ Linux C40) are the strongest cross-dimension rhymes — candidates for a future *cross-dimension* note in `_principles-index.md` once a second simplicity-framed witness exists.
- The **Redis-vs-Envoy counter-pole** (resist surface vs govern surface) should be cross-linked from *both* `implementation-simplicity.md` and `extensibility-boundaries.md` — it's the corpus's clearest "two valid opposite answers, calibrated by context."
- All four dimensions now have at least a founding extract; the remaining work is corroboration (second/third witnesses) and the triggering loop against the `software-architect` boundary — not new dimensions.
