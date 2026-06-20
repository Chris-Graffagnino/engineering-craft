# Extraction: Linux Kernel — Extensibility (Module/Driver Surface Governance)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/torvalds/linux` (GPL-2.0) at commit `a975094`. Tier 1 — `Documentation/process/stable-api-nonsense.rst`, `MAINTAINERS` (field legend + 3,244 `S:` status entries), `drivers/staging/Kconfig`. Tier 3 — `include/linux/export.h` (`EXPORT_SYMBOL`/`EXPORT_SYMBOL_GPL`), `include/linux/module.h` (`module_init`/`module_exit`).

**Seam discipline:** stays on the *craft* side of the boundary with `software-architect`. Extracts the **mechanics** of governing a huge module/extension surface — ownership, the narrow exported ABI, the quality-tiered staging area, the in-tree contract. Does **not** argue "monolithic vs microkernel" or any system-shape question — that is architecture and routes to the other skill.

Dual purpose: (1) the **second extensibility witness**, maximally independent from Envoy (C, 1991, kernel, BDFL+MAINTAINERS) — it corroborates most of Envoy's C1/C32–C38 and lifts them from 1/3 to **2/3** (see §3a); (2) it contributes two new candidates (C39, C40) and a sharp **refinement/counter-case to C34**. Nothing reaches the ≥3 bar (extensibility has only two witnesses), so no graduation into `_principles-index.md`.

---

## 0. Identity

- **Repo:** Linux kernel (`torvalds/linux`).
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `a975094`. Thesis is governance *mechanics*, not decision-evolution, so no full history.
- **Language(s):** C (modules), plus the codified process docs.
- **Why it's in the corpus:** The reference example of governing a *massive* module/driver surface over decades — and the **historical origin** of the structural-ownership file (`MAINTAINERS` predates `CODEOWNERS`). A convergence between Linux and Envoy is independent-discovery across language, era, and domain — the strongest kind the convergence test can get.

## 1. Thesis

**A vast, multi-author extension surface is governed by four levers — structural ownership, a narrow exported ABI, a quality-tiered staging area, and keeping extensions *in-tree* — and the fourth is what lets the internal interface stay deliberately *unstable*: because extensions live in the tree, the developer who changes an interface fixes every user, so the core never has to freeze.** Linux teaches that the cost of a stable internal extension contract is avoidable *if* you pay the in-tree price instead.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `Documentation/process/stable-api-nonsense.rst` (Greg Kroah-Hartman) — the doctrine: stable *userspace* ABI, deliberately *unstable* internal module ABI, and "get your driver in-tree" as the resolution.
- `MAINTAINERS` — the structural-ownership registry: `M:` maintainer, `R:` reviewer, `F:` owned files/dirs, `L:` list, `S:` status; parsed by `scripts/get_maintainer.pl`.
- `drivers/staging/Kconfig` — the staging tier's own description ("not of the 'normal' Linux kernel quality level").

**Tier 3 — Code (confirm/falsify):**
- `include/linux/export.h` — `EXPORT_SYMBOL(sym)` (license `""`) vs `EXPORT_SYMBOL_GPL(sym)` (license `"GPL"`), with symbol namespaces — the narrow ABI surface + a license/capability posture.
- `include/linux/module.h` — `module_init()`/`module_exit()`: modules self-register via initcall sections; core discovers them additively.

## 3. Candidate practices (extensibility-boundaries dimension)

### C39. Keep extensions in-tree so the core can evolve freely — the interface-breaker fixes the extensions

- Practice: Bring extensions *into the main tree* and adopt collective maintenance: when someone changes a core interface, they fix **every** in-tree user in the same change. The payoff is that the *internal* extension interface never has to be frozen — it can be reworked for bugs, security, or design whenever needed, because there are no unknown external users to break.
- Evidence: **Codified** — `stable-api-nonsense.rst`: "get your kernel driver into the main kernel tree… if a kernel interface changes, it will be fixed up by the person who did the kernel change… This ensures that your driver is always buildable"; "all of the instances of where this interface is used within the kernel are fixed up at the same time." **Convergent in Envoy** — `EXTENSION_POLICY.md`: "Any extension added via this process becomes a full part of the repository… any API breaking changes in the core code will be automatically fixed as part of the normal PR process by other contributors."
- Why / constraint: A frozen internal contract forces carrying old/broken interfaces forever; in-tree + collective-fixup buys freedom to change at the cost of carrying the extension's source in your tree and review burden.
- Counter-case: Only works if you *can* hold extensions in-tree. A product that must support **third-party / out-of-tree** plugins it never sees (most plugin ecosystems!) cannot do this — it must offer a stable versioned contract instead (C34). Linux itself supports the external contract it *can't* avoid (userspace syscalls) rigidly, and taxes out-of-tree modules (no ABI, taint) rather than serving them.
- Convergence: **2/3** — Envoy + Linux, two maximally-independent repos, same practice and reason.
- Graduates to: `extensibility-boundaries.md`

### C40. Prune the extension/interface surface — an unused extension point is untested liability

- Practice: Actively delete extension points and exported interfaces that no one uses, rather than keeping them "just in case." A surface you don't prune grows untestable and becomes a maintenance and security liability.
- Evidence: **Codified** — `stable-api-nonsense.rst`: "Kernel interfaces are cleaned up over time. If there is no one using a current interface, it is deleted. This ensures that the kernel remains as small as possible, and that all potential interfaces are tested as well as they can be (unused interfaces are pretty much impossible to test for validity)."
- Why / constraint: Every exported interface is a contract you must keep working and a path attackers/fuzzers can reach; an *unused* one carries all that cost with none of the value, and can't be validated because nothing exercises it.
- Counter-case: Aggressive pruning is safe only with the in-tree model (C39) — you can see all users and prove an interface is unused. With unknown external consumers (C34's world), you can't prove disuse, so you deprecate on a timer (C38) instead of deleting outright.
- Convergence: **1/3** (Linux). Rhymes with Redis's small-core / understandability ethos (C3) — cross-check when `implementation-simplicity.md` is written; plausibly 2/3 there.
- Graduates to: `extensibility-boundaries.md`

## 3a. Convergence cross-check — Linux as the second extensibility witness

Extensibility had a single witness (Envoy, C1/C32–C38 all 1/3). Linux independently witnesses most of them *for the same reason* via different mechanisms, lifting them to **2/3**. Two witnesses < the ≥3 bar, so nothing graduates into `_principles-index.md`.

| Envoy candidate | Linux mechanism | Verdict |
|---|---|---|
| **C1** structural ownership (sponsor + reviewers, machine-checkable) | `MAINTAINERS`: `M:`/`R:`/`F:` map owners→files (3,244 entries, parsed by `get_maintainer.pl`) — the *origin* of the pattern | **1/3 → 2/3** (strong; independent discovery across decades) |
| **C32** narrow interface + fixed surface | `EXPORT_SYMBOL`/`_GPL`: a module may use *only* exported symbols; core internals stay unexported and free to change | **1/3 → 2/3** (strong) |
| **C33** decoupled registration | `module_init()`/initcall: modules self-register, core discovers additively (no hardcoded list) | **1/3 → 2/3** (the *decoupling* converges; mechanism differs — initcall sections vs named-factory-by-config-string) |
| **C34** typed/versioned config contract | **REFINEMENT + counter-case:** *userspace* syscall ABI is sacrosanct/stable (corroborates "version the external contract"); *internal* module ABI is deliberately **unstable** (refuses to freeze the internal extension API) | **split:** external-boundary half **→ 2/3**; over-stabilizing the *internal* API gets a sharp **counter-case** (see C39) |
| **C35** declared maturity/posture metadata | `MAINTAINERS` `S:` status (Supported/Maintained/Odd-Fixes/**Orphan**/**Obsolete** — the last two honestly surface unmaintained code, like Envoy's largest-bucket `unknown`) + `EXPORT_SYMBOL_GPL` license posture + taint flags | **1/3 → 2/3** (strong) |
| **C36** tier the bar (core vs contrib) | `drivers/staging/`: "not of the normal Linux kernel quality level," default-off ("if in doubt, say N"), **self-marking** (using one taints the kernel), TODO-gated | **1/3 → 2/3** (near-perfect; the taint flag *is* C36's honesty requirement made machine-enforced) |
| **C37** same bar + dependency hygiene | staging-vs-mainline implies a normal-quality bar (corroborates the "same bar" half); kernel has no external-dep CPE/shepherd machinery | **partial** — bar half →2/3; the dependency-hygiene specifics stay **Envoy-only 1/3** |
| **C38** lifecycle (add point / timed removal) | staging→mainline graduation (TODO-gated); unused interfaces **deleted** outright (C40); driver deprecation | **1/3 → 2/3** |

**The key nuance.** Both repos independently land on the **in-tree extension model** (C39) and on **tiering the bar with a self-marking lower tier** (C36 + taint). The richest result is the C34 *refinement*: version the contract at the **external** boundary where unknown consumers depend on you (Linux: userspace ABI; Envoy: the proto config API), but you may deliberately keep the **internal** extension API *unstable* — *iff* extensions are in-tree so the breaker fixes them (C39). Whether to version an extension contract is therefore not a flat yes — it's "yes at the external boundary, optional internally, decided by whether extensions are in-tree."

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            "No stable internal API" (deliberately unstable internal extension interface).
- Why not general: A SHARP counter-case, valid ONLY under Linux's constraints — a single mainline
                   tree, GPL, and the power to hold every extension in-tree. A product that must
                   support third-party / out-of-tree plugins it never sees CANNOT refuse a stable
                   contract; it must version it (C34). Copy the calibration ("version the EXTERNAL
                   boundary; internal freedom is conditional on in-tree extensions"), NOT the stance.
```

```
- Item:            A single 30k-line MAINTAINERS file with 3,244 entries.
- Why not general: Scaled to the kernel's surface. The transferable principle is machine-checkable
                   path→owner mapping (C1), not a monolithic file — a small project uses CODEOWNERS
                   globs or a few lines. Don't build a MAINTAINERS file for a ten-module project.
```

```
- Item:            Taint-the-kernel as the lower-tier marker.
- Why not general: A kernel-specific enforcement mechanism. The transferable idea is "make the
                   lower tier machine-detectable and default-off" (Envoy does it with default-off
                   builds + metadata), not "implement a taint flag."
```

## 5. Open questions / corpus cross-checks

- This lifts **C1, C32, C33, C35, C36, C38** to **2/3** and refines **C34**; adds **C39** (2/3) and **C40** (1/3). A **third, fully-independent** extensibility witness would graduate the strong set: **VS Code / browser extension models** (manifest + declared capabilities/permissions → C32/C34/C35) or **Rust** (`#[stable]`/feature-gates → C35/C36, RFC ownership → C1) are the cleanest, both independent of *both* Envoy and Linux.
- **C1 caution:** Kubernetes `OWNERS` would be a third witness, but Envoy↔Kubernetes share CNCF lineage — that independence is weak, so a K8s extraction would only *softly* push C1 toward 3/3. Prefer Rust or an extension-marketplace model for a clean third.
- **C40** (prune unused interfaces) rhymes with Redis's small-core ethos (C3) — recheck when `implementation-simplicity.md` is written; likely 2/3 there and a bridge between the two dimensions.
- The C34 refinement (external-stable / internal-free, conditional on in-tree) is the dimension's sharpest result — make sure `extensibility-boundaries.md` foregrounds it rather than a flat "version your contract."
