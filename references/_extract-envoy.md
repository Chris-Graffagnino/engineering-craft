# Extraction: Envoy — Extensibility & Module Boundaries (Extension-Surface Craft)

Filled from `_extraction-template.md` via Distill mode. Sources: surgical blobless+sparse clone of `github.com/envoyproxy/envoy` (Apache-2.0) at commit `231c51d`. Tier 1 — `EXTENSION_POLICY.md`, `DEPENDENCY_POLICY.md`, `GOVERNANCE.md`, `STYLE.md`, `CODEOWNERS`. Tier 3 — `envoy/server/filter_config.h`, `envoy/registry/registry.h`, `envoy/http/filter.h`, `source/extensions/extensions_metadata.yaml`. Scale of the surface this governs: **2,408 files across 42 top-level extension categories** (clone census), ~343 extensions carrying explicit metadata.

**Seam discipline:** this extraction stays on the *craft* side of the boundary with `software-architect`. It extracts the **mechanics** of a clean extension surface — the interface contract, registration, typed config, and the governance that keeps the surface trustworthy. It does **not** address "should this system be a plugin architecture / microkernel," nor domain/bounded-context module boundaries — those are architecture and route to the other skill.

This **founds the extensibility dimension**; Envoy is its sole witness, so every candidate is **1/3**. Two candidates are the dimension's *application* of already-graduated corpus-wide principles — **P0** (the governing dial, via C36) and **P2** (narrow-interface seams, via C32) — which gives the dimension real grounding; see §3a. C1, C2 were seeded earlier in `_extraction-template.md`; this file formalizes C1 (extensibility) and adds C32–C38. (C2, the watermark/backpressure practice, is a *robustness* candidate, not extensibility — noted in §5.)

---

## 0. Identity

- **Repo:** Envoy (`envoyproxy/envoy`).
- **Commit / clone depth:** `--depth 1 --filter=blob:none --sparse` at `231c51d`. Thesis is extension-surface *mechanics*, not decision-evolution, so no full history.
- **Language(s):** C++ (extensions), Protobuf (the config API surface).
- **Why it's in the corpus:** The reference example of running a **huge, multi-organization first-party extension surface** without letting it rot the core — and of codifying the governance that makes that possible.

## 1. Thesis

**A large, multi-author extension surface stays healthy when extensions plug in through a *narrow, uniform contract* and are kept trustworthy by *codified governance whose rigor is tiered to each extension's blast radius* — never by letting an extension reach into the core.** Two halves that recur throughout: the **technical contract** (how an extension attaches) and the **governance** (how the surface stays trustworthy), with **calibration** the through-line.

## 2. Three-tier source inventory

**Tier 1 — Codified policy/philosophy:**
- `EXTENSION_POLICY.md` — the central doctrine: same quality bar as core, maintainer sponsorship + two reviewers, `status`/`security_posture` metadata, the core-vs-contrib tier, and the add/remove lifecycle.
- `DEPENDENCY_POLICY.md` — hygiene for the (extension) dependency surface: CPEs, pinned/direct-fetch, binary-size isolation, dependency shepherds.
- `GOVERNANCE.md`, `STYLE.md`, `CODEOWNERS` — maintainer process, coding standards, machine-checkable ownership.

**Tier 2 — Design notes:**
- `docs/root/extending/extending.rst` (referenced by the policy as the canonical extension-point catalogue); `api/STYLE.md` (config-API/proto versioning).

**Tier 3 — Code (confirm/falsify):**
- `envoy/http/filter.h` — the stream-filter interface and its `FilterHeadersStatus`/`FilterDataStatus` control protocol.
- `envoy/server/filter_config.h` — the *named, typed-config* factory interfaces (`NamedHttpFilterConfigFactory::createFilterFactoryFromProto`).
- `envoy/registry/registry.h` — `FactoryRegistry`/`RegisterFactory`, lookup by name ("a single name cannot be registered twice").
- `source/extensions/extensions_metadata.yaml` — the live `status`/`security_posture` values for the whole surface.

## 3. Candidate practices (extensibility-boundaries dimension)

### C1. Encode extension ownership structurally, not socially

- Practice: Every in-tree extension has a named maintainer **sponsor** (who shepherds *and* maintains it) plus **≥2 declared reviewers**, recorded in a machine-checkable `CODEOWNERS` file — ownership is a build-enforced fact, not a wiki convention. New extensions clear the same bar as core.
- Evidence: **Codified** — `EXTENSION_POLICY.md` ("must be sponsored by an existing maintainer"; "two reviewers… will be codified in the CODEOWNERS file"; sponsorship "ensures the extension will ultimately meet the Envoy quality bar" and "incentives are aligned… not added without sufficient thought put into future maintenance"). **Verified** — `CODEOWNERS` maps paths → owners.
- Why / constraint: A large plugin surface with many contributors decays unless responsibility is unambiguous and enforced; sponsorship aligns incentives so extensions aren't dumped in without a maintenance owner.
- Counter-case: Overkill for a small single-team codebase with few extension points — the ceremony costs more than the decay it prevents. Threshold is contributor-count × extension-surface size, not project age. (Redis answers the same decay pressure *oppositely* — keep a deliberately small core and resist adding extension surface at all.)
- Convergence: **1/3** (Envoy). Kubernetes (`OWNERS` + KEP sponsorship) and Rust (RFC + team ownership) are likely corroborators, not yet formally extracted.
- Graduates to: `extensibility-boundaries.md`

### C32. Extend through a narrow uniform interface + a small fixed control protocol

- Practice: Core defines *one small interface per extension kind* (e.g. a stream filter: `decode/encodeHeaders/Data/Trailers`); an extension implements only that and signals back through a **fixed status enum** (`Continue`, `StopIteration`, `StopIterationAndBuffer`, …) rather than touching core control flow. Core orchestrates the chain; the extension is a leaf that returns status.
- Evidence: **Verified** — `envoy/http/filter.h` (`enum class FilterHeadersStatus { Continue, StopIteration, … }`, `FilterDataStatus`; the `StreamDecoderFilter`/`StreamEncoderFilter` interfaces).
- Why / constraint: A narrow contract + fixed return protocol lets core evolve its orchestration (buffering, ordering, backpressure) without changing extensions, and stops an extension corrupting core invariants — it can only return a status core already knows how to handle.
- Counter-case: A fixed enum constrains what an extension can express; a genuinely novel control need requires *widening the protocol* (a governed change, C38), not a side-channel into core. Over-narrow interfaces force awkward encodings — the surface must actually fit the extension kind.
- Convergence: This is the **extension-side face of the narrow-interface seam that graduated as P2** (SQLite VFS/malloc, FoundationDB's Flow interfaces, Kubernetes's clientset — the same construct seen from the *test* side). Per `_principles-index.md` P2, the test-side instance lives there; this extension-specific elaboration (the control protocol) is Envoy-only = **1/3**. *Do not double-count the shared mechanism.*
- Graduates to: `extensibility-boundaries.md` (cross-linked to P2 / `testing-correctness.md`)

### C33. Decouple discovery & instantiation — a named factory in a registry, built from typed config

- Practice: Extensions never appear in core code paths. Each registers a **factory** under a unique string name in a global **registry**; core instantiates by looking up the name carried in config. The factory consumes a typed config message and returns the extension.
- Evidence: **Verified** — `envoy/registry/registry.h` (`FactoryRegistry<Base>` / `RegisterFactory<T,Base>`; "Classes are found by name, so a single name cannot be registered twice for the same Base class"; `getFactory(name)`); `envoy/server/filter_config.h` (`NamedHttpFilterConfigFactory::createFilterFactoryFromProto(const Protobuf::Message& config, …)`, `createEmptyConfigProto`).
- Why / constraint: Name+registry indirection makes adding an extension *purely additive* — no core switch-statement to edit, no link-time coupling. Typed config means the wiring is validated, not stringly-typed.
- Counter-case: The registry is global mutable state set up at startup; name collisions are a real failure mode (hence the "cannot be registered twice" guard). For a handful of compile-time-known extensions, a plain factory function — or a direct call — is simpler than a registry. Don't build a plugin registry for three built-ins.
- Convergence: **1/3** (Envoy). Factory+registry is a near-universal plugin pattern; a second formally-extracted witness would corroborate.
- Graduates to: `extensibility-boundaries.md`

### C34. Make the typed config schema the versioned contract

- Practice: An extension's *configuration is its public API* — express it as a typed, versioned schema (protobuf in a versioned API package), not free-form maps. Keep config-schema stability **orthogonal** to implementation maturity, each versioned on its own track.
- Evidence: **Verified** — config arrives as `Protobuf::Message` (`filter_config.h`). **Codified** — `api/STYLE.md` governs the proto API; `EXTENSION_POLICY.md` states the orthogonality explicitly ("an extension with a stable config proto can have a `wip` implementation," and vice-versa).
- Why / constraint: Users depend on *config*, not code; a typed versioned schema lets the implementation churn while the contract holds (and lets the config evolve without forcing an implementation rewrite). Free-form config breaks consumers silently.
- Counter-case: Protobuf + versioned-API machinery is heavy; an internal extension point consumed only in-tree can use a plain typed struct. The discipline scales with the number of *external* config consumers.
- Convergence: **1/3** (Envoy). Kubernetes CRDs + API-group versioning is the obvious corroborator (and xDS shares its lineage) — not yet extracted.
- Graduates to: `extensibility-boundaries.md`

### C35. Declare each extension's maturity and threat posture as machine-readable metadata

- Practice: Tag every extension with a `status` (`stable`/`alpha`/`wip`) and a `security_posture` (`robust_to_untrusted_downstream` / `…_and_upstream` / `requires_trusted_downstream_and_upstream` / `unknown` / `data_plane_agnostic`) in a schema-checked metadata file. `unknown` is an explicit placeholder that *forces* classification rather than implying safety.
- Evidence: **Codified + Verified** — `EXTENSION_POLICY.md` §"Extension stability and security posture"; `source/extensions/extensions_metadata.yaml` carries the values for the whole surface (verified distribution: 140 `unknown`, 79 `robust_to_untrusted_downstream`, 64 robust-both, 45 `requires_trusted…`, 15 `data_plane_agnostic`).
- Why / constraint: A large surface mixes wildly different trust/maturity levels; making each explicit and machine-readable lets users reason about what they're enabling and lets tooling gate builds/audits. The honest "unknown" bucket being the *largest* is the point — it surfaces what hasn't been vetted.
- Counter-case: Worth it only at surface scale and where a threat model exists (data-plane code facing untrusted input). A library of pure-compute plugins with no trust boundary doesn't need a `security_posture`.
- Convergence: **1/3** (Envoy). Rust's stability attributes (`#[stable]`/`#[unstable]`) + feature gates are a likely corroborator for the *status* half — not yet extracted.
- Graduates to: `extensibility-boundaries.md`

### C36. Tier the quality bar to blast radius — core vs contrib (the dimension's dial)

- Practice: Don't apply one bar to all extensions. Envoy runs two tiers: **core** (full Envoy quality bar, compiled/tested by default, eligible for security-team coverage) and **contrib** (lower barrier, **not** in default builds, requires an *end-user sponsor* who attests to real usage, **not** security-covered). The bar is set by how widely the extension ships and who is exposed.
- Evidence: **Codified** — `EXTENSION_POLICY.md` §"Contrib extensions" ("barrier to entry… lower than a core extension"; "not included by default in the main image builds"; "require an end-user sponsor"; "not eligible for Envoy security team coverage").
- Why / constraint: Holding every experimental or niche extension to the full core bar would either block useful work or dilute the bar; tiering lets low-blast-radius extensions exist at lower cost while protecting the default build.
- Counter-case: Two tiers add process; a small project needs one. And tiering must be *honest* — a lower tier is a liability if users can't tell they've opted into lower assurance (Envoy mitigates with default-off builds + explicit metadata, C35).
- Convergence: This is the **extensibility-specific application of P0** (calibrate rigor to blast radius × patch latency), the corpus's governing dial — so it inherits P0's 3/3 grounding even though the core/contrib *mechanism* is Envoy-specific (**1/3**).
- Graduates to: `extensibility-boundaries.md` (anchored to `_principles-index.md` P0)

### C37. Hold extensions — and their dependencies — to the core bar, with governed dependency hygiene

- Practice: In-tree extensions meet the same bar as core (style, review, coverage), and their *dependencies* are governed: declared with **CPEs** (for CVE tracking), fetched directly from a git repo with an audit trail, version-pinned, and — for a single non-core extension — isolated as **optional** so they don't bloat the core binary, cleared by dependency shepherds.
- Evidence: **Codified** — `EXTENSION_POLICY.md` ("held to the same quality bar as the core"); `DEPENDENCY_POLICY.md` (CPEs "compulsory"; direct git-archive fetch with audit trail; pinned; "must not substantially increase the binary size unless… confined to specific extensions"; dependency-shepherd clearance).
- Why / constraint: An extension's dependency is attack-surface and supply-chain risk for everyone who builds it in; governing deps *is* part of governing the extension surface, not a separate concern.
- Counter-case: Full CPE/shepherd machinery is sized to a security-critical, widely-deployed proxy. A low-stakes internal tool can track deps far more lightly — calibrate to blast radius (C36 / P0).
- Convergence: **1/3** (Envoy). Dependency-policy rigor recurs in other security-critical projects; not yet formally extracted.
- Graduates to: `extensibility-boundaries.md`

### C38. Govern the extension lifecycle — adding a point and removing an extension are both processes

- Practice: Treat the surface as having a lifecycle. Adding a new extension *point* is deliberate and documented (issue → core change → update the "extending Envoy" catalogue/proto). Removing an extension is a **timed deprecation**: the factory emits a deprecation warning, a 6-month interval runs (extendable for heavily-used extensions), then the code is removed.
- Evidence: **Codified** — `EXTENSION_POLICY.md` §"Removing existing extensions" (factory deprecation warning; 6-month interval; announce; then remove) and §"Adding Extension Points".
- Why / constraint: Extension points are long-lived contracts — adding them carelessly creates permanent maintenance, and removing extensions abruptly breaks users. A governed lifecycle bounds both.
- Counter-case: A timed-deprecation pipeline is for a surface with external users who need migration runway. An internal-only extension point can be added or removed in one PR.
- Convergence: **1/3** (Envoy). Deprecation-window discipline recurs (Kubernetes's API deprecation policy is a strong likely corroborator); not yet extracted.
- Graduates to: `extensibility-boundaries.md`

## 3a. Convergence / cross-corpus position

Extensibility is a **new dimension with Envoy as its sole witness → every candidate is 1/3.** No new graduations. But two candidates are the dimension's *application of already-graduated corpus-wide principles*, which is what grounds the dimension despite the single witness:

| Candidate | Anchors to | Relationship |
|---|---|---|
| **C36** (tier core vs contrib) | **P0** — calibrate rigor to blast radius | core/contrib *is* the governing dial applied to extension governance |
| **C32** (narrow interface + status protocol) | **P2** — design-for-testability via injection seams | the **extension-side face** of the same narrow-interface construct SQLite/FDB/K8s witnessed as *test seams*; P2 owns the shared mechanism, this dimension owns the extension-specific craft — **don't double-count** |

A **second extensibility witness** would lift the Envoy-specific candidates toward 2/3. Strongest options, by how many candidates they'd corroborate at once: **Kubernetes apimachinery** (typed/versioned API → C34, `OWNERS` → C1, API-deprecation policy → C38); **Rust** (RFC governance → C1, `#[stable]`/feature-gates → C35); the **Linux kernel module API** or a **browser/VS Code extension model** (narrow ABI + capability declaration → C32/C35).

## 4. Rejected / over-engineered / do-NOT-generalize

```
- Item:            The full governance stack — core/contrib tiers + status/security metadata +
                   dependency shepherds + CODEOWNERS sponsorship.
- Why not general: Justified by a security-critical proxy with a huge multi-org contributor base
                   and untrusted-traffic exposure. For a small plugin system this is bureaucratic
                   over-engineering. EXTRACT THE PRINCIPLES (narrow contract, named registration,
                   declared maturity, tiered bar) and apply them at the scale of a few interfaces
                   and a README — not a policy suite.
```

```
- Item:            Excluding Wasm extensions from the main repository.
- Why not general: A repo-specific call — Wasm lives behind a version-independent ABI (little value
                   qualifying in-tree) and pulls heavy crate dependencies. The transferable idea is
                   "keep the in-tree surface's dependencies minimal," not "ban Wasm."
```

```
- Item:            "Same quality bar as core" for ALL extensions.
- Why not general: Envoy itself walked this back by adding the contrib tier — direct evidence that a
                   single bar across a large surface is untenable. Tier the bar to blast radius (C36),
                   don't pretend one bar fits every extension.
```

## 5. Open questions / corpus cross-checks

- Every extensibility candidate is **1/3** (Envoy-only). The highest-leverage second witness is **Kubernetes apimachinery** (would corroborate C1, C34, C38 together); **Rust** is next (C1, C35).
- **C2** (backpressure via soft limits + watermarks with hysteresis), seeded from Envoy in the template, is a **robustness** candidate, not extensibility — it belongs to `operational-robustness.md` and would be a fresh 1/3 there (neither Kubernetes nor Erlang/OTP covered flow-control/backpressure). Flag for that dimension, not this one.
- Check **Redis** (the simplicity exemplar, pending) as a deliberate **counter-case** to C1: it resists extension surface entirely (small core) — a different, equally-valid answer to the same decay pressure. That contrast belongs in both this dimension and `implementation-simplicity.md`.
- C32's narrow-interface mechanism is corpus-strong via P2, but its *extension-specific* elaboration (the control protocol) still needs a second extension-centric witness to reach 2/3 on its own terms.
