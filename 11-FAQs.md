---
title: "Frequently Asked Questions (FAQs)"
chapter: "11"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:17:31-0400"
session_id: "audit-worker-11-faqs"
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
tags: [react-native, new-architecture, faq, turbomodules, fabric, bridgeless, jsi, interop, codegen]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/11-faqs/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:17:31-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Fixed: footnote [^1] URL (/docs/next/architecture/landing-page 404 → /docs/next/the-new-architecture/landing-page); footnote [^5] URL (/docs/next/the-new-architecture/renderer 404 → /architecture/fabric-renderer); Q6 corrected (iOS modules are NOT registered in the Podfile, they're registered via codegenConfig.ios.modulesProvider in package.json on the New Arch path, with the Podfile only triggering pod install/Codegen); Q6 clarified that getEnforcing throws via invariant, not returns undefined; Q3 added a note that the TurboModule interop layer is ON by default in Bridgeless; Q1 added v0.86 status (RCTIsNewArchEnabled now hardcoded to YES in C++ utils, legacy opt-out still works via gradle.properties / Podfile env var). See _verification/chapters/11-faqs/report.md."
---

# Chapter 12: Frequently Asked Questions (FAQs)

This chapter addresses common high-level questions about the React Native New Architecture.

**1. Do I have to migrate to the New Architecture?**

For now, the legacy architecture is still supported. However, as of React Native 0.76, the New Architecture is the default for new projects.[^1] All future development of React Native core and new React features (like Concurrent Rendering) will be built for the New Architecture. It is highly recommended to migrate to stay current and ensure the future viability of your application.

**Status as of v0.86 (`main`, commit `b32a6c9e9db`):** the legacy path is buildable but actively being removed. On iOS the C++ helper `RCTIsNewArchEnabled()` now unconditionally returns `YES`, and the matching setter is a documented no-op.[^6] The Cocoapods helper `NewArchitectureHelper.new_arch_enabled` also returns `true` outright.[^7] You can still opt out via `newArchEnabled=false` in `android/gradle.properties` or `ENV['RCT_NEW_ARCH_ENABLED'] = '0'` in your iOS `Podfile`,[^1] but a large surface of the legacy Bridge code is now behind `deprecated(... will be removed when removing the legacy architecture)` attributes.[^8]

**2. Is the Bridge completely gone in the New Architecture?**

In **Bridgeless Mode** (the default for new architecture apps since RN 0.74), the original message-queue Bridge is no longer initialized or used.[^2] However, an **interoperability layer** still exists to support legacy native modules. So, while the old performance-bottleneck Bridge is gone, the system can still communicate with old modules.

Concretely, the New Arch detects Bridgeless via the JS global `RN$Bridgeless`. When it's set, `TurboModuleBinding::install` mounts a `BridgelessNativeModuleProxy` (a JSI `HostObject`) as `global.nativeModuleProxy`, and any access to that proxy resolves first through the TurboModule provider, then through an optional legacy module provider.[^9]

**3. Can I still use my old native modules and components?**

Yes, for the most part. The interoperability layer allows legacy `RCTBridgeModule`s and `ViewManager`s to function. However, they will not see the performance benefits of the new architecture. For full compatibility and performance, they should be migrated to TurboModules and Fabric Components.

The TurboModule interop is **on by default in Bridgeless**: `RCTRootViewFactory.initializeReactHostWithLaunchOptions:` calls `RCTEnableTurboModuleInterop(YES)` before creating the React host on iOS,[^10] and the Android defaults class sets `useTurboModuleInterop()` to `true` whenever the New Arch is enabled.[^11] Legacy `RCT_EXPORT_MODULE` modules are wrapped by `ObjCInteropTurboModule` (which dispatches through `NSInvocation`),[^12] and legacy `ViewManager`s are bridged into Fabric through the `LegacyViewManagerInterop*` shadow node and component descriptor.[^13] The extra indirection is what costs you performance versus a "native" TurboModule or Fabric component.

**4. Do I need to learn C++ to use the New Architecture?**

Not necessarily to *use* it, but it is highly beneficial if you are *writing* native modules. You can write TurboModules and Fabric Components using only Objective-C or Java/Kotlin, as CodeGen handles the C++ interface generation. However, understanding the C++ layer (especially JSI concepts like `HostObject`) is crucial for debugging and for unlocking advanced performance patterns, such as writing a single, cross-platform module in C++. The canonical example of the C++ path lives in the repo at `packages/rn-tester/NativeCxxModuleExample/`, with `NativeCxxModuleExample.cpp` shared between iOS and Android.[^14]

**5. What is the difference between a TurboModule and a Fabric Component?**

-   **TurboModules** are for non-UI native functionality. They replace the old `NativeModules`. Examples include accessing the device's calendar, camera, or file system.
-   **Fabric Components** are for UI. They replace the old `ViewManager`s. They are native views that are rendered on the screen, like a map view or a custom video player.

**6. Why is my new TurboModule `undefined` in JavaScript?**

This is a common issue during migration. To be precise about the failure mode: `TurboModuleRegistry.getEnforcing('Foo')` does not return `undefined`. It throws via `invariant` with the message `TurboModuleRegistry.getEnforcing(...): 'Foo' could not be found. Verify that a module by this name is registered in the native binary.`[^15] The non-throwing sibling `TurboModuleRegistry.get('Foo')` returns `null` (not `undefined`) when the lookup misses both the TurboModule proxy and the legacy `NativeModules[name]` fallback. The most frequent causes are:

-   **Name Mismatch:** The name used in `TurboModuleRegistry.getEnforcing('MyModule')` in JavaScript does not exactly match the name registered in the native code (e.g., in the `BaseReactPackage` on Android).[^3]
-   **CodeGen Not Run:** You have created or changed the JS `Spec` file but have not rebuilt the app. You must re-run `pod install` or rebuild your Gradle project to trigger CodeGen and link the new module.[^4]
-   **Module Not Registered:** On Android, your module needs to be returned from a `BaseReactPackage.getModule(...)` and listed in `getReactModuleInfoProvider().getReactModuleInfos()`, with that package added to `MainApplication.getPackages()`.[^3] On iOS, the New Arch path registers modules through `codegenConfig.ios.modulesProvider` in your library's `package.json` (not in the `Podfile`). `pod install` is what *runs* CodeGen against that config; the Podfile itself is not the registration site.[^3]

**7. Is the New Architecture always faster than the old one?**

For most real-world applications, yes. The benefits in app startup (from lazy loading), UI responsiveness (from Fabric), and native call speed (from JSI) are significant.[^1][^5] The lazy loading isn't marketing: `TurboModuleBinding::getModule` only calls the module provider on first request, and the JS-side `jsRepresentation` populates each method via `TurboModule::get` the first time JS touches it.[^16]

However, in some very simple, static rendering scenarios, the overhead of the new architecture's more complex machinery can make it appear marginally slower than the legacy system. The team itself acknowledges this when documenting the migration: "enabling the New Architecture in your app or library may not immediately improve the performance or user experience."[^1] The true benefits shine in complex applications with many native interactions and dynamic UIs.

---

**Citations:**

[^1]: "About the New Architecture". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/landing-page](https://reactnative.dev/docs/next/the-new-architecture/landing-page)
[^2]: "React Native 0.74 - Yoga 3.0, Bridgeless New Architecture, and more". React Native Blog. [https://reactnative.dev/blog/2024/04/22/release-0.74](https://reactnative.dev/blog/2024/04/22/release-0.74). The matching CHANGELOG entries are "Make bridgeless the default when the New Arch is enabled" for both Android and iOS in the v0.74.0 section.
[^3]: "Turbo Native Modules". React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction)
[^4]: "Using Codegen". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen)
[^5]: "Fabric Renderer". React Native Documentation. [https://reactnative.dev/architecture/fabric-renderer](https://reactnative.dev/architecture/fabric-renderer)
[^6]: `RCTIsNewArchEnabled` always returns `YES`; `RCTSetNewArchEnabled` is a no-op. Source: `packages/react-native/React/Base/RCTUtils.mm:46-55`.
[^7]: `NewArchitectureHelper.new_arch_enabled` returns `true`. Source: `packages/react-native/scripts/cocoapods/new_architecture.rb:162-164`.
[^8]: Examples: `RCTAppDelegate.h:71-73`, `RCTReactNativeFactory.h:64,82,113,115`, `RCTArchConfiguratorProtocol.h:14`, `RCTAppSetupUtils.h:45,53,57`. All carry `deprecated(... will be removed when removing the legacy architecture)`.
[^9]: `class BridgelessNativeModuleProxy : public jsi::HostObject` and `TurboModuleBinding::install` source: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModuleBinding.cpp:20-84,117-160`.
[^10]: `RCTEnableTurboModuleInterop(YES)` is invoked unconditionally in Bridgeless host setup. Source: `packages/react-native/Libraries/AppDelegate/RCTRootViewFactory.mm:150-156`.
[^11]: `ReactNativeNewArchitectureFeatureFlagsDefaults` sets `useTurboModuleInterop()` to `true`. Source: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/internal/featureflags/ReactNativeNewArchitectureFeatureFlagsDefaults.kt:30`.
[^12]: `ObjCInteropTurboModule` (iOS) and `JavaInteropTurboModule` (Android) wrap legacy modules. Sources: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTInteropTurboModule.{h,mm}` and `.../platform/android/ReactCommon/JavaInteropTurboModule.{h,cpp}`.
[^13]: `LegacyViewManagerInterop*` files under `packages/react-native/ReactCommon/react/renderer/components/legacyviewmanagerinterop/`, plus `RCTLegacyViewManagerInteropCoordinator` for iOS.
[^14]: Source: `packages/rn-tester/NativeCxxModuleExample/NativeCxxModuleExample.{h,cpp,js,podspec}`.
[^15]: `TurboModuleRegistry.getEnforcing` uses `invariant(module != null, ...)`. `TurboModuleRegistry.get` returns `?T`. Source: `packages/react-native/Libraries/TurboModule/TurboModuleRegistry.js:35-47`.
[^16]: `TurboModuleBinding::getModule` runs the provider lazily and attaches a `jsRepresentation` whose property accesses defer to `TurboModule::get`. Source: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModuleBinding.cpp:170-218`.
