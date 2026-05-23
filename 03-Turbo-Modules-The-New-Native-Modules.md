# Chapter 4: TurboModules - The New Native Modules

TurboModules are the second pillar of the New Architecture, representing a complete overhaul of the native module system. They replace the legacy system based on `RCTBridgeModule` and are designed from the ground up to be more performant, type-safe, and efficient. Just like Fabric, TurboModules are built directly on the foundation of the JSI and are the officially supported approach for new native modules in React Native 0.76+.[^3]

**Current Status (2025):** TurboModules are now the default native module system in React Native 0.76+. All new native modules should be implemented as TurboModules, and the legacy `RCTBridgeModule` system is deprecated.

## Visual: TurboModule + CodeGen Pipeline

```mermaid
flowchart LR
  Spec[TypeScript Spec\n(define interface)] --> CodeGen[CodeGen]
  CodeGen --> iOS[ObjC/Swift scaffolding]
  CodeGen --> Android[C++/Kotlin scaffolding]
  JS[JS Module\nTurboModuleRegistry.getEnforcing()] --> JSI[JSI]
  JSI --> NativeImpl[Native Impl\n(ObjC/Swift/C++)]
  NativeImpl --> Platform[iOS/Android APIs]
  JS <-->|sync/async| JSI
```

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

The heart of every TurboModule is the C++ class `facebook::react::TurboModule`, defined in `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h` [1]. This class inherits from `jsi::HostObject`, which is what allows it to be exposed directly to the JavaScript runtime. When JavaScript code calls a method on the module, it is directly invoking the `get` method of this C++ Host Object. The `TurboModule` class maintains a map of its methods and, upon request, provides the JSI with a `jsi::Function` that wraps the actual C++ implementation, which in turn calls the platform-specific native code.

**2. The iOS Implementation (`RCTTurboModule.h`)**

On iOS, a native module conforms to the `@protocol RCTTurboModule`, defined in `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.h` [2]. The system uses a C++ bridge class, `ObjCTurboModule`, which holds a reference to the user's Objective-C module instance. When a method is called from JavaScript, the `ObjCTurboModule` receives the call and uses the Objective-C runtime (`NSInvocation`) to dynamically invoke the correct method on the Objective-C instance.

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

*After (TurboModule):*

First, define the spec:
```typescript
// NativeMyModule.ts
export interface Spec extends TurboModule {
  getSyncValue(): string;
}
```

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

On Android, the approach is similar. A native module class implements the `com.facebook.react.turbomodule.core.interfaces.TurboModule` interface and, more importantly, extends the abstract `...Spec` class generated by CodeGen. The `ReactPackage` for the module is responsible for creating the native module instance when requested.

---

**Citations:**

[^1]: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h`
[^2]: `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.h`
[^3]: "Turbo Native Modules". React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction)
