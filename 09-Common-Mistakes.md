---
title: "Common mistakes and misunderstandings"
chapter: "09"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:14:30-0400"
session_id: "b1610882-d17f-42e7-ab0e-0f415a9a3e81"
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
tags: [react-native, new-architecture, common-mistakes, jsi, turbomodules, fabric, codegen, bridgeless]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/09-common-mistakes/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:14:30-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). See _verification/chapters/09-common-mistakes/report.md."
---

# Chapter 10: Common Mistakes and Misunderstandings

The New Architecture introduces powerful concepts, but also new ways for things to go wrong. Understanding common pitfalls can save significant debugging time. This chapter outlines frequent mistakes and misunderstandings grouped by component.

(Version context: starting with v0.76, Fabric, TurboModules, and Bridgeless are the defaults. The mistakes below assume an app on v0.76 or newer. On the v0.86 line, the interop paths described below are still present but increasingly behind feature flags.)

## General Misunderstandings

**Mistake: Assuming the New Architecture automatically makes existing code faster.**

-   **Why it's a mistake:** Simply enabling the New Architecture flag does not optimize legacy components. An old, Bridge-based native module will still communicate via a compatibility layer. The performance benefits are only realized when a module is migrated to a TurboModule or a component to a Fabric Component.
-   **How to avoid:** Understand that performance gains require active migration of your native code to use the new, JSI-based APIs. The Android-side compatibility shim is the `InteropModuleRegistry`, gated behind the `useFabricInterop()` feature flag. It is annotated `@InteropLegacyArchitecture` in the source, which is a strong signal that this is a migration crutch rather than a long-term performance path.[^4]

**Mistake: Ignoring third-party library compatibility.**

-   **Why it's a mistake:** Enabling the New Architecture while using a third-party native library that does not support it is a common source of crashes and unexpected behavior. The library may rely on the old Bridge, which is not fully available in Bridgeless mode. Many UIManager methods (`measure`, `measureLayout`, `dispatchViewManagerCommand` constants, etc.) become soft errors in Bridgeless mode and log `'<method>' is not available in the new React Native architecture` instead of working.[^5]
-   **How to avoid:** Before migrating, audit all native dependencies. Check their documentation for "New Architecture," "Fabric," or "TurboModule" support. Update to compatible versions or find alternatives.

## JSI and C++ Mistakes

**Mistake: Blocking the JavaScript thread with a long-running synchronous method.**

-   **Why it's a mistake:** The ability to create synchronous methods is powerful, but it's also dangerous. Every TurboModule extends `jsi::HostObject`, so its method calls run on the calling thread (typically the JS thread) and return a `jsi::Value` directly.[^6] If a synchronous native method performs file I/O, network requests, or heavy computation, it will completely block the JS thread, freezing your app's UI.
-   **How to avoid:** Reserve synchronous methods for only the most trivial, fast operations (e.g., reading a pre-calculated value from memory). For everything else, declare the method as returning a `Promise` in the spec (CodeGen will produce `PromiseKind` on the C++ side[^6]) and do the work off-thread.

**Mistake: Ignoring JSI thread safety.**

-   **Why it's a mistake:** A `jsi::Runtime` instance is not thread-safe. The JSI source spells this out: the Runtime "may not be thread-aware, but cannot be used safely from multiple threads at once. The application is responsible for ensuring that it is used safely. This could mean using the Runtime from a single thread, using a mutex, doing all work on a serial queue, etc."[^7] Calling JSI methods that take a `jsi::Runtime&` from multiple threads without locking will lead to memory corruption and crashes.
-   **How to avoid:** Ensure all JSI interactions for a given runtime happen on a single, dedicated thread. If you need to pass data between threads, use thread-safe mechanisms and dispatch back to the JSI thread to interact with JavaScript. JSI also ships a `ThreadSafeRuntime` decorator (in `jsi/threadsafe.h`) that wraps any `Runtime` with explicit `lock()`/`unlock()` calls if you really need shared access from a small number of threads.[^8]

**Mistake: Performing complex operations in a `HostObject` destructor.**

-   **Why it's a mistake:** The destructor for a C++ `HostObject` is called when the JavaScript engine's garbage collector finalizes the object. The canonical JSI comment is worth reading verbatim: "The C++ object's dtor will be called when the GC finalizes this object. (This may be as late as when the Runtime is shut down.) You have no control over which thread it is called on. This will be called from inside the GC, so it is unsafe to do any VM operations which require a `IRuntime&`."[^9] So the destructor can fire on any thread, at any time, possibly with the runtime already dead. It is unsafe to call any JSI method that takes a `Runtime&`. Destructing other jsi objects (`Value`, `Object`, etc.) is explicitly allowed. The warning is about calling into the VM, not about C++ destruction.
-   **How to avoid:** The destructor should only be used for trivial cleanup of the C++ object's own memory. Derived classes' destructors should also avoid doing anything expensive.[^9] If you need to perform more complex cleanup that involves other systems or the JSI, implement an explicit `invalidate()` or `cleanup()` method on your Host Object that you call from JavaScript while the runtime is still alive, before releasing its reference.

## TurboModule Mistakes

**Mistake: Module name mismatch.**

-   **Why it's a mistake:** JavaScript finds a TurboModule by its name. If the name you provide in `TurboModuleRegistry.getEnforcing<Spec>('MyModule')` does not exactly match the name on the native registration side (`BaseReactPackage.getModule(name, …)` on Android[^10], or the `getTurboModule:` method on an `RCTModuleProvider` on iOS[^11]), the lookup fails. `getEnforcing` then throws via `invariant` with the message `TurboModuleRegistry.getEnforcing(...): '<name>' could not be found. Verify that a module by this name is registered in the native binary.`[^1] In practice this surfaces as a red-screen error in development and a fatal JS exception at the import callsite in production.
-   **How to avoid:** Ensure the module name is identical across the JS spec, the Android `BaseReactPackage.getModule` switch, and the iOS module provider. Both `package.json`'s `codegenConfig.libraries[].name` and the runtime registration must agree.

## Fabric Component Mistakes

**Mistake: Treating `updateProps` as a constructor or a one-time setup.**

-   **Why it's a mistake:** The `updateProps` method on a native component view will be called many times throughout the component's lifecycle. Every time a prop changes, the mounting layer calls `[componentView updateProps:newProps oldProps:oldProps]` (the Objective-C signature on the `RCTComponentViewProtocol`[^2]). The mount manager guards this with `if (oldChildShadowView.props != newChildShadowView.props)`, a pointer comparison on the immutable shared `Props`, which means any non-equal new props object triggers a call.[^12] Performing expensive, one-time setup work inside `updateProps` runs that work on every prop change.
-   **How to avoid:** Use the view's native initializer (e.g., `init` on iOS) for one-time setup. Use `updateProps` only for applying the new properties to the view. Also remember that Fabric component views are **recycled**: after unmount, the mount manager calls `prepareForRecycle`[^13] and the view returns to a pool to be reused for a different React component later. A single `UIView` instance may serve multiple React lives. Use `prepareForRecycle` to clear any state that mustn't leak across reuse (timers, observers, cached layers, etc.). Don't rely on `init` alone for "reset to defaults" behavior.

**Mistake: Passing frequently changing values as props.**

-   **Why it's a mistake:** While Fabric is much more efficient than the old UIManager, every prop change still results in a call from C++ to the native view (see the pointer-equality check at the mount-manager level cited above[^12]). If you are passing down values from an animation that updates every frame (e.g., from `useAnimatedStyle` in Reanimated), this can still create significant traffic.
-   **How to avoid:** For high-frequency updates like animations, use libraries like Reanimated that are designed to perform the animations directly on the UI thread, bypassing the React render cycle entirely. For one-off imperative actions, use commands instead of props. Commands are defined via `codegenNativeCommands<T>({ supportedCommands: [...] })`[^14] and dispatched through `ReactFabric.dispatchCommand`. The component view receives them via `handleCommand:args:` on the `RCTComponentViewProtocol`.[^2]

## CodeGen Mistakes

**Mistake: Forgetting to rebuild after changing a spec.**

-   **Why it's a mistake:** CodeGen runs during the build process. On iOS, `pod install` calls `run_codegen!` which wires `use_react_native_codegen_discovery!` into the install pipeline.[^15] On Android, the Gradle plugin makes `preBuild` depend on the `generateCodegenArtifactsFromSchema` task.[^16] If you change a JavaScript spec file (e.g., add a new method to a TurboModule), the native interface code is not automatically updated until one of those build steps runs again. You must re-run the build (`pod install` for iOS, or a Gradle build for Android) to regenerate the native files.[^3]
-   **How to avoid:** After any change to a `Spec` file, perform a clean build of your native project to ensure the generated code is up-to-date.

**Mistake: Incorrect `codegenConfig` in `package.json`.**

-   **Why it's a mistake:** If the `jsSrcsDir` or other paths in the `codegenConfig` are incorrect, the CodeGen script won't find your spec files. `generateSchemaInfos.js` joins `library.libraryPath` with `library.config.jsSrcsDir` to locate the specs.[^17] A wrong path simply yields an empty schema, and downstream artifacts (TurboModule providers, third-party component lists) end up without the bindings you expected, leading to compilation errors when the registration code references symbols that were never generated.
-   **How to avoid:** Double-check all paths and package names in the `codegenConfig` section to ensure they are correct. The `react-native` package itself is a useful reference: see its own `codegenConfig` block in `packages/react-native/package.json`, with `"name": "FBReactNativeSpec"` and `"jsSrcsDir": "src"`.[^17]

---

**Citations:**

[^1]: "Turbo Native Modules". React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction). The exact `invariant` message is in `packages/react-native/Libraries/TurboModule/TurboModuleRegistry.js:39-47`.
[^2]: "Fabric Native Components: iOS". React Native Documentation. [https://reactnative.dev/docs/next/fabric-native-components-introduction](https://reactnative.dev/docs/next/fabric-native-components-introduction). The protocol declaring `updateProps:oldProps:` and `handleCommand:args:` is `packages/react-native/React/Fabric/Mounting/RCTComponentViewProtocol.h:66-71, 96-98`.
[^3]: "Using Codegen". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen).
[^4]: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/bridge/interop/InteropModuleRegistry.kt:15-52`. The class is annotated `@InteropLegacyArchitecture` and only returns modules when `useFabricInterop()` is true.
[^5]: `packages/react-native/Libraries/ReactNative/BridgelessUIManager.js:22-101`. Many UIManager APIs (`measure`, `measureLayout`, `measureInWindow`, etc.) call `raiseSoftError(methodName)` which logs `'<methodName>' is not available in the new React Native architecture`.
[^6]: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h:25-46`. `TurboModule` extends `jsi::HostObject` (line 46). `TurboModuleMethodValueKind` enumerates the supported return shapes including `PromiseKind` (line 33).
[^7]: `packages/react-native/ReactCommon/jsi/jsi/jsi.h:687-704`. The `Runtime` class header comment is the canonical statement about JSI thread safety.
[^8]: `packages/react-native/ReactCommon/jsi/jsi/threadsafe.h:18-23`. `ThreadSafeRuntime` is a `Runtime` subclass with `virtual void lock() const = 0; virtual void unlock() const = 0; virtual Runtime& getUnsafeRuntime() = 0;`.
[^9]: `packages/react-native/ReactCommon/jsi/jsi/jsi.h:222-233`. The `HostObject` destructor comment is the source-of-truth for the GC/thread/safety constraints.
[^10]: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/BaseReactPackage.kt:22-41`. `BaseReactPackage` is the modern parent class. The older `ReactPackage.createNativeModules` is now `@Deprecated` (see `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactPackage.kt:36-38`).
[^11]: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.h:178-185`. The `RCTModuleProvider` protocol declares `- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:(const facebook::react::ObjCTurboModule::InitParams &)params;`.
[^12]: `packages/react-native/React/Fabric/Mounting/RCTMountingManager.mm:110-115`. The mount manager only calls `updateProps:` when `oldChildShadowView.props != newChildShadowView.props` (pointer equality on the immutable shared `Props`).
[^13]: `packages/react-native/React/Fabric/Mounting/RCTComponentViewProtocol.h:107-112`. `- (void)prepareForRecycle;` is documented as "Called right after the component view is moved to a recycle pool. Receiver must reset any local state and release associated non-reusable resources." A real implementation example: `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.mm:687-726`.
[^14]: `packages/react-native/Libraries/Utilities/codegenNativeCommands.js:17-31`. The helper returns command-dispatch functions that call `dispatchCommand(ref, command, args)`. Real-world example: `packages/react-native/Libraries/Components/ScrollView/ScrollViewCommands.js:45-52` (`flashScrollIndicators`, `scrollTo`, `scrollToEnd`, `zoomToRect`).
[^15]: `packages/react-native/scripts/cocoapods/codegen.rb:9-39`. `run_codegen!` wires `use_react_native_codegen_discovery!` into pod install.
[^16]: `packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/ReactPlugin.kt:264`. `project.tasks.named("preBuild", Task::class.java).dependsOn(generateCodegenArtifactsTask)`.
[^17]: `packages/react-native/scripts/codegen/generate-artifacts-executor/generateSchemaInfos.js:29-33` (joins `library.libraryPath` and `library.config.jsSrcsDir`). See also `packages/react-native/package.json:197-231` (real `codegenConfig` example).
