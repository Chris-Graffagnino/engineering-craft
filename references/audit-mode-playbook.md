# Audit mode — whole-codebase craft review (operational layer)

**Role:** consumed when the user asks for a **whole-codebase craft review** — `/engineering-craft audit [path]` (default: repo root), or the `/audit` alias. Audit is **not a new kind of judgment**: it is Review mode applied at repo scale, with a *prioritization front-end* and a *coverage-honesty back-end*. The four dimension references and `review-mode-playbook.md` own the **judgment**; this file owns the **coverage**.

**Provenance / evidence species.** Like `review-mode-playbook.md`, this is an **operational layer, not a knowledge dimension** — it adds **no** principles to `_principles-index.md`. It operationalizes principles the corpus already graduated: **P0** (calibrate rigor to blast radius × patch latency) becomes the *prioritization function*; **Review mode** — and through it P1–P12 — becomes the *per-hotspot judgment*; the skill's standing honesty rule (name what you did not cover) becomes the *coverage report*. The one thing audit genuinely adds is the explicit refusal to *imply* coverage it did not achieve.

**The governing idea.** A whole codebase will not fit in one context and cannot be reviewed exhaustively under any real budget. The characteristic failure is a review that *reads as* complete while having silently sampled. Audit's entire job is to (a) spend a bounded attention budget where blast radius is highest (P0), and (b) report exactly what that budget did and did not buy. **Coverage is declared, never implied.**

---

## The procedure

### 0. Scope, altitude, budget — before reading anything

- **Acquire the tree, never from memory.** Establish the target — repo root, or the `[path]` given — and size it: `git ls-files | wc -l`, language/LoC ballpark, where the test dirs and any Tier-1 policy files are. If it isn't a repo or the path is empty, say so and stop.
- **Hold craft altitude.** Audit judges *how the code is built*, not the *shape of the system*. Service boundaries, decomposition, consistency-model choice, ADRs → name the seam and hand off to `software-architect`. State this up front so the audit's altitude is honest.
- **Declare the budget before reviewing.** Say how many modules/files will get a deep pass and on what basis. This is a contract, not a guess refined later.

> **Counter-case — small/bounded repo.** If the tree fits in context, skip steps 1–2 entirely and review it exhaustively. The prioritization machinery below is pure overhead under that threshold; say you reviewed the whole thing rather than pretending a sample was principled.

### 1. Cheap repo-wide sweep → the hotspot map

Run the **greppable tripwires** from `review-mode-playbook.md` §2 across the *whole tree*. They are mechanical and scale ~linearly — which is exactly why they are the first pass, not the deep one. Because it is mechanical and read-heavy, the sweep is also the cheapest stage to run and the most wasteful on a frontier model — delegate it to the **cheapest capable model at low effort** (see *Model tier* in §2; this is the shape of Claude Code's Explore subagents, which run on Haiku). Build a map: which dimensions each region implicates, where tripwires cluster. A tripwire is a **place to look, not a verdict** (the SQLite-amalgamation counter-case in the playbook still holds — a hit can be the right call under a stated constraint).

Signals cheap enough to run repo-wide:

- **Structural blast-radius proxies:** import/dependency fan-in (high fan-in ⇒ high blast radius), directory size × churn (`git log --format= --name-only | sort | uniq -c | sort -rn`), test-to-code ratio per module, presence/absence of Tier-1 policy.
- **Tripwire density:** clusters of §2 red-flags — swallowed errors, assertion-free tests, duplicated blocks, convention drift across a busy path.
- **If the code is AI-generated / AI-assisted:** also run the `ai-slop-antipatterns.md` §3 inversion map as extra tripwires (the highest-precision, greppable ones first).

### 2. Prioritize by P0 — what earns the deep pass

This is the step that makes an audit *principled* rather than *arbitrary*. Rank candidate regions by **blast radius × patch latency**:

- **Blast radius** — fan-in; whether it sits on a commit / recovery / consensus / auth / billing / data-integrity path; how many callers depend on its contract (P10).
- **Patch latency** — how slowly a defect here could be fixed and shipped: a released client library or an on-disk/wire format (slow, irreversible) ranks far above an internal service, which ranks above a leaf script.

The top of that ranking gets Review mode; the long tail gets the cheap pass only. **Record the ranking and the cutoff** — together they *are* the audit's coverage contract, and they belong at the top of the report.

**Model tier — calibrate the model to the work, the same way P0 calibrates the rigor.** The ranking you just built does double duty: it sets the deep-pass cutoff *and* assigns each region's **model**, because model firepower is a cost dial in the same shape as rigor. This adds **no principle** — it is P0's calibration logic pointed at a second resource (tokens per stage). Three bands over the same ranking:

- **Mechanical / read-heavy work → cheapest capable model, low effort.** The §1 sweep and any other greppable, places-to-look-not-verdicts pass. It ingests the whole tree and emits little, so it is the most wasteful stage on a frontier model and the safest to drop down — it renders no verdicts.
- **Mid-P0 hotspots that still clear the cutoff → mid-tier model.** Genuine Review-mode judgment, but on regions below the top of the ranking.
- **Top-P0 hotspots → frontier model.** Where a wrong call is most expensive and the judgment is hardest — citing Pn, attaching the boundary, calibrating the evidence verb. This is where the firepower earns its cost.

On Claude Code the concrete mapping is **Haiku** (sweep) / **Sonnet** (mid-P0) / **Opus** (top-P0); because audit is read-heavy, the cheaper tiers' lower *input* price is where most of the saving lands. On a single-model host — or one with no model selection (Codex, others) — the bands collapse to that one model and the method is unchanged: **tiering changes cost, never standards.** Name the tier each band ran on in the coverage report; it belongs in the same contract as the P0 cutoff.

### 3. Deep-review the hotspots — Review mode, unchanged

For each prioritized region, run **Review mode** exactly as defined: acquire the code, map it to dimension(s), load those references + `review-mode-playbook.md`, judge each finding as *"this code does X; principle Pn says Y; the condition that would flip it is Z — confirm it applies here,"* and calibrate verbs to evidence. Per-finding rigor does **not** relax at audit scale — only *which* regions earn it does.

### 4. Fan-out — opt-in accelerator, host-dependent

The default is a **sequential sweep** (steps 1–3 in one context): portable, lowest cost, runs on Codex and other agents. On a host with subagent orchestration (e.g. Claude Code) the deep pass MAY fan out — one reviewer per **module × dimension**, then synthesize. Fan-out buys wall-clock and breadth at materially higher token cost; it changes **throughput, not the method or the standards**. If you fan out: say so, and **dedupe overlapping findings** before reporting.

**Assign each worker's model by its P0 band** (§2, *Model tier*). When the deep pass fans out, set the per-reviewer model at spawn — top-P0 reviewers on the frontier model, mid-P0 reviewers on the mid-tier, the §1 sweep already on the cheapest. The fan-out is already a set of subagents, so tiering them down costs no cache you were not already paying: switching models *within* one context invalidates the prompt cache, but a subagent is a fresh context, so the cheaper tier carries no such penalty — which is precisely why a cheap pass is a subagent, not a main-loop model switch. In Claude Code, pass `model` (and `effort`) on the `Agent` / `Workflow` call; on hosts without per-agent model selection the deep pass runs at one tier, and only throughput changes.

### 5. Synthesize & report — coverage is the headline

Write the report (default `AUDIT.md`, or the given path) and print a short summary inline. The report MUST make coverage legible *first*:

- **Scope & coverage (lead with this).** What was deep-reviewed, what got the cheap pass only, what was excluded and why — plus the P0 ranking and cutoff from step 2, the **model tier each band ran on** (§2 — part of the same coverage contract), and which craft dimensions the codebase exercised vs. never surfaced.
- **Altitude note.** Anything handed off to `software-architect`.
- **Findings, grouped by dimension,** ordered by P0 weight. Each carries: the code (`file:line`), the principle (P0–P12) it bears on, the **boundary condition** that would flip it, and a **calibrated verb** (Verified / Measured / Codified / Documented / Inferred / Folklore). No bare imperatives.
- **Blind spots / what would raise confidence.** The cheapest next checks — a tripwire still to verify, a high-rank module not yet deep-read. Audit names its own blind spots rather than implying it has none.

---

## Report skeleton (`AUDIT.md`)

```markdown
# Craft Audit — <repo> @ <commit>

## Scope & coverage
- Tree: <N files, langs, LoC>. Altitude: craft only (architecture → software-architect: <items, or none>).
- Budget: deep pass on <k of N> modules, chosen by P0 (blast radius × patch latency).
- P0 ranking + cutoff: <ranked list; line where deep-review stopped>.
- Model tier by band: <sweep: cheapest; mid-P0: mid-tier; top-P0: frontier — or single-model where the host has no per-agent selection>.
- Dimensions exercised: <testing / simplicity / robustness / extensibility>. Not surfaced: <…>.

## Findings (by dimension, P0-ordered)
### <Dimension> — <Pn>
- **<title>** — `file:line`. <code does X>; <Pn> says Y; flips if Z. Evidence: **<verb>**.

## Blind spots / raise-confidence-next
- <cheapest unrun check, highest-rank module not yet deep-read, tripwire to confirm>
```

---

## Boundary conditions (when audit changes or does not apply)

- **Small/bounded repo →** skip steps 1–2; review exhaustively (the §0 counter-case).
- **A diff, a PR, or a single component →** that is **Review mode**, not audit. Audit is for the *standing state* of a tree with no natural change-set boundary.
- **"How does this repo do X / what does it teach?" →** that is **Distill mode** (extract reusable practice into the corpus) — a different intent and a different output.
- **Architectural characterization ("analyze this repo") →** `software-architect`'s Analyze mode. Audit holds craft altitude and hands that off.
- **A clean audit is a real result.** Finding little in a well-built repo is an outcome, not a failure to look harder — *provided* the coverage contract (step 2) shows you looked where blast radius was highest. An audit with no findings and no stated coverage is worthless; an audit with no findings and an honest P0 contract is a verdict.
