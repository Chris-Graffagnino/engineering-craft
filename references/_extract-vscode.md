# Extraction: VS Code — Extensibility (the third-party-plugin pole)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/microsoft/vscode` (MIT) at commit `b5f8b27`. Tier 3 — `src/vscode-dts/vscode.d.ts` (the stable typed extension API — **21,235 lines**), `src/vscode-dts/vscode.proposed.*.d.ts` (**175** proposed-API files — the staging pipeline), `src/vs/platform/extensions/common/extensions.ts` (the manifest interface: `contributes`, `activationEvents`, `capabilities`, `untrustedWorkspaces`, `preview`, `enabledApiProposals`), `extensionValidator.ts` (`engines.vscode` version-compat).

**Seam discipline:** stays on the *craft* side of the boundary with `software-architect`. Extracts the **mechanics** of a third-party extension surface — the typed/versioned API contract, declarative contribution points, declared capabilities, the proposed-API pipeline. It does **not** decide whether a product *should* be extension-driven — that's architecture.

Dual purpose: (1) the **third extensibility witness**, maximally independent from Envoy (C++ proxy) and Linux (C kernel) — a TypeScript editor whose entire identity is its extension model, governed by a marketplace; (2) it **graduates the extensibility core** — C32+C33+C34 → **P10**, C35 → **P11**, C38 → **P12**. It contributes one new candidate (C48), **holds** C1 and C36, and is the **third-party-plugin counter-pole** that completes the C34 in-tree/out-of-tree refinement (and is the counter-case to C39). See §3a.

---

## 0. Identity

- **Repo:** Visual Studio Code (`microsoft/vscode`).
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `b5f8b27`. Thesis is extension-surface mechanics.
- **Language(s):** TypeScript (the editor + the extension API surface).
- **Why it's in the corpus:** The reference example of a *pure, third-party* extension surface at marketplace scale — and the cross-paradigm independence (TS editor / marketplace governance) needed to graduate the extensibility core, plus the third-party pole of the in-tree/out-of-tree calibration.

## 1. Thesis

**A large third-party extension surface is sustained by a single typed, versioned API contract that extensions attach to and can never reach behind, declarative contribution points the host discovers, declared capabilities/trust posture per extension, and a staged proposed-API pipeline that matures new surface behind an explicit opt-in before promoting it to the stable contract.** And because the extensions are *third-party* (out-of-tree), the contract **must** be versioned and stable — the host can never fix an extension it didn't write.

## 2. Three-tier source inventory

**Tier 1 — Codified policy:** the VS Code Extension API docs + the API-proposal process (code.visualstudio.com / the `vscode.proposed.*` convention) — web-attributed; plus the manifest schema below.

**Tier 3 — Code (confirm/falsify):**
- `src/vscode-dts/vscode.d.ts` — the stable, typed extension API (21,235 lines): one narrow, versioned contract the whole surface attaches to.
- `src/vscode-dts/vscode.proposed.*.d.ts` — 175 *proposed* API files; opt-in via `enabledApiProposals`, promoted to `vscode.d.ts` when stable.
- `src/vs/platform/extensions/common/extensions.ts` — the manifest interface: `contributes` + `activationEvents` (declarative registration), `capabilities` / `untrustedWorkspaces` / `virtualWorkspaces` (declared trust posture), `preview` (maturity), `engines` (API-version compat).
- `extensionValidator.ts` — `validateExtensionManifest` / `isValidVersion`: enforces `engines.vscode` compatibility.

## 3. Candidate practices (extensibility-boundaries dimension)

### C48. Stage new contract surface behind an explicit opt-in, mature it, then promote to the stable contract

- Practice: Don't add new public API straight to the stable contract. Ship it as a *proposed/experimental* surface gated behind an explicit opt-in flag; let early adopters use it knowing it can change; promote it into the stable contract only once it's proven, and never break the stable contract once promoted.
- Evidence: **Verified** — `src/vscode-dts/vscode.proposed.*.d.ts` (175 staged API files) + `enabledApiProposals` in the manifest interface; the stable surface is the separate `vscode.d.ts`. **Documented** — the VS Code API-proposal process (proposed → stable).
- Why / constraint: A stable contract for third-party consumers can't be experimented on in place (you'd break them); a gated staging surface lets the API evolve *and* keeps the stable contract's promise intact.
- Counter-case: Needs a real opt-in/gating mechanism and the discipline to actually promote or drop staged surface (staging that never graduates rots, like any backlog). Overhead unjustified for an internal API with no third-party consumers.
- Convergence: **2/3** — VS Code (proposed-API) + **Linux** (`drivers/staging/` → mainline graduation) are the same stage-then-promote pipeline. Rust's nightly-feature-gates (`#![feature]` → stable) is the obvious 3rd (would graduate it; also rhymes with Envoy's `v3alpha`).
- Graduates to: `extensibility-boundaries.md` (a maturation-pipeline refinement of P12).

## 3a. Convergence cross-check — VS Code as the third extensibility witness (the dimension graduates)

VS Code independently witnesses the extensibility core from a third direction (a third-party marketplace surface, vs Envoy's in-tree filters and Linux's in-tree modules). With three unrelated repos these clear the **≥3 bar and graduate** — completing the fourth and final dimension.

| Candidate | Envoy | Linux | VS Code | Verdict |
|---|---|---|---|---|
| Narrow interface + decoupled registration + versioned config (**C32 + C33 + C34**) | filter interface + `FactoryRegistry` + proto config | `EXPORT_SYMBOL` + `module_init` + (no internal-ABI version — counter-case) | the 21k-line typed `vscode.d.ts` + declarative `contributes`/`activationEvents` + `engines.vscode` version compat | **GRADUATES → P10** |
| Declared maturity + trust posture as machine-readable metadata (**C35**) | `status` (stable/alpha/wip) + `security_posture` | `MAINTAINERS` `S:` status + `EXPORT_SYMBOL_GPL` | `preview` + `capabilities` / `untrustedWorkspaces` / `virtualWorkspaces` | **GRADUATES → P11** |
| Govern the extension lifecycle (**C38**) | governed point-add + 6-month timed deprecation | staging→mainline graduation; delete-unused | proposed-API → stable promotion; API deprecation | **GRADUATES → P12** |
| Structural ownership (**C1**) — HELD | sponsor + 2 reviewers in `CODEOWNERS` | `MAINTAINERS` (origin) | `publisher` field + marketplace verification (*marketplace-level, not in-repo path→owner*) | **stays 2/3** — VS Code's is a weaker, marketplace-level instance |
| Tier the bar to blast radius (**C36**) | core vs contrib | `drivers/staging/` | preview/proposed vs stable; verified publishers | **anchored to P0** — partial here; doesn't need its own graduation |
| In-tree extension model (**C39**) | extensions in-repo, breaker fixes them | drivers in-tree | **COUNTER-POLE: third-party, out-of-tree** — VS Code can never fix an extension, so it MUST keep a stable versioned contract | **stays 2/3** — VS Code completes the C34 refinement from the opposite pole |

**The refinement, completed.** Linux (in-tree, *no* stable internal API) and VS Code (out-of-tree, *strict* stable versioned API) are the two poles of the C34/P10 calibration: **version the contract iff you can't fix your consumers.** Linux *can* (in-tree, the breaker fixes drivers) so it keeps the internal API free; VS Code *can't* (third-party marketplace) so it freezes the contract and stages all change through proposed-API (C48). Envoy sits between (in-tree extensions, but a versioned proto config for external operators). That spectrum — not a flat "version your contract" — is the graduated boundary on P10.

**The disciplined holds.** **C1** stays 2/3: VS Code's ownership is a *marketplace* `publisher` + verification, not an in-repo machine-checkable path→owner mapping like `CODEOWNERS`/`MAINTAINERS` — a weaker instance, so it doesn't cleanly graduate. **C36** is already P0 applied (the governing dial), partially present here (preview/verified-publisher); it surfaces as the dial, not a standalone graduate.

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            A 21k-line typed API + a 175-file proposed-API staging pipeline + a marketplace.
- Why not general: Sized to a third-party extension surface with (literally) tens of thousands of
                   independent authors. For an internal plugin system with a handful of first-party
                   extensions, this is enormous over-engineering. EXTRACT the principles (a narrow
                   versioned contract P10; declared posture P11; a governed lifecycle P12) and apply
                   them as one typed interface + a manifest + a CHANGELOG.
```

```
- Item:            Freezing the public contract and never breaking it.
- Why not general: Correct ONLY because extensions are third-party and unfixable (the out-of-tree
                   pole). A system that holds its extensions in-tree (Linux, Envoy) can and should
                   keep more internal freedom (the C34/P10 refinement). Don't freeze a contract you
                   are free to evolve because you can fix every consumer.
```

## 5. Open questions / corpus cross-checks

- **Graduations done:** C32+C33+C34 → **P10**, C35 → **P11**, C38 → **P12** (3/3 across Envoy, Linux, VS Code — three independent extension models). **This completes all four dimensions.** Move them from the tracked-candidates ledger into the validated core.
- **Held:** C1 (ownership, 2/3 — needs a clean in-repo path→owner 3rd witness, not a marketplace), C39 (in-tree, 2/3 — VS Code is its out-of-tree counter-pole), C48 (staging pipeline, 2/3 — Rust nightly would graduate it).
- **Still 1/3:** C37 (dep-hygiene specifics, Envoy), C40 (prune-surface, Linux).
- Note the cross-dimension rhyme: C48 (stage-behind-opt-in) ↔ Rust nightly feature-gates ↔ the simplicity dimension's C46 (gofmt) all live in the "mechanize/govern evolution" family; a Rust extraction would touch several at once.
