# Review-Mode Playbook — the operational layer

**Role:** loaded in **Review mode**. This is the *how-to-run-the-review* companion to the four knowledge dimensions: they hold **what** the craft practices are; this holds **how to conduct the review** — the lenses to look through, the cheap tripwires that surface candidates, and the output triage. Synthesize from it — do not quote it at the user.

**Provenance:** the three lenses in §1 and the tripwires in §2 were adapted from Cursor's `thermo-nuclear-code-quality-review` skill — a sharp, operational review *prompt*: assertion- and heuristic-driven, rather than grounded in exemplar evidence or paired with explicit boundaries. They are **re-grounded** here in this corpus's already-graduated principles, each carrying the boundary condition the prompt leaves implicit. **Nothing here is a new principle**; it is the operational application of P0, P5, P1–P4, P6, P10/P2. What this corpus adds to the source's operational sharpness is the *grounding* and the *verification gate*.

---

## Contents
- §1 The three review lenses (general — every diff)
- §2 Cheap tripwires (signals that trigger the principled question — never verdicts)
- §3 AI-generated diffs — the specialization
- §4 What this playbook is NOT
- §5 Provenance & confidence

---

## §1 The three review lenses (general — apply to every diff)

Look through these *before* the per-dimension pass. They shape **which** findings matter and **how** they're reported.

### Lens A — Code judo: prefer *deleting* complexity over *relocating* it

**The move.** For each meaningful change, ask: does this reduce the number of concepts a reader must hold, or just spread the same complexity around? Hunt for the reframing that makes whole branches / helpers / modes / layers **disappear** — not the refactor that merely centralizes them. Prefer the version that "makes the code feel inevitable in hindsight."

- **Anchor:** P5 (understandability budget; *"the best way to fight complexity is by not creating it at all"*) and C43 (avoid gratuitous layers). This is P5 turned into an *active* review move — the corpus had the principle but not the operational stance.
- **THE GATE (the boundary the source omits):** ambitious deletion is only *responsible* behind a **behavior-preservation check** — P1–P4 / `testing-correctness.md`. "Delete a whole layer, behavior preserved" is an unverified claim until a characterization test or the existing suite proves it. Deletion-bias *without* that gate is exactly the `ai-slop-antipatterns.md` §3.1 **plausible-but-silently-wrong** pattern — the most dangerous slop class, because it looks clean and passes a glance. **Never push a structural deletion without naming the test that pins the behavior.**
- **The counter-boundary (don't over-apply):** some indirection is a deliberate **seam**, not waste. A "thin wrapper" can be an injection seam (P2) or a narrow versioned contract (P10) that absorbs future change — deleting it trades a little indirection now for a refactor-the-callers cost later. C43 is *capped* in this corpus precisely because SQLite is an inherently-layered SQL engine. So: flag wrappers that buy *nothing*; protect wrappers that buy *a swap point or a stable contract*. The test is "what variation does this seam absorb?" — if "none, today or plausibly," delete it; if "a real axis of change," keep it.
- **Finding form:** *"This refactor relocates the branching but the concept-count is unchanged; P5 favors the reframing where these branches disappear — [sketch]. Gate: confirm `<test>` still passes / add a characterization test first. Flips if this indirection is a deliberate seam for `<variation>` (P10)."*

### Lens B — Blast radius on the neighbors: judge what the diff does to the code *around* it

**The move.** The source's sharpest unique angle: don't only ask "is this change good?" — ask "does this otherwise-working change make the surrounding code harder to reason about?" Be suspicious of new ad-hoc conditionals bolted into unrelated flows, special cases inserted into a busy function, "temporary" branching that will calcify into permanent debt.

- **Anchor:** P5 applied to the *surroundings* (complexity is lock-in for the whole module, not just the new lines), plus P10 (feature logic leaking across a boundary into a shared path). An honest note: there is **no graduated principle** that says "don't degrade neighbors" — this is an *application* of P5/P10, not a new law. Treat it as a high-value lens, not a citable principle.
- **Boundary:** entropy is sometimes the right local trade — a genuinely one-off special case in a leaf function is cheaper than a new abstraction (the YAGNI direction). The smell is special-casing in a **shared / busy / high-traffic** flow, where the reasoning cost compounds. Calibrate by how many readers cross this path.
- **Cross-skill:** when the "neighbor" damage is actually an architecture-boundary leak (feature logic colonizing a shared service), name the seam and hand to `software-architect`.

### Lens C — Triage: a few high-conviction structural findings, ranked — not a flood of nits

**The move.** Lead with the largest structural issues; suppress cosmetic notes when bigger problems exist. Prefer a small number of high-conviction comments over a long list. Report order:

1. Structural / maintainability regressions (the change makes the codebase messier)
2. Missed dramatic simplification (a visible code-judo move — Lens A)
3. Spaghetti / branching-complexity growth (Lens B)
4. Boundary / abstraction / type-contract problems that obscure the real invariant
5. File-size / decomposition concerns (§2 tripwire)
6. Modularity & legibility

- **Anchor:** the skill's evidence-calibration backbone (don't present *inferred* as *verified*) and Apply-mode's "ordered by how strongly each practice applies." High-conviction = a finding you can pin to a principle with a clear boundary; if you can only reach *inferred*, rank it low or hold it.
- **Boundary:** "few findings" is not "skip the security pass." The §3.4 security tripwires in `ai-slop-antipatterns.md` are *never* nits — a hardcoded secret outranks any structural elegance point. Triage suppresses *cosmetic* noise, not *high-blast-radius* defects (P0).

---

## §2 Cheap tripwires — signals that *trigger* the principled question, never verdicts

These are greppable/observable hooks. Their job is to **cheaply surface a candidate** for the expensive per-dimension analysis — they are not findings by themselves. This is the layer Review mode was missing: principle-rich, trigger-poor.

| Tripwire (cheap to detect) | The question it triggers | Principle to then apply | Calibration / counter-case (why it's not a verdict) |
|---|---|---|---|
| A file crosses a large size threshold **because of this diff** (e.g. a hand-edited source file ballooning) | "Should this be decomposed *first*?" | P5 / C43 | **Size is a smell, not a defect.** SQLite ships a ~200k-line single-file amalgamation (`mksqlite3c.tcl`) *on purpose*. The threshold is a prompt only where **a human hand-edits the file**; exempt generated, vendored, lookup-table, or amalgamated files. Tune the number to the codebase — never a universal constant (P0: "a number is not a score"). |
| New conditionals / special cases inserted into an **unrelated or busy** flow | "Is a model or helper missing here?" | P5, P10, Lens B | A genuine one-off in a leaf path may be cheaper than a new abstraction (YAGNI). Smell scales with how shared the path is. |
| `cast` / `any` / `unknown` / new optional params appear in the diff | "Is an invariant being papered over instead of made explicit?" | testing-correctness (invariants), P10 (explicit contract) | Boundaries to genuinely-dynamic external data legitimately use `unknown` + validation; flag the *silent fallback*, not the honest parse-and-validate. |
| Copy-pasted block instead of an extracted helper | "Reuse a canonical helper, or extract one?" | P5/P6 (duplication ↔ understandability) | "A little copying is better than a little dependency" (Go) — two copies can beat a premature wrong abstraction. Smell grows at the 3rd+ copy or when the copies must stay in lockstep. |
| "Temporary" / feature-flag branching with no removal trigger | "What deletes this, and when?" | P12 (governed lifecycle), Lens B | Real staged rollout *should* be flagged behind an opt-in (P12) — the smell is the flag with **no owner and no removal condition**, not flags per se. |
| Independent work serialized for no reason / related updates left half-applicable | "Cleaner as parallel / more atomic?" | operational-robustness | Don't over-index on micro-optimization; flag only when the simpler structure is *obvious* and the current one is brittle. Often an architecture call → `software-architect`. |

**The rule for the whole table:** the tripwire fires the *question*; the *principle* (with its boundary) delivers the *verdict*. Jumping straight from tripwire to verdict is the gap this layer closes — it's how a flat "1000-line rule" would flag SQLite's signature artifact.

---

## §3 AI-generated diffs — the specialization

When the change under review is AI-generated or AI-assisted, **also load `ai-slop-antipatterns.md`** and run its §3 inversion map as additional tripwires. The highest-precision (greppable) ones:

- `NEXT_PUBLIC_*` / `VITE_*` wrapping a secret; `process.env.X || "changeme"` fallback secrets; service-role keys in client code → §3.4 (robustness, P0 high).
- RLS disabled / `USING (true)`; storage buckets flipped public; webhook handlers with no signature check → §3.4.
- Tests that pass but assert nothing meaningful → §3.1 (this is your C13, mutation-testing, at scale).
- An imported package you can't immediately place → §3.2 slopsquatting (verify it *exists*; P6).
- Convention drift — generic patterns where the codebase has an established one → §3.3 (P10).

**Lens A matters more here, not less.** AI excels at producing a confident "simplified" rewrite that silently changes behavior. Every code-judo deletion on an AI diff must name its behavior-preservation test — the gate is the whole point.

---

## §4 What this playbook is NOT

- **Not a merge gate.** Engineering-craft is an advisory *craft lens*; approve/reject and bug-hunting belong to `/code-review`. Surface "presumptive craft concerns," never a block verdict. (This is the deliberate non-adoption of the source skill's hard Approval Bar — wrong altitude for this skill.)
- **Not a license to flag deliberate design as defect.** Every lens and tripwire above carries a boundary; a finding that ignores its boundary is the dogma this skill's rule 4 exists to prevent.
- **Not architecture review.** When a finding is really about service boundaries, consistency models, or distributed-transaction shape, name the seam and hand to `software-architect`.
- **Not canned phrasing.** Synthesize findings in the corpus form ("change does X → Pn says Y → flips if Z"); don't parrot fixed one-liners.

---

## §5 Provenance & confidence

- **Source:** lenses + tripwires adapted from Cursor's `thermo-nuclear-code-quality-review` — a sharp operational review prompt — and re-grounded in P0/P5/P6/P10/P2/P1–P4 + C43/C13 with the boundary conditions the prompt leaves implicit. The adaptation is the point: the source supplied operational *sharpness*; this corpus supplies the *grounding and the gate*.
- **Status:** operational layer, **nothing graduated** — it cites existing principles, it does not add any. Lens B ("don't degrade neighbors") is explicitly an *application* of P5/P10, not a citable law.
- **Confidence:** Lens A and the §2 tripwires are well-anchored (P5/C43, P0). Lens B is a useful heuristic with no dedicated principle — present its findings as *inferred* unless they land on P5/P10 cleanly. The §3 security items are the highest-confidence, highest-priority of all (multiple Measured witnesses in `_research-ai-slop-corpus.md`).
- **Maintenance:** if a future exemplar graduates a "don't degrade the surroundings" principle, promote Lens B from heuristic to cited law and update here.
