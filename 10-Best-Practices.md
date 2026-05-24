---
title: "Best practices"
chapter: "10"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:16:17-04:00"
session_id: "audit-worker-10-best-practices"
host_info:
  hostname: "macbookpro.home.arpa"
  user: "gaston"
  os: "macOS 26.5 (..)"
  kernel: "25.5.0"
  arch: "arm64"
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
tags: [react-native, new-architecture, best-practices, turbo-modules, fabric, jsi, codegen]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/10-best-practices/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:16:17-04:00 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Fixed two factually wrong bullets (callback support, updateProps framing), tightened the CallInvoker thread-naming, added pure-C++ TurboModule and view-recycle notes, and added source-citation footnotes. See _verification/chapters/10-best-practices/report.md."
---

# Chapter 11: Best Practices

Adopting the New Architecture effectively involves more than just avoiding mistakes; it requires embracing new patterns and best practices. This chapter provides a list of recommendations for writing high-quality, performant code.

## General Practices

-   **Do** migrate one module or component at a time. The New Architecture is designed for incremental adoption. While your app is on the new arch, React Native still ships an interop layer that lets legacy `RCTBridgeModule` modules and `RCTViewManager`-style components keep working until you port them.[^4] This allows you to isolate issues and learn the new patterns in a controlled way.
-   **Don't** attempt a "big bang" migration of a large, complex application all at once. This makes debugging extremely difficult.
-   **Do** keep your JavaScript specs as the single source of truth. All native implementations should strictly follow the CodeGen-generated interfaces. The generated `*Spec` base classes enforce signatures at compile time, so a drifted method will fail to build rather than fail at runtime.[^5]
-   **Do** audit your third-party dependencies early and often. Prefer libraries that explicitly state support for the New Architecture.
-   **Do** consider higher-level tooling where appropriate. For teams that want to simplify the module creation process or extract maximum performance, alternative frameworks like **Nitro** (see Chapter 14) provide abstractions on top of the standard tools and may be a good fit.

## JSI and C++ Practices

-   **Do** use C++ for performance-critical, cross-platform logic. The JSI makes it easier than ever to write a function once in C++ and use it from both iOS and Android, reducing code duplication and improving performance. The modern way to expose that C++ to JS is a **Pure C++ TurboModule**: a spec that generates a `Native<Name>CxxSpec<T>` base class, linked into both platforms from a single `.cpp`. React Native itself ships several this way, including `NativeMicrotasks` and `NativeIdleCallbacks`.[^6]
-   **Don't** put platform-specific code (e.g., `UIKit` or Android SDK calls) in your core C++ modules. Keep the C++ layer platform-agnostic and use platform-specific adapter code to call into it.
-   **Do** handle threading carefully. The JS thread is *not* the UI/main thread under the New Architecture. If a TurboModule call needs to perform work off the JS thread, dispatch it to a background queue, then use the `CallInvoker` (specifically `invokeAsync(CallFunc&&)`) to deliver the result back to the JS thread, where it is safe to touch the `jsi::Runtime` again.[^7]

## TurboModule Practices

-   **Do** keep synchronous methods trivial and fast. They are perfect for fetching simple, pre-calculated values that are already in memory.[^1]
-   **Don't** ever perform I/O (network, disk), heavy computation, or any potentially long-running task in a synchronous TurboModule method. A sync method is just a `jsi::Value(jsi::Runtime&, ...)` C++ function invoked inline on the JS thread, so anything that blocks inside it blocks JS execution and freezes the app.[^1][^8]
-   **Do** use Promises for any method that is asynchronous or might take more than a millisecond to complete. This is the standard and expected pattern for async work, and `Promise<T>` is a first-class return shape in CodeGen.[^1][^9]
-   **Do** prefer Promises over callbacks for new async methods. Callbacks are *not* legacy though: a `(arg: T) => void` parameter is a first-class TurboModule signature, generated as a `jsi::Function` and supported in both bridged and pure-C++ modules (see `NativeIdleCallbacks` and `NativeMicrotasks` in core RN for examples).[^10] Promises are preferred because they integrate cleanly with `async/await`, have a single resolution path, and unify error handling, not because callbacks "only work via interop".

## Fabric Component Practices

-   **Do** treat `updateProps` as the place to apply changed properties to your native view. The method itself isn't an automatic diff: on iOS, the protocol gives you both the new `Props::Shared` and the previous `oldProps`, and the component view casts them and compares fields by hand (`if (oldViewProps.opacity != newViewProps.opacity) { ... }` is the canonical pattern in `RCTViewComponentView`). On Android the wrapper passed in is a `ReactStylesDiffMap`, which only contains the keys that actually changed. Either way the rule is the same: apply only what differs.[^2][^11]
-   **Don't** perform expensive allocations or setup work inside `updateProps`. Place one-time setup code in the view's native initializer, and reset any per-mount state in `prepareForRecycle`, because Fabric recycles component views and the initializer only runs once per allocated view, not per mount.[^2][^12]
-   **Do** use commands for one-off, imperative actions. If you need to trigger an action on a native view (e.g., `focus()`, `clear()`), defining it as a command is much more efficient than changing a prop to trigger the effect. A command is a single JS-to-native dispatch (`dispatchCommand` → `handleCommand:args:`), while a prop change runs the full shadow-tree update + mount-transaction + `updateProps` pipeline.[^3][^13]
-   **Don't** use props for ephemeral, event-like triggers. For example, to trigger an animation, it's better to use a command like `myView.startAnimation()` than to toggle a prop like `shouldAnimate={true}`.[^3]
-   **Do** colocate your component's related files. A good pattern is to have a single directory containing the JS spec, the iOS implementation (`.mm`), and the Android implementation (`.java`/`.kt`) for a single component. React Native itself is migrating its own first-party modules to this layout under `packages/react-native/src/private/`, with paired native code in `ReactCommon/react/nativemodule/`.[^2][^14]

---

**Citations:**

[^1]: "Native Modules" (formerly titled "Turbo Native Modules"). React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction)
[^2]: "Fabric Native Components". React Native Documentation. [https://reactnative.dev/docs/next/fabric-native-components-introduction](https://reactnative.dev/docs/next/fabric-native-components-introduction)
[^3]: "Native Commands". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/fabric-component-native-commands](https://reactnative.dev/docs/next/the-new-architecture/fabric-component-native-commands)
[^4]: Interop entry points: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTInteropTurboModule.h` (legacy NM shim), `packages/react-native/ReactCommon/react/renderer/components/legacyviewmanagerinterop/` (iOS view-manager shim), and `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/fabric/interop/` (Android counterpart). React Native HEAD `b32a6c9e9db` (RN 0.86-era).
[^5]: CodeGen module pipeline: `packages/react-native-codegen/src/parsers/parsers-commons.js:328-425` (parsing) and `packages/react-native-codegen/src/generators/modules/GenerateModuleH.js` (generation). Generated specs become abstract C++ base classes that the native implementation `override`s.
[^6]: Pure C++ TurboModules in RN itself: `packages/react-native/ReactCommon/react/nativemodule/microtasks/NativeMicrotasks.{h,cpp}` and `packages/react-native/ReactCommon/react/nativemodule/idlecallbacks/`. Each one extends a `Native<Name>CxxSpec<Impl>` base generated from the JS spec under `packages/react-native/src/private/webapis/.../specs/`.
[^7]: `CallInvoker` definition: `packages/react-native/ReactCommon/callinvoker/ReactCommon/CallInvoker.h:27-50`. `invokeAsync(CallFunc&&)` schedules a callback that receives a `jsi::Runtime&` argument, which is the contract for "run me on the JS thread."
[^8]: `TurboModule` invocation surface: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h:80-86` defines `MethodMetadata::invoker` as `jsi::Value(jsi::Runtime&, TurboModule&, const jsi::Value*, size_t)`, and `TurboModule::create` (lines 116-128) calls it inline from the JS-thread host function.
[^9]: `PromiseKind` is one of the `TurboModuleMethodValueKind` enum members in the same `TurboModule.h:25-34`. The codegen path for `Promise<T>` lives in `packages/react-native-codegen/src/parsers/parsers-primitives.js:333-388` (`emitPromise`).
[^10]: Current-era TurboModule specs with callback parameters: `packages/react-native/src/private/webapis/microtasks/specs/NativeMicrotasks.js:16` and `packages/react-native/src/private/webapis/idlecallbacks/specs/NativeIdleCallbacks.js:27-30`. Generator emits `jsi::Function` for `FunctionTypeAnnotation` arguments in `packages/react-native-codegen/src/generators/modules/GenerateModuleH.js:126-127`. `packages/react-native/ReactCommon/react/bridging/CallbackWrapper.h` provides the helper for keeping a `jsi::Function` alive across threads.
[^11]: iOS `updateProps` protocol: `packages/react-native/React/Fabric/Mounting/RCTComponentViewProtocol.h:70-71`. Default no-op impl at `packages/react-native/React/Fabric/Mounting/UIView+ComponentViewProtocol.mm`. The reference manual diff lives in `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.mm:263-340`. Android counterpart: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/fabric/mounting/SurfaceMountingManager.kt:643-682` together with `ReactStylesDiffMap` at `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/uimanager/ReactStylesDiffMap.kt:14-30`.
[^12]: `prepareForRecycle` in the iOS protocol: `packages/react-native/React/Fabric/Mounting/RCTComponentViewProtocol.h:107-112`. Component views are pooled and reused across mounts, so per-mount state set in `init` will leak across recycled instances unless it is reset here.
[^13]: Command pipeline: `packages/react-native/Libraries/Utilities/codegenNativeCommands.js:17-31` (JS side), `packages/react-native/Libraries/ReactNative/BridgelessUIManager.js:240-260` (focus/blur dispatched as commands), `packages/react-native/React/Fabric/Mounting/RCTComponentViewProtocol.h:95-98` (`handleCommand:args:` on the component view).
[^14]: Migration convention: `packages/react-native/src/private/specs_DEPRECATED/__docs__/README.md` ("Do not create new specs in this directory. Instead, create them colocated with the rest of the logic in an appropriate directory in `src/private`"). The webapis modules listed in footnote 6 follow this layout end to end.
