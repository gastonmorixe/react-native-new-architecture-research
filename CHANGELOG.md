# Changelog

All notable changes to this book are documented here.

The format is loosely based on [Keep a Changelog][kac]. Versions track the
React Native version the book is **verified against**, not a book-internal
version number. Tags in this repo match the corresponding RN tag (e.g. tag
`v0.86.0-rc.1` on the audit pass that grounded the book in RN
`v0.86.0-rc.1` source).

[kac]: https://keepachangelog.com/en/1.1.0/

Detailed per-chapter audit history lives in each chapter's YAML frontmatter
`taillog` field. The audit workspace (`_verification/`, gitignored) holds
the per-chapter claims, source citations, runnable tests, and reports the
audit produced.

---

## [0.86.0-rc.1] - 2026-05-23

Comprehensive source-grounded audit pass against React Native at upstream
HEAD `b32a6c9e9db` (v0.86.0-rc.1 era), with cross-checks against
`react-native-vision-camera` v5.0.10 (`5a07d7ae`), `nitro` v0.35.7
(`326d3e4b`), and the `facebook/react-native-website` repo (`a263a3ba`).

Net delta: 18 files changed, ~2,287 insertions, ~840 deletions, ~120 new
source-grounded footnotes, full YAML frontmatter on every chapter.

### Added

- YAML frontmatter on all 18 chapter files (chapters 1-16 plus `README.md`
  and `NOTES.md`). Fields: `title`, `chapter`, `created_at`, `updated_at`,
  `rn_version_origin`, `rn_version_current`, `tags`, `audit`, `taillog`.
  Frontmatter schema in `_verification/shared/yaml-frontmatter-schema.md`
  (audit workspace).
- ~120 source-grounded footnotes across all chapters. Each cites a specific
  RN source path with line numbers, a CHANGELOG entry, or an external
  source. Footnote pattern is the existing `[^N]` style.
- Chapter 13 (Vision Camera): new paragraph on v5.0 (April 2026) which
  switched Frame from a `jsi::HostObject` to a Nitro `HybridObject`,
  deleting ~2,970 lines of in-tree JSI code. Cites the v5 PR commit
  (`30a8b5de`).
- Chapter 14 (Nitro): the actual `loadHybridMethods` body inlined from
  running nitrogen 0.35.7 end-to-end on a hand-rolled `Math.nitro.ts`
  spec. 21 generated files were captured for the recheck artifact set.
- Chapter 8 (Performance Analysis): a "Status as of v0.86 (2026)" callout
  clarifying that the chapter's "React 18 concurrent features" framing
  still holds, but RN's actual peer dep is now `react ^19.2.3`.
- This `CHANGELOG.md` and a proper repo `.gitignore`.

### Changed

- React 18 → React 18+/React 19 framing across chapters 1, 3, 8, 9. RN
  moved to React 19 in v0.78 (Feb 2025); current `main` pins
  `"react": "^19.2.3"`. The capability set the book describes is still
  correct, only the version label needed updating.
- Multiple C++/Objective-C++ code samples rewritten to compile against
  the actual RN source:
  - Chapter 2 (JSI): five hard signature errors in JSI snippets, plus one
    pre-existing overload-resolution bug.
  - Chapter 3 (Fabric): `ShadowNode` layout, `ComponentDescriptor` API
    (`cloneProps` `PropsParserContext` arg), `ShadowTree::tryCommit`
    locking, iOS mount path attribution, Yoga integration shape.
  - Chapter 5 (CodeGen): the "generated header" example invented
    `NativeMyModuleSpecJSI : public TurboModule` with virtuals. Real
    codegen emits three distinct shapes (iOS ObjC++ protocol + SpecJSI
    shim, Android Java/Kotlin abstract class + JNI SpecJSI shim, pure
    C++ CRTP `CxxSpec` template), none of which use C++ virtuals to
    enforce the contract. Section rewritten around the real artifacts.
  - Chapter 15 (Tutorial): iOS Promise signature was fabricated
    (`Promise<NSString *> &resolve`). Real codegen uses
    `RCTPromiseResolveBlock` / `RCTPromiseRejectBlock`. Android sections
    updated for `BaseReactPackage` (replacing the now-deprecated
    `TurboReactPackage`), corrected `Native<HasteBasename>Spec`
    abstract-class naming, 6-arg `ReactModuleInfo` constructor, Fabric
    `componentDescriptorProvider` + `<Name>Cls(void)` requirement.
- Chapter 13 (Vision Camera) reframed as a counter-example: VC v4 used
  `requireNativeComponent` (legacy Paper) and `RCTViewManager`, running
  under New Arch only via the interop layer. v5 (April 2026) is the
  proper New Arch implementation; the chapter now reflects both.
- Chapter 14 (Nitro): performance numbers replaced. The fabricated
  "2.5x/2.1x faster, 30% less memory" was swapped for the real published
  benchmark numbers (~16x on `addNumbers`, ~6x on `addStrings` synthetic
  100k-call loops), which Nitro itself flags as extreme cases.
- Chapter 7 (Migration): "Important Note" rewritten. Opt-out was removed
  in v0.82 (released 2025-10-07). Both Android (`newArchEnabled=false`)
  and iOS (`RCT_NEW_ARCH_ENABLED=0`) are now force-overridden by the
  build/install tooling. The C helper `RCTIsNewArchEnabled()` is
  hard-wired to `YES`.
- All 16 chapter H1 headings restored to sequential `Chapter 1`..
  `Chapter 16`, in lockstep with the README's table of contents. Three
  files had off-by-one duplicates pre-audit, and the recheck phase caught
  one regression that re-introduced a duplicate.

### Fixed

- Wrong iOS symbol name: `RCTIsNewArchitectureEnabled` (does not exist in
  RN source) → `RCTIsNewArchEnabled` (declared at
  `packages/react-native/React/Base/RCTUtils.h:18`). Fixed in chapters
  12 (FAQs), 16 (Resources), `NOTES.md`.
- Multiple broken footnote URLs:
  - Chapter 9 (Conclusion): both `[^1]` and `[^2]` returned 404. Replaced
    with verified-200 canonical URLs and split into 7 footnotes.
  - Chapter 12 (FAQs): `[^1]` and `[^5]` were dead.
  - Chapter 14 (Nitro): all six original footnote paths were stale after
    Nitro's docs reorganization into `getting-started/`, `concepts/`,
    `resources/`, `types/`.
- Three `reactnative.dev` URLs in chapter 7 that vercel.json redirects
  off-site to `github.com/reactwg/...`. Replaced with the canonical
  `turbo-native-modules-introduction` and `fabric-native-components-
  introduction` slugs the rest of the book already uses.
- One internal-redirect URL in chapter 12 (FAQs) updated to canonical
  `/architecture/landing-page`.
- Chapter 3 (Turbo Modules) footnote rendering: body referenced `[1]` /
  `[2]` but the citations block used `[^1]` / `[^2]`, so cross-links did
  not render. Normalized to the markdown footnote syntax.
- "Refined" precision on two Nitro chapter claims:
  - "Generic `Repository<T>` won't parse" was too strong. Actual nitrogen
    0.35.7 behavior is to parse it fine and fail at codegen with
    `Error: The TypeScript type "T" cannot be represented in C++!`.
    Reproducible by the audit's toy spec.
  - "TurboModule cannot be cached" was too strong. `TurboModule::get()`
    in `TurboModule.h:50-65` writes the resolved method back onto the JS
    representation via `setProperty`; subsequent accesses skip the native
    callback. The Nitro win is the prototype is installed once *per
    type*, not lazily after first access.
- Vision Camera chapter 13 citations:
  - Path `package/docs/...` was always wrong (`docs/` lives at the repo
    root). Correct path used.
  - Off-by-one line citation: `FrameHostObject.cpp:23` is blank;
    correct line is `:22`.
  - Undercount: `Four such TODO: ... TurboModules comments` → actual is
    7 in v4.7.2's `package/`, plus 2 non-TODO references.
- HostObject destructor language in chapter 10 (Common Mistakes): the
  original said "explicitly unsafe to make any JSI calls from within the
  destructor". The canonical JSI source comment
  (`jsi/jsi/jsi.h:222-233`) is narrower — operations taking a `Runtime&`
  are unsafe; destructing other jsi objects is explicitly allowed.
- TurboModule callbacks vs Promises in chapter 11 (Best Practices). The
  original said callbacks "may be supported by the interop layer", which
  is wrong: a `(arg: T) => void` parameter is a first-class TurboModule
  signature, parsed as `FunctionTypeAnnotation`, generated as
  `jsi::Function`, and used by current core specs like `NativeMicrotasks`
  and `NativeIdleCallbacks`. Promises are still recommended but for
  ergonomics, not because callbacks are unsupported.
- Fabric `updateProps` framing in chapter 11. "A diffing mechanism" was
  misleading. On iOS the method receives both `props` and `oldProps` and
  the view does the diff by hand. On Android the wrapper is a pre-
  filtered `ReactStylesDiffMap` containing only changed keys.

### Removed

- Three hallucinated Nitro features that appeared nowhere in Nitro's
  docs or source: "Hot Reload Support for native code", "IDE
  Integration", "Better Error Messages with stack traces". Each was
  removed and the chapter now documents what Nitro actually does.

### Infrastructure

- Added `.gitignore` covering: OS metadata (`.DS_Store`), editor configs
  (`.vscode/`, `.idea/`), audit workspace (`_verification/`), minimal-
  agent debug dumps (`.net-dbg/`), lockfiles (`bun.lock`, `*.lock`),
  common build outputs and caches, env files.
- Established two long-lived audit branches:
  - `audit/baseline-2026-05-23` — pre-audit snapshot of all in-progress
    edits (preserved as a safety net; do not delete unless rolling back
    is no longer needed).
  - `audit/verified-2026-05-23` — the audit result (this PR's branch).
- Tagged `v0.86.0-rc.1` on the audit HEAD to indicate the RN version the
  book is verified against.

### Methodology

The audit was carried out by 21 parallel `minimal-agent --effort max`
workers (Claude Opus 4-7) inside a shared tmux session `ma-rn-arch-audit`,
managed by a coordinator session. Each worker owned one chapter (Phase 2,
18 workers) or one external repo cross-check (Phase 5, 3 workers).

- Phase 0: workspace + baseline branch setup.
- Phase 1: triage (chapter heading inventory).
- Phase 2: 18 workers, one per chapter, ~36 min wall clock. Each worker
  extracted claims, verified against RN source + CHANGELOG, wrote and
  ran tests on every code sample, then edited the chapter.
- Phase 3: manager-level cross-chapter consistency pass (fixed 4 H1
  numbering bugs).
- Phase 4: master delta report + first audit commit.
- Phase 5: 3 specialized workers re-checking VC, Nitro, and docs URLs
  against the synced external repos. Manager applied docs-URL fixes
  and caught one chapter numbering regression.

Tests written and executed during the audit include: `tsc --strict`
type-checks on chapter TS specs, Mermaid syntax validation on all
embedded diagrams, RN CodeGen round-trip on chapter examples (chapter 4),
nitrogen 0.35.7 round-trip on a toy spec (chapter 14).

---

## [0.81.4] - 2025-09-22

Initial bootstrap of the book. Closest stable RN release at the time was
v0.81.4 (released 2025-09-10).

### Added

- A 16-chapter deep-dive report on the React Native New Architecture
  authored from a combination of the official RN source, RN blog posts,
  community libraries (Vision Camera, Reanimated), and Nitro framework
  docs.
- Initial structure:
  - **Part 1 — Core Architecture**: Introduction (the "why"), JSI, Fabric,
    TurboModules, CodeGen, Bridgeless.
  - **Part 2 — Practical Application & Guides**: Migration & adoption,
    performance analysis, conclusion & future, common mistakes, best
    practices, FAQs.
  - **Part 3 — Real-World Examples & Tooling**: Vision Camera case study,
    Nitro alternative tooling, hands-on tutorial, additional resources
    & checklists.
- Mermaid diagrams for Bridge vs JSI call paths, Fabric's render
  pipeline, TurboModule + CodeGen pipeline, Bridgeless interop,
  migration flow.
- `LICENSE` (MIT), `README.md` table of contents, `NOTES.md` author
  logbook.
