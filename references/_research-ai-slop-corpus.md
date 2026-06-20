# Research Corpus: "AI Slop" — failure modes of AI-generated code

**Purpose.** Raw, evidence-calibrated source material for extending `engineering-craft` to cover the anti-patterns of AI-generated code ("AI slop"). This is a *research-gathering* artifact (input), not a distilled dimension reference (output). It feeds Distill-style processing the same way an `_extract-<repo>.md` file does — except the "repo" here is the cross-industry literature on where AI-assisted code goes wrong.

**Method note.** Sources gathered via live web research (≈40 read; the strongest ~30 retained). Each is tagged with this skill's evidence-calibration verbs. The dominant failure mode of *this* corpus is vendor conflict-of-interest (security vendors and code-analytics vendors both have a commercial stake in the "AI is risky / measure your code" narrative) — flagged inline. The load-bearing evidence is the peer-reviewed + RCT tier; vendor data is corroborating, not foundational.

---

## 1. The definition (what "slop" precisely means for code)

The term's own popularizers draw the line not at *provenance* but at *process*: slop is unreviewed, unsolicited output shipped without the author being able to vouch for it.

- **"Slop is the new name for unwanted AI-generated content"** — Simon Willison, May 8 2024. <https://simonwillison.net/2024/May/8/slop/> — *Reference/definition.* Slop = AI output "mindlessly generated and thrust upon someone who didn't ask for it." The defect is publishing-without-review, not AI-authorship. (Coinage credited to @deepfates.)
- **"Not all AI-assisted programming is vibe coding"** — Simon Willison, Mar 19 2025. <https://simonwillison.net/2025/Mar/19/vibe-coding/> — *Documented (practitioner).* The cleanest boundary line in the corpus: *"I won't commit any code to my repository if I couldn't explain exactly what it does."* If you reviewed, tested, and can explain it, "that's not vibe coding, it's software development." Names unsafe domains for vibe-coding: production, security-sensitive, billing.
- **"Vibe coding" (origin)** — Andrej Karpathy, Feb 2 2025. <https://x.com/karpathy/status/1886192184808149383> — *Folklore (primary coinage).* "You fully give in to the vibes… I 'Accept All' always, I don't read the diffs anymore." The behavior checklist that *produces* slop, stated by the term's originator; explicitly scoped to "throwaway weekend projects."
- **"cURL's Daniel Stenberg: AI slop is DDoSing open source"** — The New Stack, Feb 15 2026. <https://thenewstack.io/curls-daniel-stenberg-ai-is-ddosing-open-source-and-fixing-its-bugs/> — *Documented.* The definitive code-domain definition: slop is "long, confident, and often completely fabricated" output that is *harder* to filter than spam precisely because it mimics technical rigor (real-looking functions, fake GDB sessions). cURL ended its bug-bounty program; accuracy of reports fell to ~1-in-20.
- **"AI isn't useless. But is it worth it?"** — Molly White, Apr 17 2024. <https://www.citationneeded.news/ai-isnt-useless/> — *Documented.* Names the quality boundary: AI produces "plausible but completely non-functional code" and "is not going to architect a project for you."

> **Working definition for the skill:** *AI slop is code that is plausible-on-its-face but unverified — generated faster than it can be understood, reviewed, or made to fit — such that its true cost (review, debugging, security, maintenance) is deferred downstream rather than removed.*

---

## 2. Measured evidence (the load-bearing tier)

This is the tier that matters most for a skill that prizes *Measured* over *Folklore*. Peer-reviewed + RCT first; large-but-correlational and vendor-measured after.

### Peer-reviewed / controlled

| Finding | Source | Calibration | The number |
|---|---|---|---|
| LLMs recommend non-existent packages | Spracklen et al., **USENIX Security 2025** — <https://arxiv.org/abs/2406.10279> | **Measured** (576k samples, 16 models) | **19.7%** of recommended packages hallucinated; 205,474 unique fake names; **43%** recur across all 10 reruns → predictable & weaponizable ("slopsquatting") |
| AI makes experienced devs *slower* on familiar repos | **METR RCT**, Jul 2025 — <https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/> | **Measured** (RCT, 16 devs, 246 real tasks) | Predicted **+24%** speedup; *actual* **−19%** (slower). Perception gap persisted post-hoc (~+20% believed) |
| Copilot reproduces CWE-Top-25 vulns | Pearce et al. (NYU), **IEEE S&P 2022** — <https://arxiv.org/abs/2108.09293> | **Measured** (1,689 programs, 89 scenarios) | **~40%** of completions vulnerable |
| AI assistants → less secure code *and* false confidence | Perry et al. (Stanford), **ACM CCS 2023** — <https://arxiv.org/html/2211.03622v3> | **Measured** (controlled, n=47) | SQLi task **36%** vulnerable (AI) vs **7%** (control); AI users *believed* their code more secure |
| **Counter-case:** no maintainability degradation | Borg et al., **"Echoes of AI," EMSE 2025** — <https://arxiv.org/abs/2507.00788> | **Measured** (controlled, n=151, 95% pros) | Phase-2 devs evolving AI-written code saw **no significant** maintainability loss; CodeHealth slightly *higher* for habitual-user code |

### Large-scale but correlational / survey

- **GitClear "AI Copilot Code Quality" (2024 & 2025)** — <https://www.gitclear.com/ai_assistant_code_quality_2025_research> — *Measured but correlational; vendor.* 211M changed lines, 2020–2024. Copy/pasted lines 8.3%→12.3%; "moved" (refactored) lines ~24%→~9.5% — copy-paste **exceeded refactoring for the first time** in 2024. Short-window churn ~doubled (3.1%→5.7%). ~**8× / "4×"** rise in duplicated 5+-line blocks. *Caveat: timing-correlational, copy-paste inferred from diffs, vendor sells code analytics.*
- **DORA / Accelerate State of DevOps 2024** — <https://dora.dev/research/2024/dora-report/> — *Measured (survey), correlational.* A 25% rise in AI adoption associated with est. **−7.2% delivery stability** and **−1.5% throughput**, despite ~75% reporting personal productivity gains ("vacuum hypothesis"). *Self-reported; headline %s via RedMonk's analysis of the report.*

### Vendor-measured (corroborating; treat as signal, not science)

- **Veracode 2025 GenAI Code Security Report** — <https://www.veracode.com/blog/genai-code-security-report/> — **45%** of generated samples failed security tests (Java **72%**, XSS unmitigated **86%**); security "flat regardless of model size." *Vendor (AppSec); synthetic prompt battery.*
- **Apiiro, "4× Velocity, 10× Vulnerabilities"** Sep 2025 — <https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/> — Shallow errors *down* (syntax −76%, logic bugs −60%) but **privilege-escalation paths +322%, architectural design flaws +153%**; larger PRs reduce reviewability. *Vendor; proprietary, non-reproducible method.*

---

## 3. Practitioner design notes (the credible middle ground)

Respected engineers, neither hype nor reflexive dismissal. Highest value: each names failure modes *and* the craft antidote.

- **Addy Osmani, "The 70% Problem"** Dec 2024 — <https://addyo.substack.com/p/the-70-problem-hard-truths-about> — *Documented.* The vocabulary the whole debate borrows: **"house of cards code,"** the **70/30 split** (AI nails the first 70%; the last 30% — edge cases, security, integration — is where it collapses), the **"knowledge paradox"** (AI accelerates seniors, misleads juniors), the **"two steps back"** debugging loop.
- **Addy Osmani, "Code Review in the Age of AI"** Jan 2026 — <https://addyo.substack.com/p/code-review-in-the-age-of-ai> — *Documented.* The bottleneck moved from *writing* to *proving*. Proposes the **"PR contract"**: every PR declares what/why, proof-it-works (tests/logs), risk tier + which parts are AI-generated, and the 1–2 areas needing human focus. Maxim: **"proof over vibes."**
- **Thomas Ptacek (fly.io), "My AI Skeptic Friends Are All Nuts"** Jun 2025 — <https://fly.io/blog/youre-all-nuts/> — *Opinion (pro-AI, craft-aware).* The enthusiast still concedes the craft: **"LLMs only produce shitty code if you let them"** — read every line, you own the merge, and agents must lint/compile/test so invented signatures surface as errors.
- **Armin Ronacher, "Agentic Coding Recommendations"** Jun 2025 — <https://lucumr.pocoo.org/2025/06/12/agentic-coding/> — *Documented (experience).* Leverage is in shaping the *environment*: prefer stable low-churn ecosystems, "write the dumbest possible thing that will work," make tools "protected against an LLM chaos monkey," log to files so the agent self-diagnoses.
- **Armin Ronacher, "Agentic Coding Things That Didn't Work"** Jul 2025 — <https://lucumr.pocoo.org/2025/7/30/things-that-didnt-work/> — *Documented.* "Keep the brain on" — automation tempts mental disengagement, "and that's where quality drops." Only automate what you've done by hand several times and empirically verified beats baseline.
- **Birgitta Böckeler / Martin Fowler, "Exploring Gen AI" series** (2023–2025) — <https://martinfowler.com/articles/exploring-gen-ai.html> — *Documented.* Names **amplification of bad practice** ("injects anything that seems relevant"), **context-poisoning** (keeps suggesting outdated patterns post-refactor), and the review-complacency biases (automation bias, sunk-cost, anchoring). Memo #13: the *most insidious* impact is long-term — "verbose and redundant tests," "lack of reuse," "overly complex or verbose code." Antidote: calibrate skepticism to *margin of error*.
- **Kent Beck, "Augmented Coding: Beyond the Vibes"** Jun 2025 — <https://newsletter.kentbeck.com/p/augmented-coding-beyond-the-vibes> — *Documented (experience).* The AI is an "unpredictable genie"; watch for three drift signals — **unnecessary loops, unrequested functionality, and cheating (disabling/deleting tests to make them pass)** — and use **TDD as the leash**.
- **Simon Willison, "Hallucinations in code are the least dangerous…"** Mar 2025 — <https://simonwillison.net/2025/Mar/2/hallucinations-in-code/> — *Documented.* Crucial reframing: hallucinated APIs are the *safe* class (compiler catches them); the dangerous class is code that **runs but is silently wrong**. *"Never trust any code until you've seen it work with your own eyes."*
- **Martin Fowler, "LLMs bring a new nature of abstraction"** Jun 2025 — <https://martinfowler.com/articles/2025-nature-abstraction.html> — *Opinion.* The new wrinkle vs. past abstraction jumps: **non-determinism** — "I can't just store my prompts in git and know I'll get the same behavior."

---

## 4. Anti-pattern & remediation taxonomy (concrete, code-level)

- **Augment Code, "Debugging AI-Generated Code: 8 Failure Patterns & Fixes"** — <https://www.augmentcode.com/guides/debugging-ai-generated-code-8-failure-patterns-and-fixes> — cleanest 1:1 anti-pattern→fix mapping.
- **"7 failure modes every AI coding platform bakes in"** — Ilya Kabanov — <https://theweatherreport.ai/posts/vibe-coding-anti-patterns/> — best *named, code-level security* anti-patterns with the exact friction the model is avoiding (service-role keys client-side; `NEXT_PUBLIC_*` leaks; RLS `USING (true)`; unverified webhooks; `|| "changeme"` fallback secrets).
- **GitGuardian, "Crappy Code, Crappy Copilot?"** — <https://blog.gitguardian.com/crappy-code-crappy-copilot/> — hardcoded secrets; the novel **"rules-file backdoor"** (a guardrail/config file is itself an attack surface; CVE-2025-53773).
- **Socket / BleepingComputer on slopsquatting** — <https://socket.dev/blog/slopsquatting-how-ai-hallucinations-are-fueling-a-new-class-of-supply-chain-attacks> · <https://www.bleepingcomputer.com/news/security/ai-hallucinated-code-dependencies-become-new-supply-chain-risk/> — remediation checklist: verify every package, lockfiles + hash pinning, lower temperature.
- **Anthropic, "Best Practices for Claude Code"** — <https://code.claude.com/docs/en/best-practices> — *Codified (vendor, official).* **"Give Claude a way to verify its work" = "the single highest-leverage thing you can do."** Names the over-engineering risk of naive adversarial review (extra abstraction, tests for impossible cases).
- **Shrivu Shankar, "How I Use Every Claude Code Feature"** — <https://blog.sshh.io/p/how-i-use-every-claude-code-feature> — team-scale: commit-time test-gate hook; the **"audit logs → refine guardrails"** flywheel.
- **Sunit Parekh (Thoughtworks), "Beyond Vibe Coding"** — <https://www.thoughtworks.com/insights/blog/generative-ai/beyond-vibe-coding-the-five-building-blocks-of-aI-native-engineering> — names **"agent thrashing"**; spec→test→review pipeline.
- **Listicle taxonomies (lower evidence, useful structure):** "The 10 Failure Modes of AI-Assisted Coding" <https://codewithrigor.com/blog/the-10-failure-modes/> (good on *process* failures: comprehension debt, automation bias, testing illusion, productivity theater; the **"2-correction rule"**); LeadDev "How AI generated code compounds technical debt" <https://leaddev.com/technical-direction/how-ai-generated-code-accelerates-technical-debt>.

### Master anti-pattern list (deduplicated, mapped to this skill's dimensions)

| # | Anti-pattern | Violates dimension | Best evidence |
|---|---|---|---|
| 1 | Package hallucination / **slopsquatting** | operational-robustness (supply chain) | USENIX 2025 (Measured) |
| 2 | Hallucinated APIs/signatures | testing-correctness | Willison; Augment |
| 3 | **Plausible-but-silently-wrong** logic | testing-correctness | Willison; Anthropic "trust-then-verify gap" |
| 4 | Code duplication / clone proliferation | extensibility-boundaries; simplicity | GitClear (Measured); LeadDev |
| 5 | Declining DRY / declining refactoring | extensibility-boundaries | GitClear (Measured) |
| 6 | Outdated/deprecated APIs (reintroduce CVEs) | operational-robustness | Augment; Weather Report |
| 7 | Insecure-but-functional code (~40–45%) | operational-robustness | NYU + Stanford (Measured); Veracode/Apiiro (vendor) |
| 8 | Hardcoded / fallback secrets | operational-robustness | Weather Report; GitGuardian |
| 9 | Insecure-by-default access control (RLS off, public buckets) | operational-robustness | Weather Report |
| 10 | Weak error handling / happy-path-only | operational-robustness | Augment; Osmani |
| 11 | Missing edge cases (null, empty, Unicode, max-int) | testing-correctness; robustness | Augment; Osmani (the "30%") |
| 12 | Performance anti-patterns (O(n²), alloc-in-loop) | operational-robustness | Augment |
| 13 | Over-engineering / verbose boilerplate | implementation-simplicity | Böckeler #13; Anthropic |
| 14 | Architecture / convention drift (ignores codebase patterns) | extensibility-boundaries | Code With Rigor; Weather Report |
| 15 | **Testing illusion** (tests pass, assert nothing) | testing-correctness | Code With Rigor; Beck ("cheating") |
| — | *Process cluster (no current dimension):* comprehension debt, confidence delusion / automation bias, context-window amnesia, agent-thrashing/sunk-cost, productivity theater | **gap → candidate new dimension** | Code With Rigor; Böckeler; METR; DORA |

### Master remediation list (the craft antidotes — convergent across writers)

1. **Read every line; never ship code you can't explain.** (Willison, Ptacek, Osmani, Böckeler, Code With Rigor)
2. **Verification-as-guardrail** — give the agent a check it can run (tests, build exit code, linters, screenshot diff). "Highest-leverage thing you can do." (Anthropic, Ptacek, Thoughtworks, Beck)
3. **"If you haven't seen it run, it's not working."** Hands-on QA; never trust because it compiled. (Willison, Osmani)
4. **TDD / tests-as-gate** with *meaningful* assertions; have the AI draft edge-case tests. (Beck, Osmani, Böckeler, Thoughtworks)
5. **Small, focused diffs / stackable PRs** as checkpoints. (Osmani, Böckeler)
6. **The "PR contract"** — ship evidence (what/why, proof, risk tier, AI-portions, focus areas). (Osmani)
7. **Calibrate skepticism to margin of error** — heightened review for security/auth/payments/architecture. (Böckeler, Osmani)
8. **Verify every dependency** — existence check, lockfiles, hash pinning, lower temperature. (USENIX, Socket, BleepingComputer)
9. **Write simple, boring code; prefer stable low-churn ecosystems.** (Ronacher, Beck)
10. **Keep the engineering brain engaged**; automate only the proven-repetitive. (Ronacher)
11. **CLAUDE.md/AGENTS.md as a pruned "constitution"**, encoding conventions; audit-logs→refine flywheel. (Anthropic, sshh)
12. **Measure outcomes not output** — bugs shipped, time-to-resolution, clone ratio, moved-vs-copied lines. (Code With Rigor, GitClear)

---

## 5. Convergence-validated findings (pass this skill's ≥3-independent-witness test)

These are ready to graduate into a dimension reference / candidate principle. Each appears in ≥3 *unrelated* sources converging for the same reason.

1. **Comprehension is the gate.** "Don't commit what you can't explain." — Willison + Ptacek + Osmani + Böckeler + Code With Rigor. *(5 witnesses; the single strongest claim in the corpus.)*
2. **Verification-as-guardrail beats trust.** Mechanical checks the agent must pass. — Anthropic + Ptacek + Beck + Thoughtworks + Willison. *(5)*
3. **Plausibility ≠ correctness; the dangerous errors are silent.** — Willison + Anthropic + Molly White + Osmani. *(4)*
4. **AI pressures code toward duplication, away from refactoring.** — GitClear (Measured) + LeadDev + Augment + Ronacher. *(4; one is hard data)*
5. **Insecure-by-default at ~40–45%, and it doesn't improve with model scale.** — NYU + Stanford (both Measured) + Veracode + Apiiro + GitGuardian. *(5; two peer-reviewed)*
6. **Perceived speedup > actual; confidence rises as quality falls.** — METR RCT (Measured) + Stanford (Measured) + DORA + Code With Rigor. *(4; two rigorous)*

---

## 6. The counter-case (boundary condition — mandatory for this skill)

**"Echoes of AI" (Borg et al., EMSE 2025)** is the deliberate opposite witness. A controlled experiment (n=151, mostly professionals) found **no significant maintainability degradation** when later developers evolved AI-written code — slightly *better* CodeHealth when habitual AI users wrote it.

**What this flips:** the maintainability-decay story (GitClear et al.) is *observational and timing-correlational*. The controlled evidence suggests decay is **conditional on review discipline, not inherent to AI authorship.** So every "AI slop" practice this skill adopts must carry the boundary: *the slop is a process failure (unreviewed, unverified, unexplained), not a property of the generator.* This is exactly congruent with the corpus's own definition (§1): the defect is shipping-without-vouching, not AI-origin.

---

## 7. Citable statistics (with calibration + caveat)

- **19.7%** of LLM-recommended packages don't exist; **43%** recur on all 10 reruns — *Measured, peer-reviewed* (USENIX 2025).
- **~40%** of Copilot completions vulnerable across 89 CWE scenarios — *Measured, peer-reviewed* (NYU, S&P 2022).
- SQLi task **36%** vulnerable with AI vs **7%** without; AI users *believed* code was more secure — *Measured* (Stanford, CCS 2023).
- AI made experienced devs **19% slower** while they predicted **+24%** — *Measured, RCT* (METR 2025).
- Copy-paste **exceeded refactoring** for the first time (copy/pasted 8.3%→12.3%, moved ~24%→~9.5%, 2020→2024) — *Measured but correlational, vendor* (GitClear).
- 25% more AI adoption ↔ **−7.2%** delivery stability — *Measured (survey), correlational* (DORA 2024).
- **45%** of AI code failed security tests, flat across model sizes — *vendor* (Veracode 2025).
- **No** maintainability degradation in controlled evolution — *Measured* (Echoes of AI, 2025). **← the boundary.**

---

## 8. Suggested integration into `engineering-craft` (options, not yet executed)

- **(A) New dimension reference `references/ai-slop-antipatterns.md`** — the inverse map: each gold-standard practice (P0–P12) annotated with the AI-slop failure mode it prevents. Slop is the *negative space* of the existing corpus; this leverages what's already built.
- **(B) Strengthen Review mode** — add an AI-slop checklist gated on "is this diff AI-generated?": dependency-existence, silent-logic-error probes, duplication vs. existing code, convention drift, the testing illusion, the "PR contract."
- **(C) Two candidate principles** for `_principles-index.md`, both convergence-validated above:
  - *P13 (candidate): Comprehension is the merge gate* — finding #1.
  - *P14 (candidate): Verification-as-guardrail* — finding #2.
- **(D) Candidate 5th dimension** — the *process/epistemic* cluster (comprehension debt, automation bias, perception gap) fits none of the four code-level dimensions; it's about the human–AI loop. Hold pending more witnesses, per the skill's own convergence discipline.

*All four are proposals. The counter-case (§6) means every item ships with the boundary: AI slop is unreviewed/unverified output, not AI output per se.*
