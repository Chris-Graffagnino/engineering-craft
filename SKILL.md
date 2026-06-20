---
name: engineering-craft
description: Engineering-craft practices for building, testing, and structuring code at the implementation and component level — distilled from gold-standard open-source codebases. Covers testing and correctness, error and failure handling, extensibility and module/plugin boundaries, backpressure and resource control, observability, API and backward-compatibility discipline, and implementation simplicity. Use whenever the user asks how to build, structure, or test a component; the best-practice or "how do the best codebases do it" approach for a code-level concern (testing, error handling, flow control, logging/metrics, API stability); for a craft-level review; or to distill engineering practices from a repository — even if they don't say "craft". Do NOT use for system-level architecture decisions — service boundaries, monolith-vs-microservices, distributed transactions, consistency models, bounded contexts, ADRs, or team topology — the software-architect skill owns those; hand off to it.
license: MIT
metadata:
  author: Chris Graffagnino
  version: 0.3.1
  status: v1 — four craft dimensions graduated (P0–P12) from a ten-repo corpus
---

# Engineering Craft

Operating instructions for reasoning about **implementation-level engineering craft** — the practices that distinguish gold-standard codebases at the level *below* system architecture. This skill is the craft complement to `software-architect`: that skill judges the *shape of the system*; this one judges *how the code is built*.

The distilled practices live in `references/` and are extracted from a small corpus of exemplar codebases using the method in this file. **The skill grows by running Distill mode on new repos; it is consumed by running Apply mode when building or reviewing code.**

## Relationship to `software-architect` (read this first — it is the routing rule)

These two skills share an epistemic backbone (evidence calibration, "what would flip this," least-worst framing) but own **disjoint altitudes**. Routing must respect the seam:

| | `software-architect` | `engineering-craft` (this skill) |
|---|---|---|
| Altitude | System / service boundaries | Code, component, module |
| Question | "Should we split / merge / restructure?" | "How should we *build / test / harden* this?" |
| Grounded in | Canonical books | Exemplar repos + their codified policy/design docs |
| Output | ADRs, fitness functions, trade-off calls | Craft practices + their boundary conditions |
| Mode | Evaluative (judge *this* system) | Generative (reusable craft) + Apply |

**Hand off to `software-architect`** the moment a question becomes about service granularity, distributed transactions/sagas, consistency-model selection, bounded contexts, team topology, or a whole-codebase *architectural* review. **Hand off to this skill** the moment it becomes about how to implement, test, extend, instrument, or simplify a component. When a request spans both, name the seam and run them in sequence (architecture decides the boundary; craft builds inside it).

"Analyze this codebase" at the *architectural* altitude (system / service shapes) is `software-architect`'s Analyze mode — hand it off. This skill has **two codebase-facing modes at the *craft* altitude**: **Distill** (extract reusable practices *from* an exemplar, into the corpus) and **Audit** (whole-codebase craft review of *this* repo). Route by intent: reusable practice → Distill; judge how *this* code is built → Audit; system shape → `software-architect`.

## When to use this skill

**Use when the user asks about:**
- How to build, structure, or test a specific component or module
- The best-practice / "how do the best codebases do it" approach to a code-level concern: testing strategy, error and failure handling, plugin/extension design, backpressure and resource limits, logging/metrics/tracing, API and backward-compatibility discipline, implementation simplicity
- A craft-level review of an implementation (not a wholesale architecture review)
- Distilling engineering practices from a particular repository into reusable form

**Do NOT use for** (→ route to `software-architect`):
- Service boundaries, decomposition, monolith-vs-microservices
- Sagas, distributed transactions, eventual-consistency design
- Consistency-model / replication / isolation selection
- Bounded-context analysis, domain modeling
- ADR drafting, fitness-function design, team topology
- Whole-codebase *architectural* characterization ("analyze this repo")

**If unclear:** ask "are you asking how to *build/test/harden a component*, or how to *structure the system*?" This skill is for the former.

## Handling arguments

- **No arguments** — greet, ask what craft question or task they have, proceed with *Apply mode*.
- **`help`** — print `assets/help.md` verbatim and stop.
- **`review`** (optionally a PR number, `diff`, or a path) — run *Review mode* below against that change set.
- **`audit`** (optionally a path; default repo root) — run *Audit mode* below: a whole-codebase craft review. `/audit` is an alias.
- **`distill`** (optionally a repo path or name) — run *Distill mode* below against that repo.
- **A natural-language craft question** — proceed with *Apply mode*.

## The craft dimensions (reference map)

Each dimension is distilled from one or more exemplar repos and lives in its own reference file. Load a reference only when the question touches its dimension (progressive disclosure).

| Reference file | Dimension | Primary exemplar(s) | Status |
|---|---|---|---|
| `references/testing-correctness.md` | Testing rigor, invariant discipline, deterministic/simulation testing | SQLite, FoundationDB | ☑ drafted |
| `references/extensibility-boundaries.md` | Extension/plugin design, module ownership, contract surfaces | Envoy, Linux, VS Code | ☑ drafted; **core graduated → P10, P11, P12** |
| `references/implementation-simplicity.md` | Simplicity under constraint, understandability budget, API honesty | Redis, SQLite, Go | ☑ drafted; **C3→P5, C45→P6 graduated** |
| `references/operational-robustness.md` | Reconciliation/retry, failure handling, resource control, observability | Kubernetes, Erlang/OTP, Temporal | ☑ drafted; **core graduated → P7, P8, P9** |
| `references/_principles-index.md` | **Cross-repo convergence-validated principles** (the capstone) | all | ☑ created (**P0–P12**) |
| `references/_extraction-template.md` | The per-repo instrument (used by Distill mode) | — | ☑ done |
| `references/_progress.md` | Corpus tracker | — | ☑ done |
| `references/ai-slop-antipatterns.md` | AI-slop anti-patterns — the **inversion layer** (load in Apply/Review when the change is AI-generated) | cross-industry literature (studies/RCT/essays), **not** exemplar repos | ☑ drafted; **nothing graduated, by design** — maps slop → P0–P12; §4 process cluster HELD |
| `references/_research-ai-slop-corpus.md` | Evidence/provenance layer for the AI-slop reference (the `_extract-*.md` analog) | — | ☑ done |
| `references/review-mode-playbook.md` | **Review-mode operational layer** — how-to-run-the-review lenses, cheap tripwires, output triage (companion to the four knowledge dimensions) | adapted from Cursor's `thermo-nuclear-code-quality-review` (operational review prompt), re-grounded in corpus principles — **not** exemplar repos | ☑ drafted; **nothing graduated** — operationalizes P0/P5/P10/P1–P4 |
| `references/audit-mode-playbook.md` | **Audit-mode operational layer** — whole-codebase coverage: P0 prioritization front-end + Review-mode judgment per hotspot + coverage-honesty report (load in Audit mode) | operationalizes P0 + Review mode at repo scale — **not** exemplar repos | ☑ drafted; **nothing graduated** — coverage layer over P0 + P1–P12 |

A practice only graduates into a dimension reference (and ultimately the principles index) once it passes the **convergence test** below. Until then it lives as a candidate in the per-repo extraction notes.

**Note — `ai-slop-antipatterns.md` is a different species, by design.** It is sourced from cross-industry *literature* (peer-reviewed studies, an RCT, practitioner essays), not exemplar repos, so it plays by different rules: it adds **no** principles to `_principles-index.md` (essay-convergence is not repo-convergence), and instead operates as an *inversion layer* — mapping each AI-code failure mode onto the already-graduated principle (P0–P12) it violates (AI slop is the *negative space* of the corpus). Its evidence/provenance layer is `_research-ai-slop-corpus.md` (the `_extract-*.md` analog). The one genuinely new region — human–AI *process* failures (perception gap, automation bias, comprehension debt) — is **held** as a candidate dimension and graduates only by distilling ≥3 *CI-enforced* organizational AI-coding policies (Tier-1 codified + Tier-3 verified), never by more essays. Load it in Apply/Review mode only when the change under discussion is AI-generated.

## The method (what keeps this skill honest)

Distilling from finished codebases has a known failure mode: you see the surviving *structure* but not the *constraints, rejected alternatives, and over-engineering that was later removed*. A skill built on "here's how X is structured" produces cargo-culting. These five rules are the guard.

### 1. Three-tier source model — prefer codified craft over inferred craft

Extractable signal comes in tiers, highest first. Always inventory which tiers a repo offers before reading code:

- **Tier 1 — Codified policy/philosophy** (`STYLE.md`, `EXTENSION_POLICY.md`, `DEPENDENCY_POLICY.md`, `DEPRECATED.md`, `MANIFESTO`, `CONTRIBUTING.md`, design RFC dirs). The project *stating its own rules*. Highest signal-to-noise — this is the codebase telling you its best practices directly.
- **Tier 2 — Design notes** (`docs/*.md`, in-source `DESIGN.md`, `flow_control.md`-style mechanism write-ups). The *why* behind a specific mechanism. Medium density, high value — shows reasoning and trade-offs.
- **Tier 3 — The code itself.** Verification that Tier-1 policy is actually followed, plus the practices that were never written down. Lowest density; use it to confirm or falsify, not to browse.

A practice sourced from Tier 1 or 2 and *confirmed* in Tier 3 is the strongest kind.

### 2. One thesis per repo

Each codebase teaches one or two things well. Reading all of Kubernetes "to learn engineering" is unfocused; reading its informer/workqueue/retry mechanics to learn *convergent reconciliation* is tractable. State the thesis before extracting; ignore the rest of the repo.

### 3. Convergence test — one repo is a style choice; several are a principle

A practice earns a place in a dimension reference only if it appears in **≥3 unrelated repos** doing it convergently for the same reason. One repo doing something is taste. Cross-checking is what keeps the skill from being "things I noticed in C projects." Record convergence explicitly in each candidate.

### 4. Counter-case requirement — every practice carries its boundary

A practice without a boundary is dogma. For each candidate, write down **when it does NOT apply** and, ideally, a named repo that deliberately does the opposite under a different constraint. "Do X when Y; Z's codebase deliberately doesn't because [constraint]" is the valuable form. The boundaries are the product.

### 5. Capture the rejected and the over-engineered

Record practices the repo *tried and removed*, and repo-specific choices that should **not** generalize (driven by a constraint the reader won't share — language, scale, history). This is the direct antidote to the distillation trap.

### Evidence calibration (shared backbone with `software-architect`)

Match the verb to the evidence class when asserting that a repo does something:

| Verb | Means |
|---|---|
| **Verified** | Confirmed in code (file:line) *and* a test/CI/policy enforces it |
| **Measured** | The repo's own benchmarks/docs show the effect |
| **Codified** | Stated as policy in a Tier-1 doc |
| **Documented** | Explained in a Tier-2 design note |
| **Inferred** | Read from code, not stated anywhere |
| **Folklore** | Asserted in issues/blogs/commit messages without code backing |

"Codified in `EXTENSION_POLICY.md` and verified in `CODEOWNERS`" is far stronger than "the project seems to value ownership."

## Distill mode (the generative engine)

**Invoked by:** the `distill` argument, or "distill / extract practices from [repo]", "what does [repo] teach about [dimension]".

1. **Clone shallow** (`git clone --depth 1 <url>`) unless decision-*evolution* is the thesis (then take full history for the one or two repos where it matters).
2. **Inventory the three tiers** — list the Tier-1 policy files, Tier-2 design notes, and the code regions relevant to the thesis. (Cheap: `ls` top level, `find … -iname '*.md'`, skim names.)
3. **State the thesis** (rule 2) and confirm scope with the user before deep reading.
4. **Fill `references/_extraction-template.md`** for this repo → save as `references/_extract-<repo>.md`. Every candidate gets: practice, tier/evidence (calibrated verb + path), the *why/constraint*, the counter-case, and a convergence column.
5. **Run the convergence test** (rule 3) against repos already in the corpus; promote graduating practices into the relevant dimension reference and, when validated across the corpus, into `_principles-index.md`.
6. **Update `references/_progress.md`.**

## Apply mode (the consumer-facing engine)

**Invoked by:** any craft question (the default).

1. Identify which **dimension(s)** the question touches; load only those references.
2. Surface the relevant distilled practices **with their boundary conditions** — never a bare "do X." Pair each with its counter-case (rule 4) and the condition that would flip it.
3. Calibrate verbs to evidence (don't present an *inferred* practice as *verified*).
4. If the question has drifted to architecture, **hand off to `software-architect`** rather than improvising at the wrong altitude.
5. If no distilled practice covers the question, say so and offer to **run Distill mode** on a relevant exemplar rather than inventing a best practice.

## Review mode (Apply mode pointed at a change set)

**Invoked by:** the `review` argument — `review <PR#>`, `review diff`, `review <path>`, or bare `review`.

Review mode is Apply mode aimed at a *diff* rather than a question: judge how a specific change is **built** against the distilled craft practices. It does **not** replace a correctness review (`/code-review` owns bug-hunting); it adds the craft layer — testing discipline, failure handling, boundary/contract design, resource control, observability, API/compat, simplicity.

1. **Acquire the change set** — never review from memory.
   - `review <PR#>` → `gh pr diff <PR#>` (add `--repo owner/name` if not inside the repo).
   - `review` / `review diff` → the working diff: `git diff` (uncommitted), or `git diff <base>...HEAD` for the branch vs its base.
   - `review <path>` → treat that file/dir as the change set.
   - If the diff is empty or can't be fetched, say so and stop.
2. **Map the changed code to dimension(s)** (Apply step 1) and load only those references — **plus `references/review-mode-playbook.md`** for the cross-dimension review lenses (code-judo / neighbor-entropy / triage), the cheap tripwires, and output ordering. **When the change is AI-generated, also load `references/ai-slop-antipatterns.md`** (run its §3 red-flags as extra tripwires). Most diffs touch one to three dimensions.
3. **Judge the change against each touched dimension's practices, with boundary conditions.** Each finding takes the form "this change does X; practice/principle Pn says Y; the condition that would flip it is Z — confirm it applies here." Never a bare "you should"; cite the backing principle (P0–P12) or dimension reference.
4. **Calibrate verbs to evidence** (Apply step 3) — don't present an *inferred* concern as a *verified* defect.
5. **Hold altitude.** If a finding is actually architectural (service boundary, consistency-model choice), name the seam and **hand off to `software-architect`** rather than reviewing it here.
6. **Report**, grouped by dimension and ordered by how strongly each practice applies. State explicitly which craft dimensions the diff did **not** touch, so the review's scope is honest. If no distilled practice covers a concern, say so rather than inventing one — offer Distill mode on a relevant exemplar.

## Audit mode (Review mode at repo scale)

**Invoked by:** the `audit` argument — `audit [path]` (default: repo root); `/audit` is an alias.

Audit is a **whole-codebase craft review**: Review mode with a P0 prioritization front-end and a coverage-honesty back-end. A whole repo never fits one judgment, so audit spends a bounded attention budget where blast radius is highest and reports exactly what that budget did and did not cover — **coverage is declared, never implied.** It adds no new principles; it operationalizes **P0** (the prioritization function) and **Review mode** (the per-hotspot judgment). **Load `references/audit-mode-playbook.md`** for the full procedure; the short form:

1. **Scope, altitude, budget.** Acquire the tree (never from memory) and size it. Hold craft altitude — architectural characterization → hand off to `software-architect`. Declare the coverage target *before* reading. A repo small enough to fit in context is audited exhaustively (skip steps 2–3's sampling — the prioritization machinery is overhead below that threshold).
2. **Cheap repo-wide sweep → hotspot map.** Run `review-mode-playbook.md` §2 tripwires across the whole tree (mechanical, scales ~linearly); add `references/ai-slop-antipatterns.md` §3 if the code is AI-generated. Tripwires are *places to look*, not verdicts.
3. **Prioritize by P0** (blast radius × patch latency). Record the ranking **and the cutoff** — together they are the audit's coverage contract.
4. **Deep-review the top hotspots with Review mode**, unchanged — only *which* regions earn the rigor changes, not the rigor itself. Fan-out per module × dimension is an opt-in accelerator on hosts with subagents: it changes throughput, not method or standards. **Tier the model to the work** — the same P0 ranking that sets the cutoff sets each region's model tier (cheapest model for the §1 sweep, mid-tier for mid-P0, frontier for top-P0), not just its rigor; on read-heavy audits the cheap tiers' lower input price is the main saving. See `references/audit-mode-playbook.md`.
5. **Report** (default `AUDIT.md`, or the given path) + a short inline summary, **coverage first**: what was deep-reviewed, what got the cheap pass, what was excluded and why; findings grouped by dimension, each citing its principle (P0–P12) + boundary condition + calibrated verb; and the audit's own blind spots. State which dimensions the codebase did *not* surface.

## Definition of done

A response from this skill is complete when:
- The question was routed correctly (craft here; architecture → `software-architect`), and the seam was named if it spanned both.
- Only the relevant dimension references were loaded.
- Every practice offered carries its **boundary condition** and the signal that would flip it — no bare imperatives.
- Verbs match the **evidence class** (Verified / Codified / Documented / Inferred / …).
- In Distill mode: thesis stated, three tiers inventoried, candidates carry evidence + counter-case + convergence, and rejected/over-engineered items were captured.
- In Review mode: the change set was actually obtained (not reviewed from memory), each finding cites its backing principle/reference with the boundary condition, craft and architecture were kept separate, and the dimensions the diff did *not* touch were named.
- In Audit mode: the tree was actually acquired (not audited from memory); the P0 prioritization **and cutoff** were stated as the coverage contract; each finding cites its principle with a boundary condition and a calibrated verb; craft altitude was held (architecture handed off); and what was *not* covered was named — **coverage declared, never implied.**
- What is *not* covered by the corpus has been named, not papered over.

## Status

**The v1 corpus build is complete.** All four dimension references are drafted and `_principles-index.md` holds the convergence-validated core:
1. ✅ Distill mode run on all four seed repos (SQLite, Envoy, Redis, Kubernetes) **plus six corroborating witnesses** (FoundationDB → testing; Erlang/OTP + Temporal → robustness; Linux + VS Code → extensibility; Go → simplicity) — ten `_extract-<repo>.md` files (SQLite also carries a §6 simplicity cross-witness).
2. ✅ Convergence test run; the four dimension references and `_principles-index.md` (**P0–P12**) written. **All four dimensions have graduated principles** — testing (P1–P4), simplicity (P5–P6, Redis+SQLite+Go), robustness (P7–P9, Kubernetes+Erlang/OTP+Temporal), and **extensibility (P10 narrow-versioned-contract, P11 declared-posture, P12 governed-lifecycle — Envoy+Linux+VS Code)**. Each dimension is graduated by *three independent witnesses*, with explicit holds where convergence is partial.
3. The skill grows by running **Distill mode** on new exemplars: held candidates (tracked in `_principles-index.md`) graduate only when a third independent witness corroborates them — optional hardening, since every dimension is already graduated. Three operational / literature-sourced references — `ai-slop-antipatterns.md`, `review-mode-playbook.md`, and `audit-mode-playbook.md` (whole-codebase **Audit mode**) — extend Review/Audit without adding to the convergence core.
