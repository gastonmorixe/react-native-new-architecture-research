---
title: "TurboModules: the new native modules"
chapter: "03"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:25:19-0400"
session_id: "audit-worker-03-turbo-modules"
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
tags: [react-native, new-architecture, turbomodules, jsi, codegen, native-modules]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/03-turbo-modules/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:25:19-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). See _verification/chapters/03-turbo-modules/report.md."
---

# Chapter 4: TurboModules - The New Native Modules

TurboModules are the second pillar of the New Architecture, representing a complete overhaul of the native module system. They replace the legacy system based on `RCTBridgeModule` and are designed from the ground up to be more performant, type-safe, and efficient. Just like Fabric, TurboModules are built directly on the foundation of the JSI and are the officially supported approach for new native modules in React Native 0.76+.[^3]

**Current Status (2025):** TurboModules are now the default native module system in React Native 0.76+. All new native modules should be implemented as TurboModules, and the legacy `RCTBridgeModule` system is deprecated.

## Visual: TurboModule + CodeGen Pipeline

```mermaid
flowchart LR
  Spec[TypeScript or Flow Spec\n(define interface)] --> CodeGen[CodeGen]
  CodeGen --> iOS[ObjC++ scaffolding\nNativeXxxSpec / NativeXxxSpecJSI]
  CodeGen --> Android[Java spec + C++/JNI glue\nNativeXxxSpec.java]
  JS[JS Module\nTurboModuleRegistry.getEnforcing] --> JSI[JSI]
  JSI --> NativeImpl[Native Impl\nObjC/Swift or Java/Kotlin/C++]
  NativeImpl --> Platform[iOS/Android APIs]
  JS <-->|sync/async| JSI
```

Note that CodeGen emits **ObjC++** for iOS and **Java + JNI/C++** for Android, not Swift or Kotlin directly. Swift code consumes the ObjC++ header via a bridging header; Kotlin extends the generated Java spec class as if it were Kotlin (the languages share a single classpath).

## The Motivation: Problems with Legacy Native Modules

Legacy native modules, while powerful, suffered from limitations tied directly to their reliance on the asynchronous Bridge:

1.  **Eager Initialization:** All native modules registered with an application had to be initialized and loaded into memory when the app started, contributing to slower startup times.
2.  **Asynchronous Everything:** Every method call was asynchronous. It was impossible for a JavaScript function to get an immediate return value from a native method.
3.  **No Type Safety:** The contract between JavaScript and native code was based on convention and runtime checks. A mismatch in method signatures or argument types would lead to runtime errors.

TurboModules were created to solve these specific problems.

## The Core Benefits of TurboModules

#### 1. Lazy Loading

TurboModules are **lazily loaded**. A native module is not instantiated until it is accessed for the first time in JavaScript (e.g., via `TurboModuleRegistry.getEnforcing('MyModule')`). This on-demand approach significantly improves application startup performance.

#### 2. Synchronous Method Execution

Because TurboModules are fundamentally `jsi::HostObject`s, they can expose methods that can be called **synchronously** from JavaScript. This is invaluable for small, fast functions that need to return data to JavaScript immediately.

#### 3. Type Safety Across Languages

TurboModules enforce a strict, type-safe contract between JavaScript and the native platforms. This is achieved through a combination of JavaScript specs and automatic code generation.

-   **The Spec is the Source of Truth:** A developer defines the module's interface in a TypeScript file.
-   **CodeGen Enforces the Contract:** CodeGen parses the spec and generates native interface files. If the native implementation's method signature doesn't match the spec, the code won't compile.

## How TurboModules Work Under the Hood

The implementation of a TurboModule spans three layers:

**1. The C++ Core (`TurboModule.h`)**

The heart of every TurboModule is the C++ class `facebook::react::TurboModule`, defined in `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h`[^1]. This class inherits from `jsi::HostObject`, which is what allows it to be exposed directly to the JavaScript runtime. The first time JavaScript reads a property on the module (for example, calling `MyModule.getSyncValue`), JSI invokes the C++ `get` method of this Host Object, which looks the name up in `methodMap_` and returns a `jsi::Function` built with `jsi::Function::createFromHostFunction`. That `jsi::Function` is then cached on the JS-side wrapper object[^4], so subsequent calls go straight to the host function without another `get` round-trip. The host function ultimately invokes the platform-specific native code.

**2. The iOS Implementation (`RCTTurboModule.h`)**

On iOS, a native module conforms to the `@protocol RCTTurboModule`, defined in `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.h`[^2]. In v0.86, `RCTTurboModule` inherits from a small `RCTModuleProvider` protocol that declares the single `- getTurboModule:` requirement[^5]. The system uses a C++ bridge class, `ObjCTurboModule`, which holds a reference to the user's Objective-C module instance via `id<RCTBridgeModule> instance_`. When a method is called from JavaScript, the `ObjCTurboModule` receives the call and uses the Objective-C runtime (`NSInvocation`) to dynamically invoke the correct method on the Objective-C instance.[^6]

**Example Migration (iOS):**

*Before (Legacy `RCTBridgeModule`):*
```objc
// MyModule.m
#import <React/RCTBridgeModule.h>

@interface MyModule : NSObject <RCTBridgeModule>
@end

@implementation MyModule
RCT_EXPORT_MODULE();
RCT_EXPORT_METHOD(getSyncValue:(RCTResponseSenderBlock)callback)
{
  callback(@[@"some sync value"]);
}
@end
```

Note that the legacy method named `getSyncValue` is actually invoked **asynchronously** by JS: `RCT_EXPORT_METHOD` always marshals through the bridge and returns the value via a callback block.[^7] The legacy bridge does have a true sync escape hatch (`RCT_EXPORT_BLOCKING_SYNCHRONOUS_METHOD`, defined in the same header), but it was discouraged because it ran on the JS thread and stalled it. The TurboModule version below is sync without that caveat, because the call goes directly through JSI without any bridge serialization.

*After (TurboModule):*

First, define the spec. The file name is significant: it must start with `Native` and match the haste name used in the generated artifacts (`NativeMyModule.ts` produces `NativeMyModuleSpec` / `NativeMyModuleSpecJSI`). The argument to `getEnforcing` is the **JS-facing module name**, which the native side exposes as the `NAME` constant on the generated spec class.[^8]

```typescript
// NativeMyModule.ts
import type {TurboModule} from 'react-native/Libraries/TurboModule/RCTExport';
import * as TurboModuleRegistry from 'react-native/Libraries/TurboModule/TurboModuleRegistry';

export interface Spec extends TurboModule {
  getSyncValue(): string;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

CodeGen also accepts Flow specs (most of RN's own bundled `Native*` files under `src/private/specs_DEPRECATED/modules/` are still Flow), but TypeScript is the recommended choice for new third-party modules.[^9]

Then, update the native class to conform to the generated spec. Note the removal of `RCT_EXPORT_` macros and the addition of the `getTurboModule` boilerplate.

```objc
// MyModule.mm
#import "MyModuleSpec.h" // Import the header generated by CodeGen

@interface MyModule () <NativeMyModuleSpec>
@end

@implementation MyModule

// Boilerplate to connect the class to the TurboModule system
- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params
{
  return std::make_shared<facebook::react::NativeMyModuleSpecJSI>(params);
}

// The actual implementation, now strongly typed by the generated spec
- (NSString *)getSyncValue
{
  return @"some sync value";
}

@end
```

**3. The Android Implementation**

On Android, the approach is similar. The marker interface is `com.facebook.react.turbomodule.core.interfaces.TurboModule`, defined in `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/turbomodule/core/interfaces/TurboModule.kt`.[^10] You don't normally implement it directly. The generated abstract spec class already does (`public abstract class NativeMyModuleSpec extends ReactContextBaseJavaModule implements TurboModule`),[^11] so your concrete module just extends the spec:

```kotlin
// MyModule.kt
@ReactModule(name = NativeMyModuleSpec.NAME)
class MyModule(reactContext: ReactApplicationContext) :
    NativeMyModuleSpec(reactContext) {

  override fun getSyncValue(): String = "some sync value"
}
```

For a method declared with a non-Promise, non-void return type (like our `getSyncValue(): string`), CodeGen tags the generated abstract method with `@ReactMethod(isBlockingSynchronousMethod = true)`, which makes JS calls execute synchronously on the JS thread.

Module registration is then handled by a `ReactPackage`. In the New Arch, the supported pattern is to extend `BaseReactPackage` and override `getModule(name, reactContext)` for lazy lookup, not the older `createNativeModules` method, which is now marked `@Deprecated("Migrate to [BaseReactPackage] and implement [getModule] instead.")`.[^12] React Native's own `MainReactPackage` uses exactly this shape, switching on the module name and returning a fresh instance only when the JS proxy first asks for it.

```kotlin
class MyPackage : BaseReactPackage() {
  override fun getModule(name: String, reactContext: ReactApplicationContext) =
      when (name) {
        NativeMyModuleSpec.NAME -> MyModule(reactContext)
        else -> null
      }

  override fun getReactModuleInfoProvider() = ReactModuleInfoProvider {
    mapOf(NativeMyModuleSpec.NAME to ReactModuleInfo(
      NativeMyModuleSpec.NAME, MyModule::class.java.name,
      false, false, false, true,
    ))
  }
}
```

---

**Citations:**

[^1]: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h` (HEAD `7f8c75f2f7b`, upstream `b32a6c9e9db`, v0.86.0-rc.1 era). `class JSI_EXPORT TurboModule : public jsi::HostObject` at line 46. The `get` override is inlined at lines 55-65; `methodMap_` (the `std::unordered_map<std::string, MethodMetadata>` that maps method names to their host-function invokers) is declared at line 85.

[^2]: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.h`. `class JSI_EXPORT ObjCTurboModule : public TurboModule` at line 52; `id<RCTBridgeModule> instance_` at line 73; `@protocol RCTTurboModule <RCTModuleProvider>` at line 191.

[^3]: "Turbo Native Modules". React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction). The 0.76 release blog post confirms the default switch: https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture. CHANGELOG entry for v0.76 in `CHANGELOG-0.7x.md`: "TurboModules will be looked up as TurboModules first, and fallback to legacy modules after" (commit `5a62606ab3`).

[^4]: The cache is the `jsRepresentation_` weak-object reference in `TurboModule.h:140`. The caching call lives in the `get` body at line 62: `jsRepresentation_->lock(runtime).asObject(runtime).setProperty(runtime, propName, prop)`. Comment at lines 58-60 states explicitly: "We don't cache misses, to allow for methodMap_ to dynamically be extended."

[^5]: `RCTModuleProvider` protocol is declared at `RCTTurboModule.h:178-185`. Its single required method is `- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:(const facebook::react::ObjCTurboModule::InitParams &)params`. This split landed so that pure-C++ TurboModules (without an Objective-C class) can still register through the same provider surface.

[^6]: `RCTTurboModule.mm` constructs the `NSInvocation` at line 725 (`NSInvocation *inv = [NSInvocation invocationWithMethodSignature:methodSignature]`), sets the selector at 726, fills each argument via `setInvocationArg` (728-732), and finally invokes the selector on `instance_` in `performMethodInvocation` (line 780+). The full call site is `ObjCTurboModule::invokeObjCMethod` at lines 760-820.

[^7]: `RCT_EXPORT_METHOD` is defined at `packages/react-native/React/Base/RCTBridgeModule.h:218` (`#define RCT_EXPORT_METHOD(method) RCT_REMAP_METHOD(, method)`). Legacy bridge dispatch routes the call through the batched JSON message queue, so the value comes back via the callback block, not as a return value. The true-sync alternative `RCT_EXPORT_BLOCKING_SYNCHRONOUS_METHOD` is at line 236 and is documented with a "WARNING: ... can have strong performance penalties" comment at lines 223-227.

[^8]: CodeGen ObjC++ generator: `packages/react-native-codegen/src/generators/modules/GenerateModuleObjCpp/index.js:36` emits `@protocol ${hasteModuleName}Spec <RCTBridgeModule, RCTTurboModule>` and `:55-58` emits `class JSI_EXPORT ${hasteModuleName}SpecJSI : public ObjCTurboModule { public: ${hasteModuleName}SpecJSI(const ObjCTurboModule::InitParams &params); };`. Java generator: `GenerateModuleJavaSpec.js:62` emits `public abstract class ${className} extends ReactContextBaseJavaModule implements TurboModule` and `:63` emits `public static final String NAME = "${jsName}";`. The full round-trip is exercised in `private/react-native-codegen-typescript-test/src/__tests__/simple-scenario-frontend-test.ts:14-40` and reproduced for this chapter at `_verification/chapters/03-turbo-modules/tests/spec_objc_gen.js`.

[^9]: TS and Flow parsers live side by side: `packages/react-native-codegen/src/parsers/typescript/` and `.../parsers/flow/`. Every shipped `Native*` spec under `packages/react-native/src/private/specs_DEPRECATED/modules/` uses Flow (see e.g. `NativeSampleTurboModule.js`); third-party modules under the New Arch are typically authored in TypeScript.

[^10]: `package com.facebook.react.turbomodule.core.interfaces ... public interface TurboModule { public fun initialize(); public fun invalidate(); }` (`TurboModule.kt:8-19`). The two methods are lifecycle hooks the bridge calls on init and on `ReactHost` tear-down.

[^11]: Confirmed by inspection of any RN-shipped module that's already migrated. For example, `BlobModule.kt:44-45` reads `class BlobModule(reactContext: ReactApplicationContext) : NativeBlobModuleSpec(reactContext), TurboModuleWithJSIBindings`. It extends the generated spec class and only adds `TurboModuleWithJSIBindings` (a separate interface for modules that install their own JSI bindings). It does NOT re-declare `TurboModule`, because the spec class already implements it.

[^12]: `ReactPackage.kt:36-38` (`@Deprecated("Migrate to [BaseReactPackage] and implement [getModule] instead.")`); `ReactPackage.kt:51-52` declares the supported `getModule(name, reactContext)` hook. Live usage in `MainReactPackage.kt:108-137`, which switches on `name` and returns lazy-instantiated modules. `BaseReactPackage` provides the boilerplate needed to back the deprecated `createNativeModules` with `getModule` calls so the registry can still enumerate when asked.
