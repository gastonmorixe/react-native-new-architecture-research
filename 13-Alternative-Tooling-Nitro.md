---
title: "Alternative tooling: Nitro"
chapter: "13"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T19:58:00-0400"
rn_version_origin:
  version: "0.81.4"
  tag: "v0.81.4"
  date: "2025-09-10"
  note: "Closest stable release at the chapter's original bootstrap date (2025-09-22)."
rn_version_current:
  version: "0.86.0-rc.1+main"
  tag: "v0.86.0-rc.1"
  commit: "b32a6c9e9db2284547183e2d48ffa0a45a73fbc6"
  date: "2026-05-22"
  note: "Upstream RN main HEAD as of the audit. Past v0.86.0-rc.1 by ~30 commits."
tags: [react-native, new-architecture, nitro, hybrid-objects, jsi, third-party]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/13-nitro/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:13:44-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db) and Nitro v0.35.7 (commit 326d3e4b). Fixed chapter number, corrected fabricated benchmark / hot-reload / IDE claims, replaced with Nitro's own published numbers and accurate feature list, fixed stale doc-path footnotes, added RN source citations for NativeState / HostObject / TurboModule. See _verification/chapters/13-nitro/report.md."
  - "2026-05-23T19:58:00-0400 | Second-pass recheck against actual Nitro source at /tmp/nitro (commit 326d3e4b, v0.35.7). Ran nitrogen 0.35.7 on a toy spec (`tests/toy-spec/Math.nitro.ts`) and confirmed: 21 files emitted, prototype-based dispatch in generated .cpp, three-piece Swift pattern. Replaced 'will not parse' with the actual nitrogen error for generic specs. Added the Nitro-bootstraps-as-a-TurboModule note. Softened the 'HostObject can't be cached by JS Runtime' framing because TurboModule.h:55-65 does set up post-first-call setProperty caching. Documented C++/Swift/Kotlin platform-language pairs and the `nitro.json` autolinking requirement. Citations now include `tmp/nitro/...` source paths. See _verification/recheck/nitro/report.md."
---

# Chapter 14: Alternative Tooling - Nitro

While the standard New Architecture tools (CodeGen, TurboModules, Fabric) provide the fundamental building blocks, the ecosystem is evolving with higher-level tooling designed to simplify the developer experience and push performance even further. One prominent example of this is **Nitro**.

Nitro is an opinionated, high-performance framework for building native modules that sits on top of the JSI, acting as an alternative and enhancement to the standard TurboModule workflow.[^1] At the time of this audit, the current stable release is **Nitro v0.35.7** (May 2026) and the documented minimum React Native version is **0.75**.[^7] (The package's own `peerDependencies` are wildcarded (`react-native: "*"`), so the 0.75 floor is enforced by the iOS/Android build requirements rather than by npm. Internal development of v0.35.7 tracks React Native 0.83.0 via the package's devDependencies.)

**Status (2026):** Nitro is production-ready and has broad adoption in the ecosystem. Libraries built on top of it include `react-native-vision-camera`, `react-native-mmkv`, `react-native-quick-crypto`, `@rive-app/react-native`, `react-native-video`, `react-native-fast-tflite`, `react-native-nitro-sqlite`, and dozens of others.[^8]

## Core Concepts of Nitro

Nitro's philosophy is centered around providing a truly object-oriented and type-safe bridge between JavaScript and native code.

### The `HybridObject`

The central abstraction in Nitro is the `HybridObject`. While a standard TurboModule is effectively a singleton (the TurboModuleManager caches one instance per module name and keeps it for the lifetime of the runtime[^9]), a `HybridObject` is a true, instantiable native object that can be created, passed around, and referenced from JavaScript.[^2] This enables a more object-oriented programming model.

For example, instead of passing an ID or a file path to a native function, you can pass an actual native object instance:

```typescript
// With a TurboModule, you might do this:
const processedImagePath = await ImageEditor.crop(imagePath, rect);

// With Nitro's HybridObjects, you can do this:
const image = await ImageFactory.loadImage(imagePath);
const processedImage: Image = ImageEditor.crop(image, rect);
```

In practice, a JS consumer creates the root Hybrid Object via `NitroModules.createHybridObject<T>('Name')`, and a Hybrid Object method can return another Hybrid Object directly. The JS-side `image` variable holds a real reference to the underlying native object, so subsequent native calls can operate on it without re-serializing through a path or ID.[^2] On the C++ side, the `HybridObjectRegistry` stores *constructor functions* keyed by name, not instances (`std::unordered_map<std::string, std::function<std::shared_ptr<HybridObject>()>>`), so `createHybridObject<T>(name)` calls the registered constructor each time and returns a fresh native object.[^23]

Note that Nitro itself is bootstrapped *as* a TurboModule: `react-native-nitro-modules` ships a single TurboModule named `NitroModules` whose only method is `install()`, which installs the `NitroModulesProxy` Hybrid Object onto JS `global`. After that one-time install, all `NitroModules.createHybridObject<T>(...)` calls and every method on every Hybrid Object go through `global.NitroModulesProxy` directly via JSI, not through TurboModule machinery.[^24] So Nitro doesn't replace TurboModules at the React Native loader level; it bootstraps via a TurboModule shim and then runs above JSI.

### The `nitrogen` Code Generator

Nitro comes with its own code generator, `nitrogen`. It serves the same purpose as React Native's built-in `codegen` (reading a TypeScript spec and generating native interface code) but with a key philosophical difference:[^3]

- **Standard `codegen`:** Runs as part of the **app's** build process. On iOS, `run_codegen!` is invoked from CocoaPods when the user runs `pod install`.[^10] On Android, the Gradle plugin wires up `GenerateCodegenArtifactsTask` for both the app project and each library project.[^11] The app rebuilds every library's generated interfaces from scratch.
- **`nitrogen`:** Is designed to be run by the **library author**. The generated C++, Swift, and Kotlin interface code is committed to the library's repository and published with the npm package, so the consumer of the library never runs nitrogen themselves.[^3]

A practical consequence: Nitrogen can also resolve imports across files (so a spec can `import type` from a sibling), whereas standard codegen cannot.[^4]

What nitrogen actually emits, concretely: an 8-line `Math.nitro.ts` with one `readonly pi: number` getter and one `add(a: number, b: number): number` method, configured for `{ ios: 'swift', android: 'kotlin' }`, produces 21 files across three platforms in `nitrogen/generated/`.[^25] The shared `HybridMathSpec.cpp` registers the prototype:[^22]

```cpp
void HybridMathSpec::loadHybridMethods() {
  HybridObject::loadHybridMethods();
  registerHybrids(this, [](Prototype& prototype) {
    prototype.registerHybridGetter("pi", &HybridMathSpec::getPi);
    prototype.registerHybridMethod("add", &HybridMathSpec::add);
  });
}
```

The Swift side comes in three pieces (`HybridMathSpec_protocol`, `HybridMathSpec_base`, and a `HybridMathSpec_cxx` bridge wrapper, glued together with a public typealias `HybridMathSpec = HybridMathSpec_protocol & HybridMathSpec_base`), and the Kotlin side is an abstract class annotated with `@DoNotStrip` / `@Keep` plus a JNI `HybridData` `CxxPart`. Each Hybrid Object also needs an `autolinking` entry in `nitro.json` that pins the implementation class name per platform, otherwise the generated `*Autolinking.swift`/`*OnLoad.kt` won't include it and the runtime registry can't find the constructor at startup.[^26]

## Key Differentiators and Performance Claims

Nitro makes several claims about its advantages over the standard TurboModule system, backed by technical choices and published benchmarks.

**1. Performance**

Nitro's documentation publishes a microbenchmark that measures the total execution time of calling a single native method 100,000 times in a tight loop:[^4]

| Operation                       | ExpoModules | TurboModules | NitroModules |
| ------------------------------- | ----------- | ------------ | ------------ |
| `addNumbers(...)` × 100,000     | 434.85 ms   | 115.86 ms    | **7.27 ms**  |
| `addStrings(...)`  × 100,000    | 429.53 ms   | 179.02 ms    | **29.94 ms** |

That puts Nitro at roughly **16x faster than TurboModules on `addNumbers` and ~6x faster on `addStrings`** in the synthetic benchmark.[^4] Nitro's own docs are explicit that the numbers reflect "extreme cases" and "do not necessarily reflect real world use-cases". For typical app workloads the gap is much smaller.[^4] The benchmark itself is reproducible inside Nitro's own example app: `example/src/screens/BenchmarksScreen.tsx` runs `addNumbers(...)` 100,000 times against both an `ExampleTurboModule` and `HybridTestObjectSwiftKotlin` with a single warmup call and `performance.now()` around the loop.[^27] Nitro attributes the difference to two architectural decisions:

- **`jsi::NativeState` vs. `jsi::HostObject`:** Standard TurboModules are implemented using `jsi::HostObject`. The RN source confirms this directly: `class JSI_EXPORT TurboModule : public jsi::HostObject` (TurboModule.h:46).[^12] Nitro's `HybridObject`s are built on top of `jsi::NativeState` (jsi.h:255).[^13] With `HostObject`, the *first* access to each property goes through the virtual `get()` callback in C++. TurboModule's `get()` does add post-first-call caching by writing the resolved method onto the JS representation via `setProperty` (TurboModule.h:55-65),[^21] but the cost of the first call and the indirection through `MethodMetadata.invoker` are still there. With `NativeState`, nitrogen-generated C++ installs concrete getters and methods on a shared `Prototype` once per type (`prototype.registerHybridGetter(...)`, `prototype.registerHybridMethod(...)`),[^22] before any JS call. The prototype is cached per `jsi::Runtime` and assigned to every Hybrid Object instance, so JS-engine inline caches can lock onto the slot the first time a property is read. Nitro pairs this with `Object::setExternalMemoryPressure` (jsi.h:1422-1430) to inform the GC about the native side's memory footprint, helping the collector free unused Hybrid Objects more aggressively.[^4]
- **Direct Swift & C++ Interop:** For iOS, Nitro requires Swift 5.9 or higher and uses the official [Swift `<>` C++ interop](https://www.swift.org/documentation/cxx-interop/).[^7] This lets the C++ JSI layer call Swift code directly. The call path is `JS → C++ → Swift`, whereas TurboModules go `JS → C++ → Objective-C → Swift` when the underlying implementation is Swift.[^4]

**2. Richer Type Support**

Nitro's tooling supports a wider range of types than standard codegen. The differences that show up in Nitro's published comparison table (and that the RN codegen source confirms it lacks) are:[^4]

- **Tuples** (`[A, B, C, ...]`): Nitro supports them. RN codegen does not (the codebase carries an explicit TODO for `TupleTypeAnnotation` support[^14]).
- **Variants of distinct types** (`A | B | Person`): Nitro supports them via its [variants](https://nitro.margelo.com/docs/types/variants) machinery. RN codegen's `emitUnion` only accepts unions of primitives (`string`, `number`, `boolean`, and their literal forms, plus a single `ObjectTypeAnnotation` or alias),[^15] so an arbitrary cross-type union is rejected.
- **Callbacks that return values** (`(T...) => R`): Nitro supports both `Promise<T>` (async, the default) and `Sync<T>` (synchronous) callback return types.[^16] In RN codegen, callbacks are restricted: the sample TurboModule spec only declares `(callback: (value: string) => void) => void`,[^17] and `throwIfUnsupportedFunctionReturnTypeAnnotationParserError` rejects function-typed return values outside the cxx-only path.[^18]
- **`null` as a return type**, **`Int64`/`UInt64`**, **`ArrayBuffer`**, **`Record<string, T>`**: all supported by Nitro, none directly supported by RN codegen.[^4]

For literal-string enums like `type MediaType = 'image' | 'video' | 'audio'`, Nitro's nitrogen actually maps them to native enums backed by compile-time string hashes (not variants), and the union must be aliased to a named type for nitrogen to generate code for it.[^19]

## Latest Nitro Features (as of v0.35.7, May 2026)

### Type-system surface

The "type-safety" benefits in Nitro come from nitrogen's TypeScript-to-native code generation: if a spec declares `add(a: number, b: number): number`, the Swift / Kotlin / C++ implementation must use the exact corresponding types or the project will not compile.[^1] In practice this means:

```typescript
// String-literal union (must be a named alias so nitrogen can emit a native enum)
type MediaType = 'image' | 'video' | 'audio';

// Variant of distinct types (each branch is a real type, not a literal)
type ImageOrCount = HybridImage | number;

// Sync callback with a return value (default callbacks return Promise<T>)
interface Server extends HybridObject<{ ios: 'swift' }> {
  fetchWithFilter(filter: Sync<(item: Item) => boolean>): Item[];
}
```

Note that nitrogen does **not** support user-defined generic interfaces as Hybrid Object specs. The generic parameter on `HybridObject<{ ... }>` is reserved for the `PlatformSpec`, which has shape `{ ios?: 'c++' | 'swift'; android?: 'c++' | 'kotlin' }` (both keys optional, so a spec can target only iOS or only Android).[^20] A spec like `interface Repository<T> extends HybridObject<{ ... }>` parses fine on the TypeScript side, and nitrogen even recognizes it as a Hybrid Object, but it fails at code generation because `T` has no native representation. Running nitrogen 0.35.7 against that spec emits:

```
⏳  Parsing Repository.nitro.ts...
    ⚙️   Generating specs for HybridObject "Repository"...
        ❌  Failed to generate spec for Repository! Error: The TypeScript type "T" cannot be represented in C++!
    ❌  No specs found in Repository.nitro.ts!
```

(Reproducible run is in `_verification/recheck/nitro/tests/toy-spec/`.) The only generics in nitrogen's surface area are the built-in ones (`Promise<T>`, `Sync<T>`, `Array<T>`, `Record<string, T>`, `Variant<...>`, etc.).

### Developer experience

- **Compile-time type safety across the bridge.** Because every Hybrid Object spec is the single source of truth, a wrong type or missing method on the native side breaks the build, not the runtime.
- **TypeScript IntelliSense in the editor.** Generated `*.d.ts` declarations give the consumer auto-complete and type-check via the standard TypeScript Language Server. This is not a Nitro-specific feature (TurboModule specs are also TypeScript and benefit from the same tooling), but it is a real consequence of nitrogen shipping generated specs.
- **JS code reload still works as usual.** React Native's Fast Refresh applies to the JS layer that calls a Hybrid Object. Changes to the **native** implementation, however, still require a full app rebuild, just as with TurboModules.

## Conclusion on Nitro

Nitro is not a replacement for the New Architecture. It is an opinionated layer on top of JSI that replaces the **TurboModule slot specifically**. Fabric, the renderer, the runtime, the runtime scheduler, the event loop, and the rest of the New Architecture stack are untouched. A Nitro Module sits next to TurboModules in the same app and uses the same JSI runtime, the same `CallInvoker`, and the same Hermes (or JSC) instance.

For developers seeking very high throughput on JS-native calls, an object-oriented surface for native instances, or a Swift-first iOS development workflow, Nitro is a credible alternative to the standard TurboModule creation process. For most app code, where the JS-native call rate is far below 100,000 calls per second, the architectural ergonomics (Hybrid Objects, typed callbacks with return values, generated specs shipped in npm packages) are usually a stronger reason to adopt Nitro than the raw microbenchmark numbers.

---

**Citations:**

[^1]: Nitro docs: `docs/docs/getting-started/what-is-nitro.md`. <https://nitro.margelo.com/docs/getting-started/what-is-nitro>
[^2]: Nitro docs: `docs/docs/concepts/hybrid-objects.md` (`createHybridObject<T>`, lines 42-71).
[^3]: Nitro docs: `docs/docs/concepts/nitrogen.md` (lines 43-47: "Nitrogen should be used by library-authors").
[^4]: Nitro docs: `docs/docs/resources/comparison.md` (benchmark table at lines 13-38, type-support matrix at lines 574-703, HostObject vs NativeState discussion at lines 255-260).
[^5]: Nitro docs: `docs/docs/getting-started/what-is-nitro.md#uses-jsinativestate` (Docusaurus auto-slug).
[^6]: Nitro docs: `docs/docs/types/custom-types.md`.
[^7]: Nitro docs: `docs/docs/getting-started/minimum-requirements.md` (RN 0.75+, Xcode 16.4+, Swift 5.9+ on iOS; compileSdk 34+, ndk 27+ on Android).
[^8]: Nitro docs: `docs/docs/resources/awesome-nitro-modules.md` (curated list of community Nitro Modules).
[^9]: RN source: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModuleManager.mm:202-203, 317-318` ("All TurboModule instances are cached, which means they're all long-lived") and `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModuleBinding.cpp:185-186` ("TurboModules are cached by name in TurboModuleManagers").
[^10]: RN source: `packages/react-native/scripts/react_native_pods.rb:223` (`run_codegen!` invoked from the Podfile install).
[^11]: RN source: `packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/ReactPlugin.kt:111-120` (`configureCodegen(... isLibrary = false)` for the app, `isLibrary = true` for each library project).
[^12]: RN source: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h:46` (`class JSI_EXPORT TurboModule : public jsi::HostObject`).
[^13]: RN source: `packages/react-native/ReactCommon/jsi/jsi/jsi.h:253-258` (definition of `class JSI_EXPORT NativeState`) and `:1389-1404` (`hasNativeState` / `getNativeState` / `setNativeState` on `jsi::Object`).
[^14]: RN source: `packages/react-native-codegen/src/parsers/error-utils.js:282` (open TODO: "Added as a work-around for now until TupleTypeAnnotation are fully supported in both flow and TS").
[^15]: RN source: `packages/react-native-codegen/src/parsers/parsers-primitives.js:438-490` (`emitUnion` accepts only `StringTypeAnnotation`, `NumberTypeAnnotation`, `BooleanTypeAnnotation`, their literal forms, `ObjectTypeAnnotation`, and `TypeAliasTypeAnnotation`).
[^16]: Nitro docs: `docs/docs/types/callbacks.md:144-167` (`Sync<T>` for synchronous return values; default callbacks return `Promise<T>`).
[^17]: RN source: `packages/react-native/src/private/specs_DEPRECATED/modules/NativeSampleTurboModule.js:54` (`getValueWithCallback: (callback: (value: string) => void) => void`).
[^18]: RN source: `packages/react-native-codegen/src/parsers/error-utils.js:128-142` (`throwIfUnsupportedFunctionReturnTypeAnnotationParserError`).
[^19]: Nitro docs: `docs/docs/types/custom-enums.md` ("TypeScript union" section: a union of literal strings is mapped to a native enum backed by compile-time string hashes, and the union must be aliased to a named type).
[^20]: Nitro source: `tmp/nitro/packages/react-native-nitro-modules/src/HybridObject.ts:7-33` (`interface PlatformSpec { ios?: 'c++' | 'swift'; android?: 'c++' | 'kotlin' }`, both keys optional, used as the constraint on `HybridObject<Platforms extends PlatformSpec>`) and `tmp/nitro/packages/nitrogen/src/getPlatformSpecs.ts:169-180` (nitrogen reads `base.getTypeArguments()[0]` as the platform spec). The C++ runtime side at `tmp/nitro/packages/react-native-nitro-modules/cpp/core/HybridObject.hpp:27` is a non-template class (`class HybridObject : public virtual jsi::NativeState, public HybridObjectPrototype, ...`), confirming that the only "generic" surface is the TS-side `Platforms` argument.
[^21]: RN source: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h:50-65` (`get(...)` calls `create(...)` and, on hit, runs `jsRepresentation_->lock(runtime).asObject(runtime).setProperty(runtime, propName, prop)`. First access goes through the virtual `get()`; subsequent accesses hit the cached JS property).
[^22]: Nitro source: `tmp/nitro/packages/react-native-nitro-modules/cpp/prototype/HybridObjectPrototype.hpp:26-30` ("The prototype should be cached per Runtime, and can be assigned to multiple jsi::Objects. When assigned to a jsi::Object, all methods of this prototype can be called on that jsi::Object, as long as it has a valid NativeState (`this`)."). Generated call sites: `_verification/recheck/nitro/tests/toy-spec/nitrogen/generated/shared/c++/HybridMathSpec.cpp:12-19` (`prototype.registerHybridGetter("pi", ...)`; `prototype.registerHybridMethod("add", ...)`).
[^23]: Nitro source: `tmp/nitro/packages/react-native-nitro-modules/cpp/registry/HybridObjectRegistry.hpp:20-46` (`using HybridObjectConstructorFn = std::function<std::shared_ptr<HybridObject>()>; static void registerHybridObjectConstructor(const std::string& hybridObjectName, HybridObjectConstructorFn&& constructorFn); static std::shared_ptr<HybridObject> createHybridObject(const std::string& hybridObjectName);`). The registry stores constructors, not instances.
[^24]: Nitro source: `tmp/nitro/packages/react-native-nitro-modules/src/turbomodule/NativeNitroModules.ts:1-56` (`import { TurboModuleRegistry } from 'react-native'`; `interface Spec extends TurboModule { install(): string | undefined }`; `turboModule = TurboModuleRegistry.getEnforcing<Spec>('NitroModules'); const errorMessage = turboModule.install(); ... nitroModules = getInstalledNitro()` where `getInstalledNitro` reads `global.NitroModulesProxy`). On old-arch iOS: `tmp/nitro/packages/react-native-nitro-modules/ios/turbomodule/NativeNitroModules+OldArch.mm:34` (`RCT_EXPORT_BLOCKING_SYNCHRONOUS_METHOD(install)`).
[^25]: Reproducible: `_verification/recheck/nitro/tests/toy-spec/` contains `Math.nitro.ts`, `nitro.json`, and `nitrogen/generated/` with 21 files (2 shared C++, 2 iOS C++ bridge, 2 Swift, 6 iOS autolinking/umbrella, 2 Android C++ JNI, 2 Kotlin, 4 Android autolinking/loader, 1 `.gitattributes`). Nitrogen 0.35.7 produced these in 0.9s; see `run.log` next to the spec.
[^26]: Nitro source: `tmp/nitro/packages/react-native-nitro-test/nitro.json` is the canonical example. Each Hybrid Object has an `autolinking` entry mapping `ios.implementationClassName` and `android.implementationClassName` to the concrete native classes that satisfy the generated spec.
[^27]: Nitro source: `tmp/nitro/example/src/screens/BenchmarksScreen.tsx:42-65` (`function benchmark(obj: BenchmarkableObject): number { obj.addNumbers(0, 3); const start = performance.now(); let num = 0; for (let i = 0; i < ITERATIONS; i++) { num = obj.addNumbers(num, 3); } const end = performance.now(); return end - start; }` with `ITERATIONS = 100_000`). The same shape is published at `mrousavy/NitroBenchmarks` for cross-environment runs.
