# Extraction: Go — Implementation Simplicity (the independence anchor)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/golang/go` (BSD) at commit `fd6f414` — `src/cmd/gofmt/` (the canonical formatter), `doc/go_spec.html` (the whole language spec, ~42k words / 8,997 lines). The simplicity *philosophy* is web-sourced and attributed inline: the **Go Proverbs** (Rob Pike, 2015, go-proverbs.github.io) and **"Go at Google: Language Design in the Service of Software Engineering"** (Pike) — the FAQ now lives in the website repo, not in-tree.

**Seam discipline:** stays on the *craft* side of the boundary with `software-architect`. It extracts Go's transferable **simplicity craft** (canonical formatting, minimal deps, readability) — practices that apply to any codebase. It does **not** extract *language design* decisions (no exceptions, no inheritance, generics-deferred) — those are language architecture, out of scope.

Dual purpose: (1) the **third simplicity witness**, and deliberately the **independence anchor** — a multi-author Google *language/toolchain*, unlike the two single-author C databases (Redis, SQLite), so the trio spans **app / library / language**; (2) it **graduates C3 and C45 to 3/3** (the first principles to reach `_principles-index.md` from a non-testing dimension — new **P5** and **P6**), lifts C43/C4 to 2/3, and is an informative **counter-point** on C42/C44. It contributes one new candidate, C46. See §3a.

---

## 0. Identity

- **Repo:** Go (`golang/go`) — the language, toolchain, and standard library.
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `fd6f414`.
- **Language(s):** Go (self-hosted), with the spec/tooling as the codified artifacts.
- **Why it's in the corpus:** The reference example of *simplicity as a language/ecosystem-wide engineering value* — and the **cross-paradigm independence** the simplicity dimension needed: it shares neither Redis's nor SQLite's "single-author C database" profile, so a convergence with it is credible across genuinely unrelated software.

## 1. Thesis

**Simplicity scales when it is made *mechanical and ecosystem-wide*: a small language a reader can hold in their head, one machine-enforced format so style stops being a debate, and a default of self-contained code with minimal dependencies.** Go shows that the Redis/SQLite simplicity ethos isn't a C-auteur quirk — it recurs, independently, in a multi-author language built for software engineering at scale.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `doc/go_spec.html` — the *entire* language definition in ~42k words (a deliberately small language; readable in an afternoon).
- The **Go Proverbs** (Pike, web): "Clear is better than clever," "A little copying is better than a little dependency," "The bigger the interface, the weaker the abstraction," "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."
- Pike, "Go at Google" (web) — simplicity/readability as explicit design goals for engineering at scale.

**Tier 3 — Code (confirm/falsify):**
- `src/cmd/gofmt/gofmt.go`, `src/cmd/gofmt/doc.go` — the canonical formatter ("Gofmt formats Go programs") and `src/go/format/` — one format, machine-enforced.
- Static self-contained binaries + a batteries-included, self-contained stdlib (canonical Go properties; web-documented).

## 3. Candidate practices (implementation-simplicity dimension)

### C46. Mechanically enforce one canonical format — take style out of the discourse

- Practice: Adopt a single, tool-enforced source format with **no options to argue about**, run it in CI / on save, and make it non-negotiable. Style stops being a code-review topic; diffs shrink to real changes; every file reads the same way.
- Evidence: **Verified** — `src/cmd/gofmt/` (`gofmt` reformats Go to one canonical layout; `go/format` is the library form). **Documented** — the Go Proverb "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite" (the rationale: a shared format nobody fully prefers beats per-person formats everyone fights over).
- Why / constraint: Formatting debates are pure complexity-of-process with no payoff; mechanizing the answer removes the debate *and* makes the whole corpus uniformly readable (serving the understandability budget, C3/P5).
- Counter-case: Needs a *good enough* canonical formatter for the language; where none exists, a heavy custom style-bot can cost more than the bikeshedding it prevents. And an over-rigid formatter on a language with legitimate layout variety can fight the author — gofmt works because Go's grammar was co-designed to be formattable.
- Convergence: **1/3** (Go). **Likely 2/3 already:** Envoy mandates `clang-format` in CI (notable — it's in-corpus; not formally extracted here). rustfmt/Prettier/Black are the broader pattern; a formal second extraction would corroborate.
- Graduates to: `implementation-simplicity.md`

## 3a. Convergence cross-check — Go as the third simplicity witness (two graduations)

Go is the corpus's **third, cross-paradigm** simplicity witness. With three unrelated repos (an in-memory store, an embedded DB, **and a language/toolchain**) two practices clear the **≥3 bar and graduate** into `_principles-index.md`.

| Redis/SQLite candidate | Go mechanism | Verdict |
|---|---|---|
| **C3** understandability budget / readable code | A ~42k-word *whole-language* spec; `gofmt` (one format); the Proverb "Clear is better than clever"; the language omits features to stay small | **2/3 → 3/3 — GRADUATES (P5)** |
| **C45** minimize the dependency surface | "A little copying is better than a little dependency" (Proverb); static self-contained binaries; a self-contained batteries-included stdlib | **2/3 → 3/3 — GRADUATES (P6)** |
| **C43** avoid *gratuitous* layers / small composable pieces | "The bigger the interface, the weaker the abstraction"; small interfaces + composition over inheritance | **1/3 → 2/3** (Redis + Go; **SQLite remains the counter-case** — an inherently-layered SQL engine — so this *caps at 2/3*, it does not graduate) |
| **C4** API honesty — don't fake capabilities | Errors are explicit values, no hidden exceptions ("don't pretend an operation can't fail") | **1/3 → 2/3** (Redis + Go) |
| **C42** simplest execution model | **COUNTER-POINT:** Go *embraces* concurrency (goroutines/channels) — it makes the hard thing *simple* rather than avoiding it. The opposite move from Redis/SQLite (avoid concurrency machinery). | **stays 2/3** (Redis + SQLite); Go sharpens it (see below) |
| **C44** special-case compact representations | **MILD COUNTER:** Go favors a few *general* structures (slices, maps) over many special-cased ones | **stays 2/3 moderate**; Go is neutral-to-counter |
| **C41** complexity budget / 95-5 | Strongly corroborates ("add a feature only when it pays" — generics deferred a decade) but is **P0** from the complexity side | corroborated; P0-anchored |

**The two graduations (P5 understandability, P6 minimal deps)** are the first principles to enter the index from a dimension other than testing/robustness. **Honesty note carried into the index:** Redis and SQLite share a "single-author C systems project" profile, so two of the three witnesses are correlated; **Go is the cross-paradigm independence anchor** that makes the trio credible. This is comparable independence texture to P1–P4 (whose witnesses are 2 databases + 1 orchestrator) — graduated to the same standard, not a looser one. A *fourth* witness from yet another paradigm (a functional language; a different-domain system) would further harden P5/P6.

**The C42 sharpening (record it).** With Redis/SQLite *avoiding* concurrency and Go *simplifying* it, the durable craft lesson is **"don't pay for concurrency complexity you don't need — either avoid the machinery (Redis/SQLite) or make it cheap and safe (Go), per your workload."** The two are not in conflict; they're two settings of the same dial. So C42 stays a *2/3 candidate with two valid forms*, not a graduate.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            Go's specific language-design omissions (no exceptions, no inheritance,
                   generics deferred for a decade).
- Why not general: These are LANGUAGE-design calls, not transferable code craft, and several were
                   later revised (generics shipped in 1.18). Extract the VALUE (defer complexity
                   until it pays — C41/P0), not the specific omissions.
```

```
- Item:            "A little copying is better than a little dependency" as an absolute.
- Why not general: A heuristic, not a law. Copying scales badly past "a little" (drift, divergent
                   bug-fixes); and reinventing crypto/TLS/compression is far worse than depending
                   on a mature library — Go's own stdlib depends on vetted implementations. The
                   principle (P6) is "minimal, UNDERSTOOD deps," not "copy instead of depend."
```

```
- Item:            One mandated formatter (gofmt) for every language/team.
- Why not general: gofmt works because Go's grammar was co-designed to be canonically formattable
                   and a good tool exists. On a language with legitimate layout variety or no
                   mature formatter, mandating one can fight authors. Adopt the PRINCIPLE
                   (mechanize the style answer where you can) calibrated to tooling reality.
```

## 5. Open questions / corpus cross-checks

- **Graduations done:** C3 → **P5**, C45 → **P6** (3/3 across Redis, SQLite, Go). Move them from the tracked-candidates ledger into the validated core; keep the independence caveat visible.
- **C43** now 2/3 (Redis + Go) but **cannot graduate** while SQLite is its counter-case — a non-layered-domain third witness would be needed, and the counter-case may simply mean it's *inherently* a calibrated practice, not a universal.
- **C4** now 2/3 (Redis + Go) — a third honesty witness (a DB/protocol library with explicit failure contracts) would graduate it.
- **C46** (mechanical formatting) is 1/3 but Envoy's mandated `clang-format` is an in-corpus likely-corroborator — a quick check would lift it to 2/3.
- A **fourth** simplicity witness from a non-C, non-systems paradigm (a functional language; a UI framework) would harden P5/P6 past the Redis↔SQLite correlation and test whether C42's "two valid forms" holds.
