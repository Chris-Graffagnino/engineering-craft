# Engineering Craft — help

Implementation-level engineering craft: the practices that distinguish gold-standard
codebases at the level *below* system architecture. This skill judges **how code is
built** (testing, failure handling, simplicity, extension boundaries, resource control,
observability, API/compat) — the complement to a system-architecture skill, which judges
the *shape of the system*.

## How to invoke

    /engineering-craft <argument>

| Argument | What it does |
|---|---|
| *(none)* or a question | **Apply mode** — answer a craft question, leading with the relevant distilled practices and their boundary conditions. |
| `review [PR# \| diff \| path]` | **Review mode** — judge how a specific change is *built* against the craft practices. Adds the craft layer on top of a correctness review; it does not replace one. |
| `distill [repo]` | **Distill mode** — extract reusable craft practices from an exemplar repository into the corpus. |
| `help` | Print this text. |

## The craft dimensions

| Dimension | Principles |
|---|---|
| Testing & correctness — invariant discipline, fault injection, deterministic/simulation testing | P1–P4 |
| Implementation simplicity — understandability budget, minimal dependency surface | P5–P6 |
| Operational robustness — reconcile/retry, failure handling, resource control | P7–P9 |
| Extensibility & boundaries — narrow versioned contracts, declared posture, governed lifecycle | P10–P12 |
| The governing dial — calibrate rigor to blast radius × patch latency | P0 |

Each principle carries its **boundary condition** — when it does *not* apply — because the
boundaries are the product. Practices are graduated only when witnessed independently in
≥3 unrelated exemplar codebases.

## What this skill is NOT for

System-level architecture — service boundaries, monolith-vs-microservices, sagas /
distributed transactions, consistency-model selection, bounded contexts, ADRs, team
topology, whole-codebase architectural review. Those belong to a software-architecture
skill; this skill hands off to it at that seam.

See `README.md` for the full method, and `references/` for the dimension references,
the principles index, and the per-repo evidence.
