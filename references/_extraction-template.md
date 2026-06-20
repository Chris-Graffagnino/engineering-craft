# Per-Repo Extraction Template

The instrument Distill mode fills for each exemplar. Copy this file to `_extract-<repo>.md` and complete it. The goal is **not** to describe the repo — it is to extract *transferable craft with boundaries*. Resist the urge to write a tour; every row must be something a reader could apply (or know not to apply) elsewhere.

Read the method in `../SKILL.md` first. The five rules (three-tier sourcing, one thesis, convergence test, counter-case requirement, capture-the-rejected) and the evidence-calibration verbs are assumed below.

---

## 0. Identity

- **Repo:**
- **Commit / clone depth:**
- **Language(s):**
- **Why it's in the corpus (one line):**

## 1. Thesis (rule 2 — one or two things this repo teaches well)

> What is the *one* craft lesson worth extracting here? Everything below serves this. If you find yourself extracting things unrelated to the thesis, either you picked the wrong thesis or those belong to a different repo's extraction.

-

## 2. Three-tier source inventory (rule 1)

List what the repo actually offers before reading code. This is the audit trail for the evidence column.

**Tier 1 — Codified policy/philosophy** (the project's own rules):
- _e.g._ `STYLE.md`, `EXTENSION_POLICY.md`, `DEPENDENCY_POLICY.md`, `DEPRECATED.md`, `MANIFESTO`, `CONTRIBUTING.md`, RFC dir

**Tier 2 — Design notes** (the *why* behind mechanisms):
- _e.g._ `docs/*.md`, in-source `DESIGN.md`, mechanism write-ups

**Tier 3 — Code regions relevant to the thesis** (confirm/falsify only):
- _e.g._ `src/<files>`

## 3. Candidate practices

One row block per candidate. **Do not** record a practice without an evidence path and a counter-case — an entry missing either is not yet extracted, it's an impression.

```
### C<n>. <short imperative name>

- Practice:        <what to do, one or two sentences>
- Evidence:        <Verified|Codified|Documented|Inferred|Measured|Folklore> — <path or doc:section>
- Why / constraint: <the problem this solves; the constraint that produced it>
- Counter-case:    <when it does NOT apply; ideally a named repo that does the opposite and why> (rule 4)
- Convergence:     <which other corpus repos also do this, same reason> (rule 3 — fill during cross-check)
- Graduates to:    <dimension reference this would join, once convergence ≥3>
```

## 4. Rejected / over-engineered / do-NOT-generalize (rule 5)

The distillation-trap antidote. Capture:
- Practices the repo **tried and removed** (check `DEPRECATED.md`, RFC graveyards, revert commits).
- Choices that are **repo-specific** and should not transfer — driven by a constraint the reader won't share (language quirk, extreme scale, historical accident, licensing). Name the constraint.

```
- Item:            <the practice or choice>
- Why not general: <the local constraint that makes it inapplicable elsewhere>
```

## 5. Open questions for the user / corpus

Things the code alone can't settle, or that need another repo to confirm convergence.

-

---

# Worked examples (real — delete when copying this file for a new repo)

These were extracted live from shallow clones of Envoy and Redis to demonstrate the instrument. They are illustrative, not yet convergence-validated (convergence columns are stubs until ≥3 repos are processed).

### C1. Encode extension ownership structurally, not socially

- Practice: Every optional/extension module must have a named owning sponsor and ≥2 declared reviewers recorded in a machine-checkable owners file — ownership is a build-enforced fact, not a wiki convention. New extensions clear the same quality bar (style, coverage, review) as core.
- Evidence: **Codified** — Envoy `EXTENSION_POLICY.md` ("All extensions … held to the same quality bar as the core"; sponsor + two reviewers "codified in the CODEOWNERS file"); **Verified** — presence of `CODEOWNERS` mapping paths → owners.
- Why / constraint: A large plugin surface with many contributors decays unless responsibility is unambiguous and enforced; sponsorship aligns incentives so extensions aren't dumped in without a maintenance owner.
- Counter-case: Overkill for a small single-team codebase with few extension points — the ceremony costs more than the decay it prevents. Threshold is contributor count × extension-surface size, not project age. (Contrast Redis, which keeps a *small* core deliberately and resists adding extension surface at all — a different answer to the same decay pressure.)
- Convergence: _stub_ — check Kubernetes (`OWNERS` files + KEP sponsorship), Rust (RFC + team ownership). Plausibly ≥3.
- Graduates to: `extensibility-boundaries.md`

### C2. Backpressure via soft limits + watermarks with hysteresis

- Practice: Bound every buffer with a *soft* limit and a high/low watermark pair. Crossing the high watermark fires a callback that signals the source to slow/stop; resume only after draining to the **low** watermark (≈half), not the high one — the gap is deliberate hysteresis to prevent oscillation.
- Evidence: **Documented** — Envoy `source/docs/flow_control.md` (watermark callback chain; "drains (generally to half of the high watermark to avoid thrashing)"); **Inferred→Verified** — `Network::ConnectionImpl` write-buffer + `TcpProxy` `readDisable` coordination named in the doc, confirmable in source.
- Why / constraint: Unbounded buffers convert backpressure into OOM; a single threshold (stop-at-N, resume-at-N) thrashes the source on and off at the boundary. The two-threshold gap trades a little extra buffering for stability.
- Counter-case: Unnecessary where the transport already provides end-to-end flow control you can lean on, or where work is bounded and small. The hysteresis gap costs latency/memory headroom — wrong for ultra-low-latency single-item paths.
- Convergence: _stub_ — TCP itself (window), reactive-streams implementations, Kafka consumer pause/resume. Likely strong (≥3).
- Graduates to: `operational-robustness.md`

### C3. Hold an understandability budget; treat complexity as lock-in

- Practice: Make "one programmer can understand the whole system by reading the source for a bounded time" an explicit, defended project goal. Reject features whose complexity outweighs their value; prefer not creating complexity over cleverly managing it ("solve 95% of the problem with 5% of the code when acceptable").
- Evidence: **Codified** — Redis `MANIFESTO` #6 ("remain understandable … reading the source code for a couple of weeks"; "Complexity is also a form of lock-in"), #10 (opportunistic programming).
- Why / constraint: Code only one author understands can't be safely modified by others regardless of license; comprehensibility is a hard constraint on long-term maintainability, not an aesthetic.
- Counter-case: A bounded "read it all in two weeks" budget doesn't scale to genuinely large-domain systems (a compiler, a kernel) where irreducible essential complexity exceeds any single-reader budget — there the unit of comprehensibility becomes the *module*, not the whole. The principle survives; the scope of "the whole" shrinks.
- Convergence: _stub_ — Go's simplicity ethos, SQLite's single-reader clarity. Check for ≥3.
- Graduates to: `implementation-simplicity.md`

### C4. Don't fake capabilities you can't deliver in all cases — expose the trade-off

- Practice: Refuse to provide an API that *appears* to work uniformly but silently fails outside a happy path. Where a clean abstraction can't hold (e.g. multi-key ops across a partitioned system), expose the seam and the migration/trade-off to the user rather than papering it with magic.
- Evidence: **Codified** — Redis `MANIFESTO` #8 ("We don't want to provide the illusion of something that will work magically when actually it can't in all cases … expose the trade-offs to the user").
- Why / constraint: A leaky abstraction that hides its leaks produces correctness surprises at the worst time; honest seams are debuggable.
- Counter-case: In tension with API ergonomics — for *generic/supporting* surfaces a slightly-leaky convenience layer may be the right call; the honesty rule earns its keep on *core* correctness-sensitive paths. (Note the altitude rhyme: `software-architect` makes the same honesty argument at the consistency-model level — "effectively-once via idempotence," not "exactly-once." Same value, different layer. Keep the craft instance here; route the architecture instance there.)
- Convergence: _stub_ — check distributed KV/DB APIs broadly.
- Graduates to: `implementation-simplicity.md` (with a cross-link noted in `_principles-index.md`)
