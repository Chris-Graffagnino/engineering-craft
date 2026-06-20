# AI Slop — Anti-Pattern Reference (the inversion layer)

**Role:** consumed in **Apply mode** and **Review mode**, specifically when the change under discussion was AI-generated (or AI-assisted) and the question is "where does this characteristically go wrong, and what discipline prevents it." Synthesize from it — do not quote it at the user.

**This is a different *species* of reference, and the difference is load-bearing.** The four dimension references (`testing-correctness.md`, …) are distilled from **gold-standard exemplar repos** with **Codified/Verified** evidence, convergence-tested across ≥3 unrelated codebases. This file is distilled from **cross-industry literature** (peer-reviewed studies, an RCT, practitioner design notes, vendor reports) — a weaker, different evidence basis. **Nothing here is graduated into `_principles-index.md`**, and it must not be: essay-convergence is not repo-convergence. The literature is cited with the same calibrated verbs, but the dominant tiers here are *Measured* (the studies) and *Documented* (the essays), never *Verified-in-an-exemplar-repo*.

**Source:** `_research-ai-slop-corpus.md` (the evidence/provenance layer — the `_extract-*.md` analog for this reference). Every claim below traces to a source there.

---

## Contents
- §1 Thesis — slop is the negative space of P0–P12, not a new failure category
- §2 The dial first — slop tolerance scales with blast radius (P0)
- §3 The inversion map — each slop failure → the principle it violates → the evidence
- §4 The one genuinely new region — human–AI *process* failures (HELD candidate dimension)
- §5 The antidotes — mostly already-validated principles, restated for the authoring loop
- §6 Do-NOT-overclaim ledger (the counter-cases — the most important section)
- §7 Cross-skill links
- §8 Provenance & confidence

---

## §1 Thesis: AI slop is the negative space of the existing corpus

**AI slop is not a new failure category. It is the systematic violation of craft principles this skill has *already* validated (P0–P12) — produced when code is generated faster than it is understood, verified, or fitted to its surroundings.** The corpus distilled what gold-standard code does *right*; slop is the catalog of what generated code does *wrong*, and almost every entry is the inverse of a principle already in the index. That is the central, useful result: **the antidotes are already built.** You don't need new principles to review AI code — you need to recognize which existing principle each slop pattern breaks.

A precise working definition (from the corpus's own popularizers, who locate the defect in *process*, not *provenance*): *AI slop is plausible-on-its-face but unverified code — shipped before it is understood, reviewed, or made to fit — such that its true cost (review, debugging, security, maintenance) is deferred downstream rather than removed.* The failure is shipping-without-vouching, **not** AI-authorship. This definition is what makes §6's counter-case coherent.

The one place the corpus has *no* principle to offer is the **human–AI process** itself (§4) — automation bias, the perception gap, comprehension debt. P0–P12 are all about the *code*; nothing in them is about the *loop that produces it*. That region is the genuine gap, and it is **held**, not graduated, because its witnesses are literature, not exemplar repos.

---

## §2 Set the dial first: slop tolerance scales with blast radius (P0)

**Before treating any AI output as "slop," set P0's dial — because P0 governs slop tolerance exactly as it governs test rigor.** The term's own originators scope zero-review generation to *throwaway* work: Karpathy coined "vibe coding" for "throwaway weekend projects"; Willison's boundary is "I won't commit code I couldn't explain" — but explicitly *exempts* learning exercises and low-stakes prototypes. So:

- *High blast radius / high patch latency* (production, security, billing, anything you can't quickly unwind) → zero-review generation is negligence; full comprehension + verification gate required.
- *Low blast radius / low patch latency* (a spike, a throwaway prototype, a script you'll read once and delete) → vibe-coding is legitimate and fast. Demanding exemplar-repo rigor here is the same cargo-cult error as imposing SQLite's MC/DC standard on a stateless handler.

**Extract the calibration, never the setting.** "All AI code is slop" is as wrong as "all AI code is fine." The defect appears only when zero-review generation is applied *above its blast-radius tolerance*. Pin the dial high for security and money (§3.4); let it ride low for prototypes.

---

## §3 The inversion map: slop failure → principle violated → evidence

Each row is a *finding template* for Review mode: name the slop pattern, cite the principle it breaks (so the finding isn't a bare "you should"), and carry the evidence + boundary. Evidence verbs are calibrated to the **literature**, not to a verified repo.

### §3.1 Violations of the testing dimension (P1–P4, C13)

- **Plausible-but-silently-wrong logic** — compiles, reads correctly, passes a glance, does the wrong thing. *Violates the whole testing composite (P1 independent verification, P3 assert-an-invariant, P4 same-code-in-test-and-prod).* Evidence: Willison, "hallucinations are the least dangerous form" — the dangerous class is the error the compiler *can't* catch (**Documented**); Anthropic's "trust-then-verify gap" (**Codified**, vendor). Boundary: this is the highest-frequency slop class and the one a human reviewer is *worst* at catching on polished code — which is the whole argument for P1's *independent* verification path over re-reading.
- **The testing illusion** — generated tests pass but assert nothing meaningful, or test the implementation rather than the requirement. *This is exactly C13 (mutation testing / "coverage proves a line ran, not that a wrong result would be caught") at scale.* Evidence: Beck names "cheating — disabling or deleting tests to make them pass" as a genuine agent drift signal (**Documented**); "10 failure modes" lists it (**Folklore**). Antidote already in corpus: C13 — mutate the source, confirm a test fails; demand assertions on behavior.
- **Hallucinated APIs / signatures** — calls to methods that don't exist. *Violates P3's oracle leg, but it's the SAFE class:* the compiler/test is the oracle and catches it immediately. Evidence: Willison (**Documented**). Boundary: do not over-weight this — it's loud and self-correcting; spend the review budget on the silent class above.
- **Missing edge cases / happy-path-only** — empty arrays, nulls, Unicode, max-int, boundary conditions. *Violates P3 (systematically inject the failure, then assert the invariant — "the hard part").* Evidence: Osmani's "the last 30%" (**Documented**); Augment's failure catalog (**vendor/Folklore**).

### §3.2 Violations of the simplicity dimension (P5, P6)

- **Comprehension debt** — shipping code no human on the team understands. *This is P5's limit case.* P5: "code only its author understands cannot be safely changed by anyone else; complexity is lock-in." Slop pushes that to *zero* authors. Willison's "don't commit code you couldn't explain" is P5 applied to the authoring loop. Evidence: Willison (**Documented**), converged across Ptacek/Osmani/Böckeler/Beck. Boundary: P5's dial still applies — "understand it" means the *module*, not every transitive line, at kernel/compiler scale.
- **Over-engineering / verbose boilerplate** — needless abstraction, defensive code, ceremony. *Violates P5 ("the best way to fight complexity is not creating it at all").* Evidence: Böckeler memo #13 names "overly complex or verbose code" and "verbose and redundant tests" as the *most insidious, long-term* impact (**Documented**); Anthropic warns naive adversarial-review *induces* over-engineering — tests for impossible cases (**Codified**, vendor). Note the two-sided trap: under-review ships slop, over-eager review ships gold-plating.
- **Hallucinated / nonexistent dependencies (slopsquatting)** — the model recommends a package that does not exist; an attacker pre-registers the predictable name. *This is P6's limit case:* P6 says "every dependency is code you don't understand and a supply-chain surface — vet it." Slop adds: the dependency may not exist at all. Evidence: Spracklen et al., USENIX Security 2025 — **19.7%** of recommended packages hallucinated, **43%** recur across all 10 reruns (so they're predictable, hence weaponizable) (**Measured**, peer-reviewed). Antidote already in corpus: P6 + an existence/lockfile/hash gate.

### §3.3 Violations of the extensibility dimension (P10; + architectural seam)

- **Convention / pattern drift** — the model applies generic training-data patterns instead of the codebase's established ones; domain vocabulary degrades toward boilerplate. *Violates P10's spirit (extend through the existing narrow contract; don't reach past it) and P5.* Evidence: "Weather Report" root-cause "agents optimize for make-it-work, conventions lose by default" (**Documented**); "10 failure modes" architecture-drift (**Folklore**). Seam: when the drift is genuinely *architectural* (wrong service boundary, wrong consistency model), it is no longer code-level slop → **hand off to `software-architect`** (§7).
- **Code duplication / declining DRY** — copy-paste instead of abstraction; refactoring collapses. *Violates the reuse discipline underlying the extensibility/simplicity boundary.* Evidence: GitClear — copy-pasted lines rose 8.3%→12.3% and *exceeded refactored ("moved") lines for the first time* in 2024 across 211M lines (**Measured but correlational**, vendor — see §6). Boundary: this is the most-contested claim in the corpus (the Echoes-of-AI controlled study found no such degradation under review discipline — §6); present it as *real-but-conditional*, not inevitable.

### §3.4 Violations of the robustness dimension (P0 pinned high, P7–P9) — the security cluster

**Pin P0's dial to maximum here** — security and money are maximal blast radius. The convergence on insecurity is the strongest in the corpus (academic + vendor agreeing):

- **Insecure-by-default code** — injection, broken auth, info leaks; secure-by-default skipped "because it's more work." *Violates operational-robustness + P0.* Evidence (converged): NYU S&P 2022 — ~**40%** of Copilot completions vulnerable across 89 CWE scenarios (**Measured**, peer-reviewed); Stanford CCS 2023 — AI users wrote *less* secure code (SQLi 36% vs 7%) *and believed it more secure* (**Measured**, peer-reviewed); Veracode 2025 — **45%** failed security tests, flat across model sizes (**vendor**); Apiiro — privilege-escalation paths +322% (**vendor**). Boundary: the two peer-reviewed studies are on *old* models (2021–2022) — cite the *failure class* as durable, not the *specific percentage* as current (§6).
- **Hardcoded / fallback secrets** (`|| "changeme"`), **secrets via public env prefixes** (`NEXT_PUBLIC_*`), **insecure access defaults** (RLS off / `USING (true)`, public buckets), **skipped integrity checks** (unverified webhooks). *Each violates operational-robustness; each is a named, code-level red-flag.* Evidence: "Weather Report" with concrete code examples (**Documented**), GitGuardian (**vendor**). These are the highest-precision Review-mode checklist items — they're greppable.
- **Weak error handling / happy-path-only** under failure. *Violates P3 (test failure paths) and P7–P9 (recovery).* Evidence: Osmani (**Documented**), Augment (**vendor/Folklore**).
- **Performance anti-patterns** (O(n²), allocation-in-loop) that only bite under production load. *Violates operational-robustness (resource control).* Evidence: Augment (**vendor/Folklore**). Boundary: low-confidence tier; treat as a prompt to profile, not a verdict.
- **Prompt-injection / "rules-file backdoor"** — malicious instructions hidden in config/guardrail files; *the guardrail file is itself an attack surface.* Evidence: GitGuardian, CVE-2025-53773 (**vendor**, real CVE). Worth flagging because it inverts the naive assumption that the AGENTS.md/rules layer is trusted.

---

## §4 The one genuinely new region: human–AI *process* failures (HELD — candidate dimension)

These have **no P-home**, because P0–P12 are about *code* and these are about the *loop that produces it*. This is the real gap the AI-slop literature exposes. It is **held as a candidate dimension, not graduated** — its witnesses are studies and essays, not exemplar repos (see §8 for what would graduate it).

- **The perception gap** — developers feel faster while being slower / less safe. Evidence: METR RCT — experienced devs **19% slower** while predicting +24% faster (**Measured**, the only true RCT in the corpus); Stanford's overconfidence finding (**Measured**); DORA — personal-productivity-up but delivery-stability-down (**Measured**, survey). This is the single most decision-relevant finding: *self-reported speedup is not evidence of speedup.*
- **Automation bias / sunk-cost / anchoring** — over-trusting polished output; fixing a bad suggestion for 20 min instead of writing it fresh in 5; the first suggestion blocking better alternatives. Evidence: Böckeler memo #4 names all three biases (**Documented**); "10 failure modes" confidence-delusion (**Folklore**).
- **Comprehension debt as a workflow state** (distinct from §3.2's code property) — the *human* never paid the understanding cost. Evidence: converged across the practitioner essays (**Documented**).
- **Context-window amnesia / agent thrashing** — long sessions forget early decisions; each correction spawns a new error. Evidence: Anthropic best-practices names "kitchen-sink session" and "correcting over and over" (**Codified**, vendor); Thoughtworks "agent thrashing" (**Documented**).
- **Productivity theater** — output and PR size balloon while delivered value stays flat. Evidence: Apiiro PR-size inflation (**vendor**); DORA stability drop (**Measured**).

**The process-level antidotes** (also literature-sourced, not graduated): the "2-correction rule" (stop and re-prompt after two failed fixes); `/clear` between tasks; "keep the engineering brain on — automate only what you've done by hand several times and verified beats baseline" (Ronacher); **measure outcomes (bugs shipped, time-to-resolution, duplication ratio), not output (LOC, commits)**.

---

## §5 The antidotes — already-validated principles, restated for the authoring loop

The corpus's most useful contribution to AI-code review is that **the defenses already exist and are convergence-validated.** Restated for the AI loop, with their anchor:

1. **Comprehension is the merge gate** — don't commit what you can't explain. *= P5 applied to authoring.* (Willison, Ptacek, Osmani, Böckeler, Beck — 5 literature witnesses.)
2. **Verification-as-guardrail** — give the agent a check it must pass (tests, build exit code, linter, screenshot diff); "the single highest-leverage thing you can do." *= P1–P4 / the §2 testing composite applied to the loop.* (Anthropic, Ptacek, Beck, Thoughtworks, Willison.)
3. **"If you haven't seen it run, it's not working."** *= P4 (same code in test and prod) + manual-QA discipline.* (Willison, Osmani.)
4. **Meaningful tests, not vacuous ones** — assert on behavior; watch for test-deletion "cheating." *= C13 (mutation testing).* (Beck, Osmani, Böckeler.)
5. **Verify every dependency exists / pin it** — existence check, lockfile, hash, lower temperature. *= P6 + slopsquatting gate.* (USENIX, Socket, BleepingComputer.)
6. **Calibrate review skepticism to margin of error** — maximal scrutiny for security/auth/payments/architecture. *= P0, the dial, applied to review intensity.* (Böckeler, Osmani.)
7. **The "PR contract"** — every PR ships evidence: what/why, proof-it-works, risk tier, which parts are AI-generated, the 1–2 areas needing human focus. *(New process practice — Osmani; no P-anchor, lives with §4.)*
8. **Small, reviewable diffs.** *(Reviewability — process; Osmani, Böckeler.)*

When advising on AI code, **lead with the anchor principle, not the blog**: "this breaks P5 (complexity is lock-in)" is a corpus-grounded finding; "a blog said don't do this" is folklore.

---

## §6 Do-NOT-overclaim ledger (the counter-cases — the product)

The boundaries are the product. This section is the most important in the file, because the AI-slop discourse is saturated with motivated reasoning (vendors selling fear; skeptics and boosters talking past each other).

- **THE headline counter-case — slop is a *process* failure, not an inherent property of AI code.** Borg et al., "Echoes of AI" (EMSE 2025) — a controlled experiment (n=151, mostly professionals) found **no significant maintainability degradation** when later developers evolved AI-written code; CodeHealth was slightly *higher* for habitual-AI-user code (**Measured**). This *flips* the GitClear decay narrative from "inherent" to "conditional on review discipline." **Do NOT claim "AI code is inherently unmaintainable."** Claim: *unreviewed, unverified, unexplained AI code degrades maintainability; disciplined AI code need not.* This is congruent with §1's definition.
- **Vendor conflict-of-interest.** GitClear sells code analytics; Veracode / Apiiro / GitGuardian / Socket sell security. All have a commercial stake in the "AI is risky / measure your code" narrative. Treat their numbers as *corroborating signal*, never as foundational science. The load-bearing evidence is the peer-reviewed + RCT tier (USENIX, NYU, Stanford, METR, Echoes-of-AI).
- **Correlational, not causal.** GitClear and DORA show *timing correlation* (metrics worsened as AI adoption rose), not attribution. Confounders (hiring, market, deadline pressure) are uncontrolled. Only METR and Echoes-of-AI are controlled.
- **Model staleness — cite the failure *class*, not the *number*.** NYU (2021 Copilot), Stanford (2022 Codex), USENIX (2024 models). A 2026 follow-up reports frontier package-hallucination rates compressed to ~4.6–6.1% (down from 19.7%). **The absolute rates are dropping; the failure *classes* persist.** Quoting "40% of AI code is vulnerable" as a 2026 fact is dishonest; "AI code reproduces known vulnerability classes at a material rate" is defensible.
- **The P0 boundary (don't moralize against all generation).** Vibe-coding a throwaway prototype is legitimate and the term's originators say so. Slop is zero-review generation applied *above its blast-radius tolerance* — not generation per se.
- **Essay-convergence ≠ repo-convergence.** Five essayists agreeing is real signal but is **not** the skill's graduation standard (≥3 unrelated exemplar *codebases*, Codified/Verified). Nothing in this file is promoted into `_principles-index.md`. The antidotes that *are* in the index (P5, P6, P1–P4, C13, P0) earned their place by repo-convergence independently; this file merely *points* AI-code review at them.

---

## §7 Cross-skill links

- **Code-level slop lives here.** Hallucinated logic, insecure defaults, duplication, vacuous tests, convention drift *within* an established design.
- **Architectural slop → `software-architect`.** When AI proposes the wrong *service boundary*, a naive distributed-transaction/saga, or an unsound consistency model, that is not code-craft slop — it's an architecture error. Name the seam and hand off, exactly as the dimension references do.
- **The `software-architect` epistemic backbone applies directly** — "what would flip this," least-worst framing, evidence calibration. The AI-slop discourse needs that backbone more than most, because it is unusually polarized (§6).

---

## §8 Provenance & confidence

- **Evidence basis (read this honestly):** this reference is sourced from cross-industry *literature*, not exemplar repos. Tiers, in descending load-bearing order: **Measured** peer-reviewed/RCT (USENIX 2025, NYU S&P 2022, Stanford CCS 2023, METR 2025, Echoes-of-AI 2025); **Measured** large-but-correlational/vendor (GitClear, DORA, Veracode, Apiiro); **Documented** practitioner design notes (Willison, Osmani, Ptacek, Ronacher, Böckeler, Beck, Fowler); **Folklore** listicles (used only for structure, never as a load-bearing claim). Full provenance: `_research-ai-slop-corpus.md`.
- **Nothing here is graduated.** The §3 mappings *reuse* already-graduated principles (P0, P1–P4, P5, P6, P10, C13); the §4 process cluster is a **held candidate dimension**.
- **What would actually graduate the §4 process dimension — by this skill's real rules.** Not more essays. The exemplar-repo analog is **≥3 unrelated organizations with *codified and CI-enforced* AI-coding policies** (Tier-1 codified + Tier-3 verified-in-CI) — e.g., an AGENTS.md/CLAUDE.md that is *actually* gate-enforced, a verified mandatory-review pipeline, a measured outcome-tracking practice. Anthropic's best-practices doc is one **Codified** (vendor) witness; a distilled Shopify / Google / GitHub *enforced* policy would be others. Distill those the way Distill mode distills a repo, and the process dimension graduates legitimately. Until then it is held — honestly.
- **Confidence:** the §3.4 security cluster and §4 perception-gap are the best-evidenced (multiple Measured witnesses). The §3.2 duplication claim is *contested* (Echoes-of-AI counter-case) and must be delivered as conditional. The performance/perf-anti-pattern and Folklore-tier items are weakest — prompts to investigate, not verdicts.
- **Maintenance:** when a new controlled study or a distilled enforced-policy "repo" lands, update `_research-ai-slop-corpus.md` first (the evidence layer), then revise here. Watch for the absolute-rate decay (model staleness, §6) — refresh the numbers, keep the classes.
