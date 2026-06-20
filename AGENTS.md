# Engineering Craft — AGENTS.md adapter

Portable entry point for **Codex and other `AGENTS.md`-aware agents** (Cursor, Gemini CLI, Jules, …).

This skill judges **how code is built** at the implementation/component level — testing and
correctness, failure handling, implementation simplicity, extension/plugin boundaries, resource
control, observability, and API/backward-compatibility discipline. It is the craft complement to a
*system-architecture* skill, which judges the shape of the system.

## What this file is

The distilled knowledge in [`references/`](references/) is **model-agnostic** — plain engineering
practice extracted from exemplar codebases. Only the *packaging* was Claude-specific. This file is the
adapter that replaces the three Claude-native mechanisms so the same corpus runs here:

| Claude skill mechanism | Replaced here by |
|---|---|
| `SKILL.md` YAML frontmatter + `/engineering-craft` slash command | This Markdown file + the **trigger conditions** below (self-trigger; no slash UI) |
| Native progressive disclosure (auto-load one reference) | **Manual**: the *reference map* tells you which file to open and read on demand |
| Handoff to a `software-architect` skill | The **architecture seam** below, with graceful degradation when no such skill exists |

[`SKILL.md`](SKILL.md) remains the canonical, full-length method and the Claude-native entry point.
This file inlines enough to run **Apply** and **Review** standalone; for the full **Distill** machinery
and the five method rules in full, read `SKILL.md`.

> **Read the references — do not answer from this file alone.** The actual practices, their evidence
> tags, and their boundary conditions live in `references/`. This file only routes you there and states
> the discipline for using them.

## How to use with Codex / other agents

- **To use the corpus in your own project:** clone this repo somewhere, then in *your* project's root
  `AGENTS.md` add a pointer — e.g. *"Engineering-craft practices live at `<path>`; when a code-level
  craft question or craft review arises, follow `<path>/AGENTS.md`."* Or copy this file's body into your
  `AGENTS.md` and fix the `references/` paths to point at the checkout.
- **To work on the corpus itself:** this file sits at the repo root, where Codex reads it automatically
  for any work in this repo.
- Codex merges `~/.codex/AGENTS.md` (global), the repo-root file, and a cwd file. Paths below are
  relative to **this file** — adjust them if you relocate it.

## When to engage (there is no slash command — you must self-trigger)

Engage these practices, even if the user never says the word "craft," when they ask:

- how to **build, structure, or test** a specific component or module;
- the best-practice / "how do the best codebases do it" approach to a code-level concern — testing
  strategy, error/failure handling, plugin/extension design, backpressure and resource limits,
  logging/metrics/tracing, API and backward-compatibility discipline, implementation simplicity;
- for a **craft-level review** of an implementation (a complement to, not a replacement for, a
  correctness/bug review);
- for a **whole-codebase craft review** ("audit this repo for craft") — run Audit mode;
- to **distill** engineering practices from a particular repository.

**Do not engage** (→ architecture seam) on: service boundaries, monolith-vs-microservices, sagas /
distributed transactions, consistency-model selection, bounded-context analysis, ADRs, fitness
functions, team topology, or whole-codebase *architectural* characterization ("analyze this repo" is
architecture's Analyze mode — this skill's codebase-facing mode is **Distill**, a different intent).

**If unclear:** ask *"are you asking how to build/test/harden a component, or how to structure the
system?"* This skill is for the former.

## The reference map (manual progressive disclosure)

Open and read **only** the reference(s) the question touches:

| Open this | When the question touches | Principles | Exemplars |
|---|---|---|---|
| [`references/testing-correctness.md`](references/testing-correctness.md) | testing rigor, invariant discipline, fault injection, deterministic/simulation testing | P1–P4 | SQLite, FoundationDB, Kubernetes |
| [`references/implementation-simplicity.md`](references/implementation-simplicity.md) | simplicity under constraint, understandability budget, dependency surface, API honesty | P5–P6 | Redis, SQLite, Go |
| [`references/operational-robustness.md`](references/operational-robustness.md) | reconcile/retry, failure handling, resource control, observability | P7–P9 | Kubernetes, Erlang/OTP, Temporal |
| [`references/extensibility-boundaries.md`](references/extensibility-boundaries.md) | extension/plugin design, module ownership, contract surfaces | P10–P12 | Envoy, Linux, VS Code |
| [`references/_principles-index.md`](references/_principles-index.md) | the convergence-validated core / the governing dial | **P0–P12** | all |
| [`references/ai-slop-antipatterns.md`](references/ai-slop-antipatterns.md) | **load in Apply/Review when the change is AI-generated** — maps slop onto the principle it violates | — (inversion layer) | cross-industry literature |
| [`references/review-mode-playbook.md`](references/review-mode-playbook.md) | **load in Review mode** — review lenses, cheap tripwires, output triage | — (operational layer) | — |
| [`references/audit-mode-playbook.md`](references/audit-mode-playbook.md) | **load in Audit mode** — whole-codebase coverage: P0 prioritization + per-hotspot Review + coverage report | — (operational layer) | — |
| [`references/_extraction-template.md`](references/_extraction-template.md) | the instrument **Distill** mode fills for a new repo | — | — |
| [`references/_progress.md`](references/_progress.md) | corpus development log | — | — |

## The non-negotiable discipline (what keeps the references honest)

Apply these to every answer; they are the reason the corpus is not just assertion.

1. **Every practice ships with its boundary condition.** Never a bare "do X." State *when it does not
   apply* and the signal that would flip it — ideally a named repo that deliberately does the opposite
   under a different constraint. The boundaries are the product.
2. **Calibrate the verb to the evidence class** — never present an inferred practice as a verified one:

   | Verb | Means |
   |---|---|
   | **Verified** | Confirmed in code (file:line) *and* a test/CI/policy enforces it |
   | **Measured** | The repo's own benchmarks/docs show the effect |
   | **Codified** | Stated as policy in a Tier-1 doc (`STYLE.md`, `CONTRIBUTING.md`, RFCs…) |
   | **Documented** | Explained in a Tier-2 design note |
   | **Inferred** | Read from code, not stated anywhere |
   | **Folklore** | Asserted in issues/blogs/commits without code backing |

3. **Convergence ≠ style.** A practice is a *principle* only when witnessed independently in **≥3
   unrelated repos** doing it convergently for the same reason. One repo is a style choice.
4. **The governing dial (P0).** Rigor is calibrated to **blast radius × patch latency**. Extract the
   *calibration*, never the *setting* — SQLite's 590:1 test-to-code ratio is the output of an extreme
   dial, not a target to copy.
5. **If no distilled practice covers the question, say so** and offer to run Distill mode on a relevant
   exemplar — do not invent a best practice to fill the gap.

## Modes

### Apply mode (default — any craft question)

1. Identify which dimension(s) the question touches; open and read **only** those references.
2. Surface the relevant practices **with their boundary conditions** and the condition that would flip
   each (discipline #1).
3. Calibrate verbs to evidence (discipline #2).
4. If the question has drifted to architecture, follow the **architecture seam** instead of improvising
   at the wrong altitude.
5. If nothing in the corpus covers it, say so and offer Distill mode (discipline #5).

### Review mode (Apply pointed at a change set)

Adds the craft layer to a review; it does **not** replace a correctness/bug review.

1. **Acquire the change set — never review from memory.** Use the shell:
   - a PR number → `gh pr diff <N>` (add `--repo owner/name` if outside the repo);
   - working changes → `git diff` (uncommitted) or `git diff <base>...HEAD` (branch vs base);
   - a path → treat that file/dir as the change set.
   - If the diff is empty or can't be fetched, say so and stop.
2. Map the changed code to dimension(s) and read those references **plus**
   [`references/review-mode-playbook.md`](references/review-mode-playbook.md). **If the change is
   AI-generated, also read [`references/ai-slop-antipatterns.md`](references/ai-slop-antipatterns.md)**
   and run its §3 red-flags as extra tripwires.
3. Frame every finding as *"this change does X; principle Pn (or dimension ref) says Y; the condition
   that would flip it is Z — confirm it applies here."* Cite the backing principle; never a bare
   "you should."
4. Calibrate verbs to evidence (discipline #2) — an inferred concern is not a verified defect.
5. **Hold altitude.** Architectural findings → name the seam and hand off; do not review them here.
6. **Report**, grouped by dimension and ordered by how strongly each practice applies. State explicitly
   which craft dimensions the diff did **not** touch, so the review's scope is honest.

### Audit mode (Review mode at repo scale)

A **whole-codebase craft review** — Review mode with a P0 prioritization front-end and a
coverage-honesty back-end. A whole repo never fits one judgment, so audit spends a bounded attention
budget where blast radius is highest and reports exactly what it did and did not cover — **coverage is
declared, never implied.** It adds no new principles; it operationalizes P0 (prioritization) and Review
mode (judgment). **Read [`references/audit-mode-playbook.md`](references/audit-mode-playbook.md)** for
the full procedure; the short form:

1. **Scope, altitude, budget.** Acquire the tree via the shell (`git ls-files`, `ls`) — never from
   memory — and size it. Hold craft altitude (architectural characterization → the architecture seam).
   Declare the coverage target *before* reading. A repo small enough to fit in context is audited
   exhaustively (skip steps 2–3).
2. **Cheap repo-wide sweep → hotspot map.** Run the `review-mode-playbook.md` §2 tripwires across the
   whole tree (mechanical, scales ~linearly); add `ai-slop-antipatterns.md` §3 if the code is
   AI-generated. Tripwires are *places to look*, not verdicts.
3. **Prioritize by P0** (blast radius × patch latency); record the ranking **and cutoff** — the
   coverage contract.
4. **Deep-review the top hotspots with Review mode**, unchanged. Fan-out per module × dimension is an
   opt-in accelerator on hosts with subagents — it changes throughput, not method or standards.
5. **Report** (default `AUDIT.md`, or the given path) + inline summary, **coverage first**: what was
   deep-reviewed, what got the cheap pass, what was excluded and why; findings grouped by dimension,
   each citing its principle + boundary + calibrated verb; and the audit's own blind spots. Coverage is
   **declared, never implied**.

### Distill mode (extract reusable practices from an exemplar repo)

Compact loop — for the full machinery and the five method rules, read `SKILL.md` (§ "The method" and
§ "Distill mode"):

1. **Clone shallow** (`git clone --depth 1 <url>`) unless decision-*evolution* is the thesis.
2. **Inventory the three source tiers** — Tier 1 codified policy (`STYLE.md`, `CONTRIBUTING.md`, RFC
   dirs) > Tier 2 design notes > Tier 3 the code itself. Prefer codified over inferred; confirm Tier-1
   claims in Tier-3.
3. **State one thesis** for the repo and confirm scope before deep reading (a repo teaches one or two
   things well; ignore the rest).
4. **Fill** [`references/_extraction-template.md`](references/_extraction-template.md) → save as
   `references/_extract-<repo>.md`. Every candidate carries: practice, calibrated evidence verb + path,
   the *why/constraint*, the **counter-case**, and a convergence column. Capture what the repo *tried
   and removed*, and repo-specific choices that should **not** generalize.
5. **Run the convergence test** (≥3 unrelated repos) against the existing corpus; promote graduating
   practices into the dimension reference and, when validated, into `_principles-index.md`.
6. **Update** [`references/_progress.md`](references/_progress.md).

## The architecture seam (handoff with graceful degradation)

This skill owns **only** the code/component altitude. The moment a question becomes about service
granularity, monolith-vs-microservices, distributed transactions/sagas, consistency-model selection,
bounded contexts, ADRs, or team topology, it is **architecture**:

- If your environment has a software-architecture skill, agent, or doc — e.g. the companion
  [`software-architect`](https://github.com/Chris-Graffagnino/software-architect) — **name the seam and
  hand off**.
- If it has none, say plainly that the question is **out of craft scope** rather than answering at the
  wrong altitude.
- When a request spans both, name the seam and run them in sequence: **architecture decides the
  boundary; craft builds inside it.**

## Definition of done

A response is complete when: the request was routed correctly (craft here; architecture → seam) and the
seam was named if it spanned both; only the relevant references were read; every practice carries its
boundary condition and the signal that would flip it; verbs match the evidence class; and what the
corpus does *not* cover was named, not papered over. In **Review** mode, additionally: the change set
was actually obtained (not reviewed from memory), each finding cites its backing principle with a
boundary condition, and the dimensions the diff did *not* touch were named. In **Audit** mode: the tree
was acquired (not audited from memory), the P0 prioritization + cutoff were stated as the coverage
contract, craft altitude was held, and what was *not* covered was named — coverage declared, never
implied. In **Distill** mode: thesis
stated, three tiers inventoried, candidates carry evidence + counter-case + convergence, and
rejected/over-engineered items were captured.
