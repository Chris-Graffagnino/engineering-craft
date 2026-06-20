# Engineering Craft

A Claude skill for **implementation-level engineering craft** — the practices that
distinguish gold-standard codebases at the level *below* system architecture. It is the
craft complement to a software-architecture skill: that judges the *shape of the system*;
this judges *how the code is built* — testing and correctness, failure handling,
implementation simplicity, extension/plugin boundaries, resource control, observability,
and API/backward-compatibility discipline.

The distilled practices live in [`references/`](references/) and are extracted from a
corpus of exemplar open-source codebases using the method described below. **The skill
grows by running Distill mode on new repos; it is consumed by running Apply or Review
mode** when building or reviewing code.

## What makes it different

Most "best-practice" advice is ungrounded assertion. This skill is built on an explicit
epistemic discipline, so every practice it offers is traceable and bounded:

- **Distilled from exemplar repos, not opinion.** Ten codebases: SQLite, FoundationDB,
  Kubernetes, Erlang/OTP, Temporal, Envoy, Linux, VS Code, Redis, and Go.
- **The convergence test.** A practice graduates into the principles index only when it is
  witnessed independently in **≥3 unrelated codebases**, doing it convergently for the same
  reason. One repo doing something is a style choice; three is a principle.
- **Every practice carries its boundary.** A practice without a counter-case is dogma. Each
  one ships with *when it does not apply* — ideally a named repo that deliberately does the
  opposite under a different constraint. The boundaries are the product.
- **Evidence is calibrated.** Claims are tagged Verified / Measured / Codified / Documented /
  Inferred / Folklore, so an inferred practice is never presented as a verified one.
- **One governing dial (P0).** Rigor is calibrated to *blast radius × patch latency* — the
  skill extracts the *calibration*, never the *setting*. (SQLite's 590:1 test-to-code ratio
  is the output of an extreme dial, not a target to copy.)

## The craft dimensions

| Dimension | Reference | Principles | Exemplars |
|---|---|---|---|
| Testing & correctness | [`testing-correctness.md`](references/testing-correctness.md) | P1–P4 | SQLite, FoundationDB, Kubernetes |
| Implementation simplicity | [`implementation-simplicity.md`](references/implementation-simplicity.md) | P5–P6 | Redis, SQLite, Go |
| Operational robustness | [`operational-robustness.md`](references/operational-robustness.md) | P7–P9 | Kubernetes, Erlang/OTP, Temporal |
| Extensibility & boundaries | [`extensibility-boundaries.md`](references/extensibility-boundaries.md) | P10–P12 | Envoy, Linux, VS Code |
| The governing dial | [`_principles-index.md`](references/_principles-index.md) | P0 | all |

The capstone [`_principles-index.md`](references/_principles-index.md) holds the
convergence-validated core (P0–P12); each principle links to its dimension reference and to
the per-repo evidence.

## Installation

This is a Claude skill. Clone it where Claude discovers skills:

```sh
# Personal (all projects)
git clone https://github.com/<you>/engineering-craft ~/.claude/skills/engineering-craft

# or per-project
git clone https://github.com/<you>/engineering-craft .claude/skills/engineering-craft
```

The directory name should match the skill name (`engineering-craft`). It is then
invocable as `/engineering-craft`.

## Usage

| Invocation | Mode |
|---|---|
| `/engineering-craft <question>` | **Apply** — answer a craft question with the relevant distilled practices and their boundary conditions. |
| `/engineering-craft review [PR# \| diff \| path]` | **Review** — judge how a specific change is *built*. Adds the craft layer; does not replace a correctness review. |
| `/engineering-craft distill [repo]` | **Distill** — extract reusable craft practices from an exemplar repo into the corpus. |
| `/engineering-craft help` | Print the help card ([`assets/help.md`](assets/help.md)). |

## Repository layout

```
SKILL.md                      operating instructions + the distillation method
assets/help.md                the help card
references/
  _principles-index.md        capstone: convergence-validated principles P0–P12
  testing-correctness.md      } the four
  implementation-simplicity.md  } dimension
  operational-robustness.md     } references
  extensibility-boundaries.md }
  _extract-<repo>.md          per-repo evidence (the provenance for every practice)
  _extraction-template.md     the instrument Distill mode fills for a new repo
  _progress.md                corpus development log
  ai-slop-antipatterns.md     } literature-sourced extensions (see below)
  review-mode-playbook.md     }
  _research-ai-slop-corpus.md }
```

The `_extract-*.md` files are shipped on purpose: they are the auditable provenance behind
every claim, and the worked record that lets others run Distill mode.

## Two literature-sourced extensions

[`ai-slop-antipatterns.md`](references/ai-slop-antipatterns.md) and
[`review-mode-playbook.md`](references/review-mode-playbook.md) extend Review mode toward
AI-generated-code failure modes and the mechanics of running a review. They are a
**different evidence species** — sourced from cross-industry literature (studies, an RCT,
practitioner essays) rather than exemplar repos — and they are honestly labeled as such:
**they add no principles to the convergence-validated core**. They map AI-code failure modes
onto the existing P0–P12 (AI slop is the *negative space* of the corpus) and operationalize
the review, without claiming the same standard of evidence as the dimensions.

## Companion: a software-architecture skill

This skill deliberately owns only the *code/component* altitude. Questions about service
boundaries, monolith-vs-microservices, distributed transactions, consistency models,
bounded contexts, ADRs, or team topology are *architecture*, and the skill hands them off
to a software-architecture skill at that seam. A companion architecture skill is optional —
without one, the handoff still reads as an honest "this is out of scope for craft."

## Status

v1 — all four dimensions graduated (P0–P12), each by three independent witnesses, across a
ten-repo corpus. Held candidates (recorded in the principles index) graduate as further
witnesses corroborate them.

## License

[MIT](LICENSE) © 2026 Chris Graffagnino
