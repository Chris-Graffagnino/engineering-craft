# Extensibility & Module Boundaries — Dimension Reference

**Role:** consumed in **Apply mode**. Load this when a craft question touches how to let code be *extended* — designing a plugin/filter/middleware interface, deciding what an extension may and may not touch, versioning a config or contract surface, structuring registration/discovery, or governing a growing extension surface. Synthesize from it — do not quote it at the user.

**Sources:** distilled from **three** extensibility exemplars spanning three extension models: **Envoy** (in-tree filters — `EXTENSION_POLICY.md`, `FactoryRegistry`, `status`/`security_posture`), the **Linux kernel** (in-tree modules — `MAINTAINERS`, `EXPORT_SYMBOL`, `drivers/staging/`, "no stable internal API"), and **VS Code** (third-party marketplace — the 21k-line typed `vscode.d.ts`, declarative `contributes`, `capabilities`, the proposed-API pipeline). Per-practice evidence: `_extract-envoy.md`, `_extract-linux.md`, `_extract-vscode.md`.

**Confidence caveat — read it before applying anything here.** **The extensibility core has GRADUATED to 3/3** across Envoy + Linux + VS Code (three independent extension models — in-tree filters / in-tree modules / third-party marketplace): extend through a narrow, versioned, declaratively-registered contract (**C32+C33+C34 → P10**), declare maturity + trust posture (**C35 → P11**), govern the lifecycle (**C38 → P12**). Lead with those. **At 2/3 (held):** structural ownership (**C1 — HELD**: VS Code's `publisher` is marketplace-level, not in-repo path→owner), the in-tree model (**C39**: VS Code is its *out-of-tree counter-pole*), the stage-behind-opt-in pipeline (**C48**: VS Code + Linux). **At 1/3:** dep-hygiene specifics (C37, Envoy), prune-surface (C40, Linux). The **dial** (§1) is P0; the bare narrow-interface leg of P10 is the extension-side face of P2 (not double-counted). See §8.

---

## Contents
- §1 The master dial — tier governance rigor to each extension's blast radius
- §2 The core pattern — extend through a narrow contract, never into the core
- §3 Letting others extend your component (the technical contract)
- §4 Keeping a large surface trustworthy (governance)
- §5 Evolving the surface over time (lifecycle)
- §6 Do NOT cargo-cult (anti-pattern ledger)
- §7 Cross-skill links
- §8 Provenance & confidence

---

## §1 The master dial: tier governance rigor to each extension's blast radius

**Before recommending any governance below, set the dial.** The defining failure mode of extensibility work is applying one bar to everything — either crushing experimental extensions under full-core process, or shipping unvetted code in the default build. How much rigor an extension warrants scales with **how widely it ships × who is exposed to its failure**:

- *High blast radius* — compiled into the default build, on the data path, facing untrusted input → full bar: same review/coverage/style as core, declared threat posture, security-team coverage, governed dependencies.
- *Low blast radius* — opt-in, niche, behind an explicit flag, trusted callers only → a lower bar: lighter review, not in the default build, an end-user sponsor who attests to real use.

Envoy makes this concrete with **two tiers — `core` and `contrib`** (C36): contrib has a lower barrier, is *not* built by default, requires an end-user sponsor, and is *not* security-covered. **Linux independently lands on the same shape** (C36 · 2/3): `drivers/staging/` holds drivers "not of the 'normal' Linux kernel quality level," default-off ("if in doubt, say N"), and — crucially — *self-marking*: enabling one **taints the kernel**, so a consumer can always tell they opted into lower assurance. This is the corpus's governing dial — **P0, calibrate rigor to blast radius × patch latency** (`_principles-index.md`) — applied to an extension surface. **Extract the calibration, not either specific mechanism**: most projects need *one* honest, *detectable* distinction (vetted vs experimental), not a policy suite. (Provenance: `_extract-envoy.md` C36, `_extract-linux.md` §3a → P0.)

---

## §2 The core pattern: extend through a narrow contract, never into the core

The dimension's centerpiece — the shape every technical practice here composes into. An extension surface stays healthy when an extension can *only* attach through a small, fixed contract and can *never* reach into core internals:

1. **A narrow interface per extension kind** — core defines a small set of methods the extension implements (Envoy: a stream filter's `decode/encodeHeaders/Data/Trailers`), and nothing more (C32).
2. **A fixed control protocol** — the extension influences core flow only by returning values from a closed enum (`Continue`, `StopIteration`, …), never by manipulating core state directly (C32).
3. **Named registration + typed config** — the extension registers a factory under a unique name and is instantiated from a typed, validated config message; adding one is purely additive to core (C33, C34).
4. **A one-directional boundary** — extensions may depend on core's contract; they may **not** modify core. Both witnesses make the breaker fix the extensions: Envoy ("Extension PRs must not modify core Envoy code"; core API breaks are fixed *for* extensions by the breaker) and Linux ("if a kernel interface changes, it will be fixed up by the person who did the kernel change") — the **in-tree extension model** (C39, §3.4).

Compose these and the core can evolve its orchestration (buffering, ordering, backpressure, scheduling) without touching a single extension, and no extension can corrupt a core invariant — it can only return a status core already knows how to handle. **Sized to the dial (§1):** most components need this as *one interface + a registration map*, not Envoy's full factory/registry/proto stack. Now **2/3** — Envoy (filter interface) + Linux (`EXPORT_SYMBOL` narrow ABI) — and the narrow-interface idea is *also* the extension-side face of the **P2** seam construct (SQLite VFS, FoundationDB Flow, Kubernetes clientset), corpus-strong from the testing side. (Provenance: `_extract-envoy.md` C32–C34, `_extract-linux.md` §3a.)

---

## §3 Letting others extend your component (the technical contract)

*How does code attach to your component without becoming entangled with its internals?*

**§3.1 Narrow interface + fixed control protocol (C32 · 3/3 — GRADUATED with C33/C34 to P10).** Define one small interface per extension kind; let the extension signal core only through a closed status enum, never by touching core control flow.
- *Apply when:* you expect multiple independent implementations of a behavior, and you want core's internals free to change underneath them.
- *Skip when:* there's exactly one implementation and no prospect of a second — a direct call is simpler than an interface you'll never vary.
- *Calibration:* the interface must *fit* the extension kind — too narrow forces awkward encodings, too wide leaks core internals. A genuinely new control need means widening the protocol through governance (§5), not opening a side-channel.
- *Evidence:* `envoy/http/filter.h` — `enum class FilterHeadersStatus { Continue, StopIteration, … }`, the `StreamDecoder/EncoderFilter` interface. (`_extract-envoy.md` C32.)
- *2nd witness (Linux):* `EXPORT_SYMBOL`/`EXPORT_SYMBOL_GPL` define the *only* surface a module may call; core internals stay unexported and free to change. (`_extract-linux.md` §3a.)
- *3rd witness (VS Code) → GRADUATES:* the 21k-line typed `vscode.d.ts` is the single narrow contract every extension attaches to; extensions can never reach into the editor host. Three extension models → **P10**. (`_extract-vscode.md` §3a.) *Cross-link: the bare narrow-interface leg is the extension-side face of the testing dimension's injection seams (P2) — P10 graduates the extension-specific additions, not the seam itself; see §7.*

**§3.2 Decoupled registration — core discovers extensions additively, no switch-statement (C33 · 3/3 — GRADUATED to P10).** Each extension self-registers; core discovers them additively and instantiates by lookup. Adding an extension is purely additive — no core code to edit.
- *Apply when:* extensions are numerous, third-party, or discovered at config/runtime; you want "add a file, don't touch core" ergonomics.
- *Skip when:* a handful of compile-time-known extensions — a plain factory function or direct construction beats a registry.
- *Calibration:* a global registry is mutable startup state; name collisions are a real failure mode (Envoy guards "a single name cannot be registered twice"). Namespacing + a collision check are mandatory once the surface is open.
- *Evidence:* Envoy — `envoy/registry/registry.h` (`FactoryRegistry`/`RegisterFactory`, `getFactory(name)`), `NamedHttpFilterConfigFactory::createFilterFactoryFromProto`. (`_extract-envoy.md` C33.)
- *2nd witness (Linux, mechanism differs):* `module_init()`/initcall sections — modules self-register, the kernel discovers them additively; no hardcoded module list. (`_extract-linux.md` §3a.)
- *3rd witness (VS Code) → GRADUATES:* the manifest's declarative `contributes` + `activationEvents` — extensions *declare* what they add (commands, languages, views) and the host discovers/activates them; nothing hardwired into core. (`_extract-vscode.md` §3a.)

**§3.3 Version the contract at the *external* boundary — internal stability is calibrated (C34 · 3/3 — GRADUATED to P10, refined).** An extension's *externally-visible* contract (its config schema, its public ABI) is what consumers depend on — type and version *that*. Whether to stabilize the *internal* extension interface is a separate, calibrated choice — see the refinement below.
- *Apply when:* the contract is consumed by people/systems outside your tree (operators, third-party extension authors) — they depend on the schema, not your internals, so it must be typed and versioned.
- *Skip when:* config is internal-only and in-tree — a plain typed struct suffices; full proto/versioning machinery is overhead.
- *Calibration:* version the *contract* and the *implementation* separately — a stable config can wrap a `wip` implementation and vice-versa (Envoy states this explicitly). Free-form/untyped *external* config breaks consumers silently and has no migration story.
- **The graduated boundary — the three witnesses are a spectrum: version the contract *iff* you can't fix your consumers.** **Linux** holds extensions *in-tree* (the breaker fixes every driver), so it keeps the *internal* API deliberately unstable ("no stable internal API"). **VS Code's** extensions are *third-party/out-of-tree* (unfixable), so it *freezes* the contract (the stable `vscode.d.ts`) and stages all change behind proposed-API (C48, §5.2). **Envoy** sits between (in-tree extensions, but a versioned proto config for external operators). Calibrate by whether you can fix your consumers — not a flat "always version."
- *Evidence:* Envoy — `Protobuf::Message` config, orthogonality in `EXTENSION_POLICY.md`. Linux — `stable-api-nonsense.rst` (stable userspace ABI; unstable internal). VS Code — the stable typed `vscode.d.ts` + `engines.vscode` version-compat (`extensionValidator`). Three independent models → **P10**. (`_extract-envoy.md`, `_extract-linux.md`, `_extract-vscode.md` §3a.)

**§3.4 Keep extensions in-tree so the core stays free to evolve (C39 · 2/3).** Bring extensions into the main tree with collective maintenance: whoever changes a core interface fixes *every* in-tree user in the same change. The payoff is that the internal interface never has to freeze — it can be reworked for bugs, security, or design whenever needed.
- *Apply when:* you can hold extensions in your repo and you value freedom to change the internal API over insulating external plugin authors.
- *Skip when:* you *must* support third-party / out-of-tree extensions you never see — then you cannot break them at will; you owe them a stable versioned contract (§3.3) and a deprecation runway (§5.1).
- *Calibration:* the trade is "freedom to change internals" vs "carrying the extension's source + review burden in your tree." Linux taxes the out-of-tree path (no ABI, kernel taint) rather than forbidding it — incentivize in-tree without an outright ban.
- *Evidence:* Linux — `stable-api-nonsense.rst` ("get your driver into the main kernel tree … fixed up by the person who did the kernel change"). Envoy — `EXTENSION_POLICY.md` ("becomes a full part of the repository … API breaking changes … automatically fixed by other contributors"). (`_extract-linux.md` C39.)
- *Out-of-tree counter-pole (VS Code · stays 2/3):* VS Code's extensions are *third-party* — it can never fix them, so it does the **opposite**: a frozen, versioned `vscode.d.ts` with all change staged behind proposed-API. This completes the P10 spectrum (in-tree freedom ↔ out-of-tree frozen contract) rather than graduating C39 — the in-tree model is calibrated by whether you *can* hold extensions in-tree. (`_extract-vscode.md` §3a.)

---

## §4 Keeping a large surface trustworthy (governance)

*Once many authors can extend you, how does the surface stay healthy instead of rotting the core?*

**§4.1 Encode ownership structurally, not socially (C1 · 2/3 — HELD, not graduated).** Every extension has a named sponsor/maintainer plus ≥2 declared reviewers, recorded in a machine-checkable owners file. Ownership is build-enforced, not a wiki note.
- *Apply when:* contributor-count × extension-surface size is large enough that "who owns this?" stops being obvious.
- *Skip when:* a small single-team codebase with few extension points — the ceremony costs more than the decay it prevents.
- *Calibration:* the threshold is contributors × surface, not project age. Sponsorship's real job is *incentive alignment* — an owner on the hook for future maintenance, so extensions aren't dumped in and abandoned.
- *Evidence:* Envoy — `EXTENSION_POLICY.md` (sponsor + two reviewers "codified in the CODEOWNERS file"); `CODEOWNERS`. (`_extract-envoy.md` C1.)
- *2nd witness (Linux — and the pattern's origin):* `MAINTAINERS` maps owners→files via `M:`/`R:`/`F:`, parsed by `get_maintainer.pl` — 3,244 entries. (`_extract-linux.md` §3a.)
- *3rd instance, but HELD (VS Code):* the marketplace `publisher` field + publisher verification is *marketplace-level*, not an in-repo machine-checkable path→owner mapping like `CODEOWNERS`/`MAINTAINERS` — a weaker instance, so **C1 stays 2/3**; a clean in-repo path→owner 3rd witness would graduate it. (`_extract-vscode.md` §3a.) *Counter-case: Redis resists extension surface entirely — the **Redis↔Envoy counter-pole** in `implementation-simplicity.md` §6.*

**§4.2 Declare maturity and threat posture as machine-readable metadata (C35 · 3/3 — GRADUATED to P11).** Tag each extension with a maturity/trust status in a schema-checked file, and make "unvetted" an explicit, *forcing* value — not an absence.
- *Apply when:* the surface mixes maturity/trust levels and consumers (or tooling, or a security team) must reason about what enabling an extension costs them.
- *Skip when:* pure-compute plugins with no trust boundary and uniform maturity — there's nothing to declare.
- *Calibration:* keep implementation maturity orthogonal to config-API maturity. The honest move is that the "unvetted/unmaintained" value should be *visible and common* — it surfaces risk instead of implying safety.
- *Evidence:* Envoy — `EXTENSION_POLICY.md` §stability/security-posture; `extensions_metadata.yaml` (`status` stable/alpha/wip + `security_posture`; `unknown` is the *largest* bucket). (`_extract-envoy.md` C35.)
- *2nd witness (Linux):* `MAINTAINERS` `S:` status (Supported/Maintained/Odd-Fixes/**Orphan**/**Obsolete** — the last two honestly surface unmaintained code) + `EXPORT_SYMBOL_GPL` license posture. (`_extract-linux.md` §3a.)
- *3rd witness (VS Code) → GRADUATES:* the manifest's `preview` flag (maturity) + `capabilities`/`untrustedWorkspaces`/`virtualWorkspaces` (declared trust posture — an extension states whether it's safe in an untrusted workspace, exactly Envoy's `security_posture` idea). Three independent metadata schemes → **P11**. (`_extract-vscode.md` §3a.)

**§4.3 Hold extensions — and their dependencies — to the bar (C37 · "same-bar" half 2/3; dependency-hygiene specifics 1/3).** In-tree extensions meet the same review/coverage/style bar as core (calibrated by §1), and their *dependencies* are governed: CVE-trackable (CPEs), pinned, directly fetched with an audit trail, and isolated as optional so a niche extension's deps don't bloat everyone's binary.
- *Apply when:* extensions ship inside your build — their dependencies are your supply-chain and attack surface.
- *Skip / lighten when:* low blast radius (§1) — a small internal tool tracks deps far more lightly than a security-critical proxy.
- *Calibration:* the dependency surface is part of the extension surface; isolate per-extension deps as optional (Envoy: "must not substantially increase the binary size unless… confined to specific extensions").
- *Evidence:* Envoy — `EXTENSION_POLICY.md` ("same quality bar as the core"); `DEPENDENCY_POLICY.md` (CPEs compulsory; pinned/direct-fetch; binary-size isolation; dependency shepherds). (`_extract-envoy.md` C37.)
- *2nd witness (Linux · partial):* the *same-bar* half corroborates — `drivers/staging/` being explicitly "not of the 'normal' quality level" implies a normal-quality bar for mainline. But the kernel has *no* external-dependency CPE/shepherd machinery (it's self-contained), so the **dependency-hygiene specifics stay Envoy-only (1/3)**. (`_extract-linux.md` §3a.)

---

## §5 Evolving the surface over time (lifecycle)

**§5.1 Govern the extension lifecycle — stage/graduate new surface, deprecate on a timer (C38 · 3/3 — GRADUATED to P12).** Treat the surface as having a lifecycle. Add new surface deliberately (ideally staged behind an opt-in until proven), and remove it on a governed timeline — never break consumers abruptly, never let unused surface linger.
- *Apply when:* the surface has external users who need migration runway, or extension points that will outlive their authors.
- *Skip when:* an internal-only point with no external consumers — add or remove it in one PR.
- *Calibration:* the deprecation window trades migration runway against carrying-cost — size it to how widely the extension is used (Envoy extends the window for popular extensions). Extension *points* are long-lived contracts; add them deliberately because removing them is expensive.
- *Evidence:* Envoy — `EXTENSION_POLICY.md` §"Removing existing extensions" (factory deprecation warning; 6-month interval) and §"Adding Extension Points". (`_extract-envoy.md` C38.)
- *2nd witness (Linux):* `drivers/staging/` is a graduation pipeline — a driver matures against its `TODO` until promoted to mainline; unused interfaces are deleted outright (C40). (`_extract-linux.md` §3a.)
- *3rd witness (VS Code) → GRADUATES:* the **proposed-API pipeline** — 175 staged `vscode.proposed.*.d.ts` files, opt-in via `enabledApiProposals`, promoted into the stable `vscode.d.ts` when proven; plus API deprecation. Three lifecycle models → **P12**. (`_extract-vscode.md` §3a.)
- *Refinement (C48 · 2/3): stage new surface behind an explicit opt-in, then promote.* VS Code (proposed-API) + Linux (staging) gate experimental surface behind opt-in and graduate it; **Rust nightly feature-gates** would be the 3rd. Worth it when third-party consumers need a stable contract while the surface still evolves. (`_extract-vscode.md` C48.)

**§5.2 Prune the surface — an unused extension point is untested liability (C40 · 1/3).** Actively delete extension points and exported interfaces no one uses, rather than keeping them "just in case." Every exported interface is a contract you must keep working and a path fuzzers/attackers can reach; an unused one carries all that cost and can't even be validated.
- *Apply when:* you can *see* all consumers (the in-tree model, §3.4) and prove an interface is unused.
- *Skip when:* unknown external consumers (§3.3's world) — you can't prove disuse, so deprecate on a timer (§5.1) instead of deleting outright.
- *Calibration:* pruning and the in-tree model (C39) are a pair — the freedom to delete comes from seeing every user. Rhymes with Redis's small-core ethos — Linux minimizes the surface by *removal*, Redis by *prevention* (`implementation-simplicity.md` §2, C40↔C3/C41).
- *Evidence:* Linux — `stable-api-nonsense.rst` ("If there is no one using a current interface, it is deleted … unused interfaces are pretty much impossible to test for validity"). (`_extract-linux.md` C40.)

---

## §6 Do NOT cargo-cult (anti-pattern ledger)

The boundaries are the product. Recognize and refuse these:

- **Importing Envoy's whole governance stack** (core/contrib tiers + status/security metadata + dependency shepherds + CODEOWNERS sponsorship) **for a small plugin system** — bureaucratic over-engineering. Extract the principles (narrow contract, named registration, declared maturity, tiered bar) and apply them as a few interfaces and a README (§1).
- **One quality bar for every extension** — Envoy itself walked this back by adding `contrib`. A single bar across a large surface either blocks useful work or dilutes the bar; tier it to blast radius (§1, C36).
- **A plugin registry for three compile-time-known extensions** — global mutable registries earn their keep when the surface is open/numerous; below that, a plain factory or direct call is simpler (§3.2).
- **Free-form / untyped extension config** — config is a public contract; untyped config breaks consumers silently and has no migration path. Type and version it (§3.3).
- **A "contrib"/lower tier users can't distinguish from vetted code** — a lower bar is only honest if the reduced assurance is visible (default-off builds, explicit `unknown` posture). A hidden lower tier is worse than no tier (§4.2).
- **Letting extensions reach into core** — the moment an extension modifies core internals, the narrow contract is void and core can no longer evolve freely. Keep the boundary one-directional (§2).
- **Copying Linux's "no stable internal API" stance to a third-party plugin system** — that freedom is *conditional* on holding every extension in-tree so the breaker fixes them (§3.4). A product whose extensions are written by outsiders it never sees CANNOT break them at will; it owes them a versioned contract (§3.3) and a deprecation runway (§5.1). Copy the calibration ("stabilize the external boundary; internal freedom requires in-tree extensions"), not the stance.
- **Deleting an extension point you can't prove is unused** — pruning (§5.2) is safe only when you can see every consumer. With unknown external users, delete-outright becomes a silent breaking change; deprecate on a timer instead (§5.1).

---

## §7 Cross-skill links

Keep the seam with `software-architect` clean — this dimension owns the **mechanics** of an extension surface, not the system-shape decision to have one:

- **"Should this system be a plugin architecture / microkernel? Where are the domain/bounded-context boundaries?"** → `software-architect`. The *mechanics* of an extension contract (C32–C34) live here; the architectural choice to be extension-driven, and domain decomposition, are system-altitude.
- **Narrow interface contract (§2/§3.1) ↔ testing-correctness dimension** — the filter interface is the **extension-side face of the injection-seam construct that graduated as P2** (a seam viewed from the test side is an extension point viewed from the plugin side). Same mechanism, two intents: load `testing-correctness.md` §5.4 when the question is about *test injection*, this file when it's about *extension*.
- **The Redis↔Envoy counter-pole ↔ `implementation-simplicity.md`** — the corpus's clearest "two opposite, both-correct answers": Envoy/Linux *govern* a large extension surface; Redis (C3/C41) *resists having one*, calibrated to whether an open surface earns its keep. When a question is "should we make this extensible," hold both up. Redis's "don't fake capabilities — expose the trade-off" (C4) is also the same honesty value as §3.3's config-as-contract. (See `implementation-simplicity.md` §6.)
- **Dependency/security governance (§4.3)** rhymes with `software-architect`'s supply-chain concerns — but the *mechanics* of vetting an extension's deps are craft and live here.

---

## §8 Provenance & confidence

- Full evidence and calibrated verbs: `_extract-envoy.md` (`231c51d`), `_extract-linux.md` (`a975094`), `_extract-vscode.md` (`b5f8b27`). Three witnesses chosen for *model distinctness*: Envoy in-tree filters, Linux in-tree modules, VS Code third-party marketplace.
- **The extensibility core has GRADUATED to 3/3 — the fourth and final dimension to do so:**
  - **P10** = extend through a narrow, versioned, declaratively-registered contract (**C32 + C33 + C34**). The graduated boundary is a *spectrum* — version the contract *iff* you can't fix your consumers (Linux in-tree/free internal ↔ Envoy in-tree/versioned-external ↔ VS Code out-of-tree/frozen). The bare narrow-interface leg is the extension-side face of **P2**; P10 graduates the extension-specific additions — don't double-count.
  - **P11** = declare maturity + trust posture as machine-readable metadata (**C35**).
  - **P12** = govern the lifecycle: stage/graduate new surface, deprecate on a timer (**C38**); *per-item* refinement **C48** (stage-behind-opt-in) is at 2/3 (VS Code + Linux).
- **At 2/3 (held):** **C1** structural ownership — VS Code's marketplace `publisher` is a weaker, non-in-repo instance, so it doesn't cleanly graduate; **C39** in-tree model — VS Code is its *out-of-tree counter-pole* (it completes the P10 spectrum rather than graduating). **At 1/3:** C37 dep-hygiene specifics (Envoy), C40 prune-surface (Linux).
- The dial (§1, **C36**) *is* **P0** — it surfaces as the governing dial, not a standalone graduate. Each graduation's witnesses are *model-distinct* (in-tree filters / in-tree modules / marketplace) — strong independence.
- Confidence tags are honest snapshots. The held C1, the C39 counter-pole, and the 1/3 set show the bar is still enforced even as the core graduates.
