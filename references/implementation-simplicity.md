# Implementation Simplicity — Dimension Reference

**Role:** consumed in **Apply mode**. Load this when a craft question touches *whether and how to keep an implementation simple* — should we add this feature/abstraction/dependency, is this design too clever, how do we keep code understandable, when is "good enough" actually good enough, should this be threaded, do we build it or pull in a library. Synthesize from it — do not quote it at the user.

**Sources:** distilled from **three** simplicity exemplars spanning app / library / language: **Redis** (its `MANIFESTO`, confirmed in a flat readable `src/`), **SQLite** (zero-dependency single-file amalgamation + heavily-commented source), and **Go** (a small whole-language spec, `gofmt`, the Go Proverbs — the cross-paradigm independence anchor). Per-practice evidence: `_extract-redis.md` (C3, C4, C41–C45), `_extract-sqlite.md` §6, `_extract-go.md` (C46 + the graduating cross-check).

**Confidence caveat — read it before applying anything here.** **Two practices have GRADUATED to 3/3** across Redis + SQLite + Go — the understandability budget (**C3 → P5**) and the minimal-dependency surface (**C45 → P6**) — the first principles to reach `_principles-index.md` from a non-testing dimension. Lead with those. **At 2/3:** avoid-gratuitous-layers (C43 — Redis + Go, *capped* because SQLite is the counter-case), API honesty (C4 — Redis + Go), simplest-model (C42, *two valid forms* — see below), compact-representations (C44, moderate). **At 1/3:** mechanical formatting (C46, Go-only). **Honesty on the graduations:** Redis and SQLite share a single-author-C-systems profile; **Go is the cross-paradigm anchor** that makes the trio credible (comparable texture to P1–P4's "2 DBs + 1 orchestrator"). The **dial** (§1) *is* **P0** from the complexity side, and the dimension is the deliberate **counter-pole** to extensibility (§6). See §7.

---

## Contents
- §1 The master dial — the complexity budget (earn complexity with a real use case)
- §2 The core pattern — the understandability budget; complexity is lock-in
- §3 Keeping the implementation simple
- §4 Keeping the interface honest
- §5 Do NOT cargo-cult (anti-pattern ledger)
- §6 Cross-skill & cross-dimension links
- §7 Provenance & confidence

---

## §1 The master dial: the complexity budget — earn complexity with a real use case

**Before adding anything, set the dial.** The defining failure mode of simplicity work is paying for complexity you don't yet need — generality, abstraction, threading, a dependency — against *hypothetical* rather than *present* use cases. Spend complexity only when a real use case earns it:

- *Earned* — a present, real workload needs it, and the simpler option genuinely can't serve it → add the complexity, targeted and bounded.
- *Unearned* — "we might need it," "it's more general," "it's more elegant" → don't. The generality is cost now and may never pay off.

Redis names this **opportunistic programming** (C41): "get the most for the user with minimal increases in complexity — **solve 95% of the problem with 5% of the code** when it is acceptable," and follow the flow of actual user requests, not a fixed feature schedule. **This is the corpus's governing dial — P0, calibrate investment to real consequence — pointed at complexity** instead of test rigor; they are *one dial*. The crucial calibration: **95/5 is a *feature* heuristic, never a *correctness* one** — on a correctness- or security-critical path the missing 5% may *be* the catastrophic case, so there the dial (P0/blast-radius) says spend. (Provenance: `_extract-redis.md` C41 → P0.)

---

## §2 The core pattern: the understandability budget — complexity is lock-in

The dimension's centerpiece (C3 · **3/3 — GRADUATED to P5**). Make **"one programmer can understand the whole system by reading the source for a bounded time" an explicit, defended goal** — and recognize that *the best way to fight complexity is to not create it at all*.

The load-bearing reframe: **complexity is a form of lock-in.** Code so complex that only its author understands it cannot be safely changed by anyone else — *regardless of the license*. Comprehensibility is therefore a hard maintainability constraint, not an aesthetic preference. Redis defends a concrete budget ("a single programmer… reading the source code for a couple of weeks") and backs it structurally (a flat `src/`, small self-contained modules like the 511-line `ae.c`).

- *Apply when:* the code will outlive its author's attention and be maintained by others — i.e. almost always.
- *Calibration:* the budget *scales* — "read the whole thing in two weeks" doesn't survive a compiler or a kernel, where irreducible essential complexity exceeds any single-reader budget. There the unit of comprehensibility becomes the **module**, not the whole system. The principle holds; the scope of "the whole" shrinks. (Cross-dimension: this rhymes with Linux C40 "prune the surface" — Redis minimizes by *prevention*, Linux by *removal*; same value, two dimensions.)
- *Evidence:* Redis — `MANIFESTO` #6 (verbatim above); flat `src/` (125 top-level `.c` files). (`_extract-redis.md` C3.)
- *2nd witness (SQLite):* heavily-commented source (`src/btree.c`: 11,600 lines, ~31% comment-leading); clarity is *bought by* the testing regime — you refactor freely because the tests catch regressions. Readability and testability reinforce each other. (`_extract-sqlite.md` §6.)
- *3rd witness (Go) → GRADUATES:* a ~42k-word *whole-language* spec, `gofmt` (one canonical format), the Proverb "Clear is better than clever," features omitted to stay small. Three unrelated repos (app / library / language) clear the ≥3 bar → **P5** in `_principles-index.md`. *Independence note: Redis + SQLite are both single-author C systems projects; Go (multi-author language) is the cross-paradigm anchor — see P5.* (`_extract-go.md`.)

---

## §3 Keeping the implementation simple

*Given a real need, how do you implement it without importing accidental complexity?*

**§3.1 Pay for only the concurrency complexity your workload needs (C42 · 2/3 — *two valid forms*).** Don't carry concurrency machinery you don't need. There are two settings of this dial, both valid: **avoid** the machinery (Redis/SQLite — single-threaded / in-process), or **make it cheap and safe** (Go — goroutines/channels). Pick by workload; either way, don't add concurrency complexity speculatively.
- *Apply when:* the workload fits a simpler model (I/O-bound, per-op-fast, shardable) — the simpler model buys both simplicity and predictability.
- *Skip when:* a CPU-bound or embarrassingly-parallel workload that genuinely cannot be served single-threaded — there the concurrency is *essential*, not speculative.
- *Calibration:* when you must add concurrency, target the *measured* low-hanging fruit (Redis added threaded I/O, not a threaded core). Never add concurrency speculatively.
- *Evidence:* Redis — `MANIFESTO` #7 ("an efficient (mostly) single threaded… shared nothing approach is not just much simpler… parallelism only for I/O, which is the low hanging fruit"); `ae.c`. (`_extract-redis.md` C42.)
- *Witnesses, two forms:* Redis (single-threaded core, parallelize only measured I/O) and SQLite (in-process library, no concurrency machinery of its own) *avoid* it; **Go is the counter-point** — it *embraces* concurrency but makes it cheap and safe (goroutines/channels), so the hard thing becomes simple rather than forbidden. Not in conflict — two settings of one dial. (`_extract-redis.md` C42, `_extract-sqlite.md` §6, `_extract-go.md` §3a.) *Seam: the **choice** of concurrency model/topology is `software-architect`'s; the craft lesson "don't add concurrency complexity speculatively" is here. Rhymes with Erlang/OTP's share-nothing (robustness dimension).*

**§3.2 Avoid *gratuitous* layers — make complexity obvious, compose from small pieces (C43 · 2/3, capped).** Expose fundamental operations directly and let complex behavior be *the sum of basic operations*, keeping "just the layers of abstraction that are really needed," so the cost of an operation stays visible to the caller.
- *Apply when:* callers benefit from seeing real cost, and complex behavior genuinely decomposes into primitives (Redis's data-type commands).
- *Skip when:* a domain needs a higher-level abstraction to be usable at all (a query planner over raw storage) — there the layer is essential. The rule is "no *gratuitous* layer," not "never abstract."
- *Calibration:* each layer hides cost and adds surface — add one only when it removes more complexity than it introduces. **SQLite is the explicit counter-case** (stays 1/3): a SQL engine is *deliberately layered* (tokenizer → parser → code-gen → VDBE → B-tree → pager → VFS) because the domain demands it — and SQLite's simplicity is *clean boundaries between* those layers, not their absence. So the sharpened rule: **avoid *gratuitous* layers; when the domain demands layers, keep their boundaries clean.**
- *Evidence:* Redis — `MANIFESTO` #3 ("avoid intermediate layers in API, so that the complexity is obvious… as the sum of the basic operations"), #4. **2nd witness Go** — "The bigger the interface, the weaker the abstraction"; small interfaces + composition over inheritance. So **2/3 (Redis + Go)** — but *capped*: SQLite is the counter-case (an inherently-layered SQL engine), which is why this can't graduate and may be inherently calibrated. (`_extract-redis.md` C43, `_extract-go.md` §3a, SQLite counter-case `_extract-sqlite.md` §6.) *Inverse of the extensibility dimension's narrow-extension-layer (C32) — open surface vs minimal core, different goals.*

**§3.3 Special-case the common case with a compact, self-contained representation (C44 · 2/3, moderate).** Don't reach for one general data structure; implement small, special-cased representations that fit the common (often small) case cheaply, each self-contained and readable, switching by size threshold when it pays.
- *Apply when:* the common case is hot and a special representation is meaningfully cheaper (Redis's `listpack` for small collections, `sds` for strings).
- *Skip when:* you're speculatively optimizing a *cold* path — that's complexity without payoff, a §1 violation. Proliferating special cases is itself complexity.
- *Calibration:* each representation plus its switching logic is code to maintain; justify it by a hot common case, and keep each representation self-contained enough to understand in isolation.
- *Evidence:* Redis — `src/listpack.c` (fast-path for small entries), `src/sds.h`; `MANIFESTO` #4, #1 (the data type *is* its space/time complexity). (`_extract-redis.md` C44.)
- *2nd witness (SQLite · moderate):* the on-disk record format uses variable-length integers (varints) + type-tagged serial values — compact for the common small value. (`_extract-sqlite.md` §6.) *(Go is neutral-to-counter here — it favors a few *general* structures, slices/maps, over many special-cased ones; so this stays a 2/3 moderate candidate, not a graduate.)*

**§3.4 Minimize the dependency surface — prefer self-contained code you understand (C45 · 3/3 — GRADUATED to P6).** Default to writing small, self-contained implementations you fully understand rather than pulling in external code; use external libraries only when they're genuinely self-contained and warranted.
- *Apply when:* the piece is small and well-understood — writing it can be *simpler* than vendoring and tracking an external dep.
- *Skip / invert when:* the thing is something a mature library does far better and safer than you would (crypto, TLS, compression) — Redis itself vendors jemalloc and Lua. The rule is "minimal, *understood* deps," not "reinvent everything."
- *Calibration:* every dependency is code you don't understand, a supply-chain surface, and a constraint on your own evolution — weigh that against the cost of owning the code.
- *Evidence:* Redis — `MANIFESTO` #5 ("beautiful self contained libraries when needed… write smaller stories that fit into other code"). (`_extract-redis.md` C45.)
- *2nd witness (SQLite — the more extreme exemplar):* **zero external dependencies** (verified: no dependency manifests/vendored libs), shipped as a **single-file amalgamation** (`tool/mksqlite3c.tcl` builds one `sqlite3.c`). (`_extract-sqlite.md` §6.)
- *3rd witness (Go) → GRADUATES:* "A little copying is better than a little dependency" (Proverb); static self-contained binaries; a self-contained batteries-included stdlib. Three unrelated repos → **P6** in `_principles-index.md`. **The boundary that graduated with it:** P6 is "minimal, *understood* deps," NOT "no deps" — reinventing crypto/TLS/compression is worse than a vetted library; all three depend on mature libs where warranted. (`_extract-go.md`.) *Cross-dimension: Envoy *governs* its dep surface (C37), Linux is largely self-contained — the same value at other altitudes.*

**§3.5 Mechanically enforce one canonical format — take style out of the discourse (C46 · 1/3).** Adopt a single tool-enforced source format with no options to argue about; run it in CI / on save; make it non-negotiable. Style stops being a review topic, diffs shrink to real changes, and every file reads the same — directly serving the understandability budget (§2 / P5).
- *Apply when:* a team of more than one, and a good canonical formatter exists for the language (gofmt, rustfmt, Prettier, Black).
- *Skip when:* no mature formatter exists, or the language has legitimate layout variety a formatter would fight — a heavy custom style-bot can cost more than the bikeshedding it prevents.
- *Calibration:* gofmt works because Go's grammar was co-designed to be canonically formattable. Mechanize the style answer *where the tooling supports it*; don't force it where it fights authors.
- *Evidence:* Go — `src/cmd/gofmt/` ("Gofmt formats Go programs"); the Proverb "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite." (`_extract-go.md` C46.) *Likely 2nd witness: Envoy mandates `clang-format` in CI (in-corpus, not yet formally extracted).*

---

## §4 Keeping the interface honest

**§4.1 Don't fake capabilities you can't deliver — expose the trade-off (C4 · 2/3).** Refuse an API that *appears* to work uniformly but silently fails outside a happy path. Where a clean abstraction can't hold (multi-key ops across a partitioned system), expose the seam and the trade-off rather than paper it with magic.
- *Apply when:* a core, correctness-sensitive surface — an honest seam is debuggable; a hidden leak surprises at the worst time.
- *Skip / soften when:* a *generic/supporting* convenience surface where a slightly-leaky ergonomic layer is the right call. The honesty rule earns its keep on core paths, not every helper.
- *Calibration:* the test is "does this abstraction hold *in all cases* I imply it does?" If not, narrow the promise or surface the seam.
- *Evidence:* Redis — `MANIFESTO` #8 ("We don't want to provide the illusion of something that will work magically when actually it can't in all cases… expose the trade-offs to the user"). (`_extract-redis.md` C4.)
- *2nd witness (Go · 2/3):* errors are explicit values, no hidden exceptions — "don't pretend an operation can't fail." Honesty about failure modes is the same value as honesty about capabilities. (`_extract-go.md` §3a.) *Cross-skill: appears one altitude up as `software-architect`'s consistency-honesty ("effectively-once via idempotence, not exactly-once") — §6.*

---

## §5 Do NOT cargo-cult (anti-pattern ledger)

The boundaries are the product. Recognize and refuse these:

- **Pointing "95% with 5%" at a correctness or security path** — the dial (§1) is a *feature* heuristic; on a critical path the missing 5% may be the catastrophic case. There P0 says *spend*, not skimp.
- **Copying "be single-threaded"** — that's Redis's *output* given an I/O-bound, shardable workload, not a universal. Extract "prefer the simplest model your workload allows; parallelize the measured hot spot" (§3.1).
- **"Write it yourself" for things a mature library does better** — minimal-deps (§3.4) is about *small, understood* pieces; reinventing crypto/TLS/compression is the opposite of simplicity. Redis vendors jemalloc and Lua.
- **Adding an abstraction layer for elegance** — a layer that hides cost without removing more complexity than it adds is accidental complexity (§3.2). "Might need it later" is an unearned complexity purchase (§1).
- **A faux-uniform API that silently degrades** — a leaky abstraction that hides its leaks is worse than an honest narrower one (§4.1).
- **Treating "simple" as "small/clever"** — the goal is *understandable* (§2), not minimal line count. A dense one-liner that no one else can modify fails the understandability budget exactly as a bloated abstraction does.

---

## §6 Cross-skill & cross-dimension links

- **The Redis ↔ Envoy counter-pole (the sharpest cross-dimension result).** Envoy and Linux *govern a large extension surface* (`extensibility-boundaries.md`); Redis's answer to the same decay pressure is to **not have one** — keep a small core and resist features that don't earn their complexity. **Both are valid, calibrated to whether an open extension surface earns its keep.** When a question is "should we make this extensible / add a plugin layer," hold *both* answers up: govern-the-surface (extensibility dimension) vs resist-the-surface (here). This is the corpus's clearest "two opposite, both-correct answers."
- **API honesty (§4.1) ↔ `software-architect`.** Redis's "don't fake capabilities" is the *craft* instance; `software-architect`'s consistency-honesty ("effectively-once via idempotence, not exactly-once") is the *architecture* instance — same value, different altitude. Keep the craft instance here; route the consistency-model question there.
- **The dial (§1) ↔ P0 and every other dimension.** The complexity budget is P0 from the complexity side; it's the same dial the testing dimension uses for rigor, robustness for machinery, and extensibility for governance tiers. Simplicity is the meta-value the whole skill's methodology already embodies (extract the calibration not the setting; the boundaries are the product).
- **Concurrency model & topology → `software-architect`.** "Single-node vs Cluster vs Proxy," "should this be an actor system" are architecture; the craft lesson "don't add concurrency complexity speculatively" (§3.1) is here.

---

## §7 Provenance & confidence

- Full evidence and calibrated verbs: `_extract-redis.md` (C3, C4, C41–C45, `44a6a9f`), `_extract-sqlite.md` §6 (`3a697a2`), `_extract-go.md` (C46 + cross-check, `fd6f414`). Practices are **Codified** (Redis `MANIFESTO`; Go Proverbs/spec) and several **Verified** (Redis `ae.c`/flat `src/`; SQLite zero-deps + `mksqlite3c.tcl` + `btree.c` density; Go `gofmt` + the ~42k-word spec).
- **Three witnesses spanning app / library / language. Two have GRADUATED at 3/3:** the understandability budget (**C3 → P5**) and minimal-deps (**C45 → P6**) — the first principles into `_principles-index.md` from a non-testing dimension. **At 2/3:** avoid-gratuitous-layers (C43, Redis + Go, *capped* by SQLite's counter-case), API honesty (C4, Redis + Go), simplest-model (C42, *two valid forms*), compact-representations (C44, moderate). **At 1/3:** mechanical formatting (C46, Go-only; Envoy `clang-format` a likely 2nd).
- **Independence note on the graduations (keep visible):** Redis and SQLite share a single-author-C-systems profile, so two of the three witnesses are correlated; **Go is the cross-paradigm anchor** that makes P5/P6 credible — comparable independence texture to P1–P4 ("2 DBs + 1 orchestrator"), graduated to the same standard. A 4th witness from a non-C, non-systems paradigm (a functional language; a UI framework) would harden them further.
- **Anchored corpus-wide:** the dial (§1, C41) *is* **P0**, and the dimension is the deliberate **counter-pole** to extensibility. Cross-dimension rhymes: §3.1 ↔ Erlang/OTP share-nothing, §3.4/P6 ↔ Envoy C37 / Linux self-containment, §2/P5 ↔ Linux C40 (minimize the surface).
- Confidence tags are honest snapshots. The Redis `MANIFESTO` + Go Proverbs make this the corpus's most *articulate* dimension — it states its own reasoning, which is partly why two principles graduated cleanly.
