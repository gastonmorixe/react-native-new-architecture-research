---
title: "Tutorial - Building a Native Module and Component"
chapter: "14"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:23:33-0400"
session_id: "audit-worker-14-tutorial"
host_info:
  hostname: "macbookpro.home.arpa"
  user: "gaston"
  os: "macOS 26.5"
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
tags: [react-native, new-architecture, tutorial, turbomodule, fabric, codegen]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/14-tutorial/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:23:33-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Fixed iOS Promise method signature (RCTPromiseResolveBlock / RCTPromiseRejectBlock, not facebook::react::Promise<T>), Android base class (BaseReactPackage vs deprecated TurboReactPackage), generated Java spec class name (NativeDeviceInfoModuleSpec, not DeviceInfoSpec), ReactModuleInfo 6-arg ctor, added componentDescriptorProvider and <Name>Cls registration for the iOS Fabric view, added module-name companion object on Android. See _verification/chapters/14-tutorial/report.md."
---

# Chapter 15: Tutorial - Building a Native Module and Component

> **Audit note (RN 0.86):** the original draft of this chapter ships with a handful of incorrect signatures and out-of-date base classes that would break the build verbatim. The corrected code below is grounded in the current React Native source. Inline footnotes point at the exact files. The H1 still reads "Chapter 15" to match the rest of the book's running numbering, even though the filename starts with `14-`; this is a book-wide off-by-one that was not in scope to retitle.

This chapter provides a hands-on, step-by-step tutorial to create a simple React Native library that uses both a TurboModule and a Fabric Component. This will demonstrate how the concepts discussed in previous chapters come together in practice.

Our library, `react-native-deviceinfo`, will provide:
- A **TurboModule** called `DeviceInfoModule` to get the device's model name.
- A **Fabric Component** called `DeviceInfoView` that displays a label passed via a prop.

## Part 1: Project Setup

First, we need a standard file structure for a native library. You can create this manually or use a community tool like `create-react-native-library`.

The essential structure will be:

```
react-native-deviceinfo/
├── android/
├── ios/
├── src/
└── package.json
```

Next, configure `package.json` to tell React Native about your native module and, crucially, to configure CodeGen.[^1]

**`package.json`:**
```json
{
  "name": "react-native-deviceinfo",
  "version": "1.0.0",
  "main": "src/index.ts",
  "react-native": "src/index.ts",
  "files": [
    "src",
    "android",
    "ios",
    "react-native-deviceinfo.podspec"
  ],
  "codegenConfig": {
    "name": "DeviceInfoSpec",
    "type": "all",
    "jsSrcsDir": "src/specs",
    "android": {
      "javaPackageName": "com.reactnativedeviceinfo"
    }
  }
}
```
- **`codegenConfig`**: This is the most important part. It tells CodeGen to look for spec files in the `src/specs` directory. The `name` field (`DeviceInfoSpec`) is the **library name** that namespaces the generated artifacts on iOS (and shows up in generated include paths like `<react/renderer/components/DeviceInfoSpec/Props.h>`). The `type` field accepts `"all" | "modules" | "components"`.[^1]
- **iOS podspec discovery**: there is no `podspecName` top-level field in React Native's `package.json` schema.[^1] The podspec is discovered by `use_native_modules!` in the host app's `Podfile` based on the `.podspec` file you ship in the package root (here, `react-native-deviceinfo.podspec`). The basename of that file becomes the pod's name.

## Part 2: Building the TurboModule

Our TurboModule will expose one asynchronous method, `getDeviceName()`.

### Step 2.1: The JavaScript Spec (TypeScript)

This is the **source of truth**. Create a file at `src/specs/NativeDeviceInfoModule.ts`. The file name's basename (without the `.ts` extension) is what codegen calls the **haste module name**; it ends up in the generated artifact names on both platforms (for example, the Android abstract class is named `NativeDeviceInfoModuleSpec`).[^2]

**`src/specs/NativeDeviceInfoModule.ts`:**
```typescript
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // A promise-based async method
  getDeviceName(): Promise<string>;
}

// 'DeviceInfoModule' is the name our native code will use to register itself.
export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfoModule');
```

### Step 2.2: The iOS Implementation (Objective-C++)

On iOS, the implementation requires an `.mm` file. This is **Objective-C++**, a language that can understand both the C++ generated by CodeGen and the Objective-C of the native iOS frameworks (like `UIDevice`).[^2]

The conventional shape in the React Native source is to keep the public header minimal (declaring `<RCTBridgeModule>` conformance only) and to declare the generated spec protocol conformance in a class extension inside the `.mm` file. That keeps the spec name out of the public header surface.[^3]

**`ios/DeviceInfoModule.h`:**
```objc
#import <Foundation/Foundation.h>
#import <React/RCTBridgeModule.h>

@interface DeviceInfoModule : NSObject <RCTBridgeModule>
@end
```

**`ios/DeviceInfoModule.mm`:**
```objc
#import "DeviceInfoModule.h"
#import <UIKit/UIKit.h>

// Import the header generated by CodeGen. The library name comes from
// codegenConfig.name ("DeviceInfoSpec"). The build system maps this to a
// header search path during the codegen step.
#import <DeviceInfoSpec/DeviceInfoSpec.h>

@interface DeviceInfoModule () <NativeDeviceInfoModuleSpec>
@end

@implementation DeviceInfoModule

RCT_EXPORT_MODULE()

// Hand back the JSI-callable wrapper. The generated SpecJSI class is what
// actually bridges JS calls into the Objective-C method below.
- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params
{
    return std::make_shared<facebook::react::NativeDeviceInfoModuleSpecJSI>(params);
}

// The codegen-generated protocol declares the Promise-returning method with
// RCTPromiseResolveBlock / RCTPromiseRejectBlock, NOT C++ Promise types.
//   typedef void (^RCTPromiseResolveBlock)(id result);
//   typedef void (^RCTPromiseRejectBlock)(NSString *code,
//                                         NSString *message,
//                                         NSError *error);
- (void)getDeviceName:(RCTPromiseResolveBlock)resolve
               reject:(RCTPromiseRejectBlock)reject
{
    @try {
        NSString *deviceName = [[UIDevice currentDevice] name];
        resolve(deviceName);
    } @catch (NSException *exception) {
        reject(@"E_DEVICE_NAME", exception.reason ?: @"Unknown error", nil);
    }
}

@end

// Exposed C function the codegen-generated plugin map (RCTFabricComponentsPlugins.mm
// equivalent for modules: RCTModuleProviders / <Lib>ModuleProvider.mm) calls to
// hand back the concrete class.
Class DeviceInfoModuleCls(void)
{
    return DeviceInfoModule.class;
}
```

A few corrections worth flagging if you cross-reference older tutorials:

- The resolve/reject parameters are `RCTPromiseResolveBlock` and `RCTPromiseRejectBlock` (typedef'd `id`-and-error blocks in `React/RCTBridgeModule.h`), not C++ `facebook::react::Promise<T>`.[^4] The C++ bridging side does have an `AsyncPromise<T>` class (`react/bridging/Promise.h`), but it is used internally by the generated wrapper and never appears in the protocol your class implements.
- `RCTPromiseRejectBlock` takes three arguments (`code`, `message`, `error`), not a single exception.[^4]
- `RCT_EXPORT_MODULE()` is what registers the class under the JS-visible name (`"DeviceInfoModule"` by default, matching the symbol after `RCT_EXPORT_MODULE` strips the `RCT` prefix). Without it the TurboModule manager has no way to find your class.

About `[[UIDevice currentDevice] name]`: starting with iOS 16, this returns a generic model string (e.g. `"iPhone"`) for any app that does not have the special `UIDevice.name` user-name entitlement. If you actually want the user-customized name, the entitlement is the only path; otherwise switch to a model identifier like `[[UIDevice currentDevice] model]`.

**Why not pure Swift?** While the logic could be in Swift, you still need this `.mm` file to bridge between the C++ world of JSI and the Swift world. The common pattern is to have the `.mm` file call a pure Swift class. Swift's C++ interop is opt-in and still maturing; until your team is ready to depend on it, the `.mm` → `.swift` indirection is the safe path.

### Step 2.3: The Android Implementation (Kotlin)

On Android, we'll use Kotlin. The class must extend the generated abstract Spec class.

**Naming gotcha.** The generated abstract class is **`Native<JsFilename>Spec`**, where the JS filename is the haste basename. Our spec file is `NativeDeviceInfoModule.ts`, so the generated abstract class is `NativeDeviceInfoModuleSpec` (not `DeviceInfoSpec`, which is the codegenConfig library `name`).[^5] The generated class also already declares `public static final String NAME = "DeviceInfoModule"` and a `getName()` that returns it, so you do not need to override either of them in the subclass.[^5]

**`android/src/main/java/com/reactnativedeviceinfo/DeviceInfoModule.kt`:**
```kotlin
package com.reactnativedeviceinfo

import android.os.Build
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.annotations.ReactModule

// NativeDeviceInfoModuleSpec is generated under codegenConfig.android.javaPackageName,
// which we set to com.reactnativedeviceinfo above, so it lives in this same package
// and no explicit import is needed.

@ReactModule(name = NativeDeviceInfoModuleSpec.NAME)
class DeviceInfoModule(context: ReactApplicationContext)
    : NativeDeviceInfoModuleSpec(context) {

    // No getName() override needed: NativeDeviceInfoModuleSpec already
    // returns its NAME constant.
    // No @ReactMethod needed on the override: the abstract method is already
    // annotated with @ReactMethod @DoNotStrip in the generated parent class.
    override fun getDeviceName(promise: Promise) {
        promise.resolve(Build.MODEL)
    }

    companion object {
        // Re-expose NAME on the subclass so DeviceInfoPackage can reference it
        // as DeviceInfoModule.NAME (Java statics declared on the generated parent
        // are not auto-inherited under Kotlin's companion-style access).
        // `val`, not `const val`: Kotlin requires `const val` initializers to be
        // literals or other `const val`s, not Java `static final` references.
        @JvmField val NAME: String = NativeDeviceInfoModuleSpec.NAME
    }
}
```

You also need a `Package` file to register the module with React Native.

**`android/src/main/java/com/reactnativedeviceinfo/DeviceInfoPackage.kt`:**
```kotlin
package com.reactnativedeviceinfo

import com.facebook.react.BaseReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider

// BaseReactPackage replaced TurboReactPackage in the v0.76 cycle; TurboReactPackage
// is now @Deprecated and exists only as a thin alias.
class DeviceInfoPackage : BaseReactPackage() {

    override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? =
        if (name == DeviceInfoModule.NAME) DeviceInfoModule(reactContext) else null

    override fun getReactModuleInfoProvider(): ReactModuleInfoProvider =
        ReactModuleInfoProvider {
            mapOf(
                DeviceInfoModule.NAME to ReactModuleInfo(
                    DeviceInfoModule.NAME,                  // name
                    DeviceInfoModule::class.java.name,      // className (FQN of impl)
                    false,                                  // canOverrideExistingModule
                    false,                                  // needsEagerInit
                    false,                                  // isCxxModule
                    true                                    // isTurboModule
                )
            )
        }
}
```

Two corrections worth calling out:

- The recommended base class is `BaseReactPackage`. `TurboReactPackage` is `@Deprecated` since the 0.76 cycle and only kept as a no-op subclass for source compatibility.[^6]
- `ReactModuleInfo`'s primary constructor has **six** parameters today (no `hasConstants`). The older seven-arg overload is `@Deprecated` and `@Suppress("UNUSED_PARAMETER")`-marks `hasConstants`, so it compiles but warns and the value is ignored.[^7]

## Part 3: Building the Fabric Component

Our Fabric component will be a simple native `View` that displays a text label.

### Step 3.1: The JavaScript Spec (TypeScript)

Create a file at `src/specs/DeviceInfoViewNativeComponent.ts`.[^3]

**`src/specs/DeviceInfoViewNativeComponent.ts`:**
```typescript
import type { ViewProps } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

// Define the props that our native view will accept
export interface NativeProps extends ViewProps {
  label: string;
}

// 'DeviceInfoView' is the name our native code will use
export default codegenNativeComponent<NativeProps>('DeviceInfoView');
```

### Step 3.2: The iOS Implementation

In the New Architecture, a Fabric component on iOS is a subclass of `RCTViewComponentView`.[^8] You do **not** need an `RCTViewManager` for a Fabric-only library. The legacy `RCTViewManager` pattern only matters for the Paper renderer; in the RN source you can see this directly because files like `Libraries/Image/RCTImageViewManager.mm` are wrapped in `#ifndef RCT_REMOVE_LEGACY_ARCH`.[^9] Fabric discovers third-party component classes through a codegen-generated plugin map (`RCTFabricComponentsProvider` in `RCTFabricComponentsPlugins.mm`) that maps a component name string to a C function returning the class.[^9] Two things you must therefore provide on the view:

1. A class method `+ (ComponentDescriptorProvider)componentDescriptorProvider` returning `concreteComponentDescriptorProvider<DeviceInfoViewComponentDescriptor>()`. Without it, Fabric does not know which `ComponentDescriptor` to use for layout/shadow tree construction.
2. A C function named `<ComponentName>Cls(void)` that returns the view's class. The generated plugin map calls it during component registration.

**`ios/DeviceInfoView.h`:**
```objc
#import <React/RCTViewComponentView.h>

@interface DeviceInfoView : RCTViewComponentView
@end
```

**`ios/DeviceInfoView.mm`:**
```objc
#import "DeviceInfoView.h"

#import <React/RCTConversions.h>
#import <react/renderer/components/DeviceInfoSpec/ComponentDescriptors.h>
#import <react/renderer/components/DeviceInfoSpec/Props.h>
#import <react/renderer/components/DeviceInfoSpec/RCTComponentViewHelpers.h>
#import <react/renderer/components/DeviceInfoSpec/ShadowNodes.h>

using namespace facebook::react;

@implementation DeviceInfoView {
    UILabel *_labelView;
}

- (instancetype)initWithFrame:(CGRect)frame
{
    if (self = [super initWithFrame:frame]) {
        // Seed _props with the codegen-generated default values so the first
        // updateProps has a sane "old" baseline. The base RCTViewComponentView
        // declares _props as a protected ivar of type SharedViewProps. Every
        // generated ShadowNode exposes a static defaultSharedProps() for this
        // purpose (compare RCTSwitchComponentView.mm:45, etc.).
        _props = DeviceInfoViewShadowNode::defaultSharedProps();

        _labelView = [UILabel new];
        [self addSubview:_labelView];
    }
    return self;
}

// Tell Fabric which ComponentDescriptor describes this view's shadow node + props.
+ (ComponentDescriptorProvider)componentDescriptorProvider
{
    return concreteComponentDescriptorProvider<DeviceInfoViewComponentDescriptor>();
}

// Fabric calls this with const-pointer-to-const Props pairs. Props::Shared is
// `using Shared = std::shared_ptr<const Props>` in react/renderer/core/Props.h.
// The house style in RN's built-in component views is to reference-cast
// `*shared_ptr` rather than `*std::static_pointer_cast` (no control-block churn).
- (void)updateProps:(const Props::Shared &)props oldProps:(const Props::Shared &)oldProps
{
    const auto &oldViewProps = static_cast<const DeviceInfoViewProps &>(*_props);
    const auto &newViewProps = static_cast<const DeviceInfoViewProps &>(*props);

    if (oldViewProps.label != newViewProps.label) {
        // RCTNSStringFromString is the canonical helper in RCTConversions.h;
        // it handles empty / UTF-8 edge cases better than +stringWithUTF8String:.
        _labelView.text = RCTNSStringFromString(newViewProps.label);
        [self setNeedsLayout];
    }

    [super updateProps:props oldProps:oldProps];
}

- (void)layoutSubviews
{
    [super layoutSubviews];
    _labelView.frame = self.bounds;
}

@end

// C registration function picked up by the codegen-generated plugin map.
Class<RCTComponentViewProtocol> DeviceInfoViewCls(void)
{
    return DeviceInfoView.class;
}
```

If you are tempted to keep an empty `RCTDeviceInfoViewManager` "just in case", don't: it adds a TurboModule registration the runtime never reaches, and the `RCT_EXPORT_MODULE()` macro would conflict with the same name being claimed by the Fabric plugin map.[^9]

### Step 3.3: The Android Implementation

On Android the picture is different from iOS: the `ViewManager` is still the entry point under Fabric. Codegen generates a `<Component>ManagerInterface<T>` and a `<Component>ManagerDelegate<T, U>` in the `com.facebook.react.viewmanagers` package. The manager implements the interface and forwards prop sets to the delegate via `getDelegate()`.[^10] The View itself is a plain Android `View` / `ViewGroup` subclass; no Fabric-specific base class is needed.

**`android/src/main/java/com/reactnativedeviceinfo/DeviceInfoView.kt`:**
```kotlin
package com.reactnativedeviceinfo

import android.content.Context
import android.widget.TextView

// A simple TextView that will display our label
class DeviceInfoView(context: Context) : TextView(context)
```

**`android/src/main/java/com/reactnativedeviceinfo/DeviceInfoViewManager.kt`:**
```kotlin
package com.reactnativedeviceinfo

import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.ViewManagerDelegate

// Generated delegate + interface live in com.facebook.react.viewmanagers.
import com.facebook.react.viewmanagers.DeviceInfoViewManagerDelegate
import com.facebook.react.viewmanagers.DeviceInfoViewManagerInterface

@ReactModule(name = DeviceInfoViewManager.NAME)
class DeviceInfoViewManager :
    SimpleViewManager<DeviceInfoView>(),
    DeviceInfoViewManagerInterface<DeviceInfoView> {

    // BaseViewManager.getDelegate() returns ViewManagerDelegate<T>?; declaring the
    // explicit type avoids a Kotlin variance error against the parent signature.
    private val delegate: ViewManagerDelegate<DeviceInfoView> =
        DeviceInfoViewManagerDelegate(this)

    override fun getDelegate(): ViewManagerDelegate<DeviceInfoView> = delegate

    override fun getName(): String = NAME

    override fun createViewInstance(reactContext: ThemedReactContext): DeviceInfoView =
        DeviceInfoView(reactContext)

    // The interface declares this; the @ReactProp annotation is what wires
    // up the legacy / paper code path. Both are kept for backwards compat.
    @com.facebook.react.uimanager.annotations.ReactProp(name = "label")
    override fun setLabel(view: DeviceInfoView, label: String?) {
        view.text = label
    }

    companion object {
        const val NAME = "DeviceInfoView"
    }
}
```

Notes worth flagging:

- Drop the constructor-injected `ThemedReactContext`. RN passes the context to `createViewInstance`; storing it on the manager produces a leak across surface reloads.
- `getDelegate()`'s return type must be `ViewManagerDelegate<DeviceInfoView>` (the parent's signature). Returning the concrete delegate class works in Java but Kotlin's stricter variance check sometimes complains; the explicit field type sidesteps it.
- The generated `DeviceInfoViewManagerInterface` already declares `setLabel`. The override is enough; the `@ReactProp` annotation is only needed for the legacy bridge path and is currently still required for prop discovery on Paper.

Finally, the manager needs to be included in your `BaseReactPackage` (or `TurboReactPackage`) via `createViewManagers`:

```kotlin
class DeviceInfoPackage : BaseReactPackage() {
    // ...module wiring as in §2.3 above...

    override fun getViewManagers(reactContext: ReactApplicationContext) = listOf(
        com.facebook.react.bridge.ModuleSpec.viewManagerSpec({ DeviceInfoViewManager() })
    )
}
```

## Part 4: Using the Library

Finally, create an `index.ts` file in your `src` directory to export the components for consumers of your library.

**`src/index.ts`:**
```typescript
import DeviceInfoModule from './specs/NativeDeviceInfoModule';
import DeviceInfoView from './specs/DeviceInfoViewNativeComponent';

export { DeviceInfoModule, DeviceInfoView };

// Example Usage in an app:
//
// import { DeviceInfoModule, DeviceInfoView } from 'react-native-deviceinfo';
//
// const name = await DeviceInfoModule.getDeviceName();
// <DeviceInfoView label={`Device: ${name}`} style={{ width: '100%', height: 30 }} />
```

## Summary of Language Roles

-   **TypeScript:** Defines the public API contract for both modules and components. It is the single source of truth.
-   **CodeGen:** The invisible tool that reads TypeScript and generates the native C++, Java, and Objective-C++ interfaces.
-   **C++:** The language of the JSI and the core of the New Architecture. It's what the generated interfaces are written in.
-   **Objective-C++ (`.mm`):** The essential "glue" language on iOS. It's required to communicate between the C++ world (JSI, CodeGen) and the Objective-C world (`UIKit`).
-   **Swift:** Can be used for pure logic on iOS, but it must be called from an Objective-C++ file.
-   **Kotlin / Java:** The languages for implementing the logic on Android. Kotlin is the modern standard.

---

**Citations:**

[^1]: `codegenConfig` schema (fields, `type` values, `jsSrcsDir`): see `packages/react-native/scripts/codegen/generate-artifacts-executor/index.js` and the working fixture at `packages/react-native/scripts/codegen/__fixtures__/test-app/lib/test-library/package.json`. The `type` field accepts `"all" | "modules" | "components"` (see `generateRCTModuleProviders.js:59` and `generateRCTThirdPartyComponents.js:52`). External docs: "Using Codegen", [reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen). `podspecName` is not a standard field in any RN source path under `packages/`.

[^2]: TurboModule type and registry: `packages/react-native/Libraries/TurboModule/RCTExport.{js,d.ts}` (defines `export interface TurboModule`) and `packages/react-native/Libraries/TurboModule/TurboModuleRegistry.{js,d.ts}` (defines `getEnforcing<T extends TurboModule>(name): T`). Canonical spec example: `packages/react-native/src/private/specs_DEPRECATED/modules/NativeSampleTurboModule.js`. Haste-module-name derivation is the basename of the spec file. External docs: "Turbo Native Modules", [reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction).

[^3]: `<RCTBridgeModule>`-only public header pattern: `packages/react-native/Libraries/Settings/RCTSettingsManager.h:12` (public) and `RCTSettingsManager.mm:18` (`@interface RCTSettingsManager () <NativeSettingsManagerSpec>` in a class extension).

[^4]: Promise codegen for ObjC++: `packages/react-native-codegen/src/generators/modules/GenerateModuleObjCpp/serializeMethod.js:111-115` pushes `{paramName: 'resolve', objCType: 'RCTPromiseResolveBlock'}` and `{paramName: 'reject', objCType: 'RCTPromiseRejectBlock'}` when the return type is a `PromiseTypeAnnotation`. The typedefs live in `packages/react-native/React/Base/RCTBridgeModule.h:42` (`typedef void (^RCTPromiseResolveBlock)(id result);`) and `:49` (`typedef void (^RCTPromiseRejectBlock)(NSString *code, NSString *message, NSError *error);`). The codegen snapshot at `packages/react-native-codegen/src/generators/modules/__tests__/__snapshots__/GenerateModuleHObjCpp-test.js.snap:803-804` shows the generated `@protocol` form.

[^5]: Generated Android spec class name: `packages/react-native-codegen/src/generators/modules/GenerateModuleJavaSpec.js:570` (`const className = \`${hasteModuleName}Spec\`;`). The generated class also already provides `getName()` returning a static `NAME` constant, confirmed by the snapshot at `packages/react-native-codegen/src/generators/modules/__tests__/__snapshots__/GenerateModuleJavaSpec-test.js.snap:75-85`.

[^6]: `TurboReactPackage` deprecation: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/TurboReactPackage.kt:10-14` (`@Deprecated(message = "Use BaseReactPackage instead", replaceWith = ReplaceWith("BaseReactPackage"))`). The replacement landed in the v0.76 cycle: see `CHANGELOG-0.7x.md:1262` ("turbomodules: Replace TurboReactPackage with BaseReactPackage", commit `e881a1184c`). RN itself uses `BaseReactPackage` (`packages/react-native/ReactAndroid/src/main/java/com/facebook/react/runtime/CoreReactPackage.kt:48`).

[^7]: `ReactModuleInfo` constructor evolution: `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/module/model/ReactModuleInfo.kt:16-23` (current 6-arg primary ctor: name, className, canOverrideExistingModule, needsEagerInit, isCxxModule, isTurboModule). The 7-arg overload at `:24-42` is `@Deprecated`, and the `hasConstants` parameter is `@Suppress("UNUSED_PARAMETER")`-marked so it is silently ignored.

[^8]: `RCTViewComponentView`'s declaration and `updateProps:oldProps:` signature: `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.h:25` (`@interface RCTViewComponentView : UIView <RCTComponentViewProtocol, RCTTouchableComponentViewProtocol>`) and `:70-71` (`- (void)updateProps:(const facebook::react::Props::Shared &)props oldProps:(const facebook::react::Props::Shared &)oldProps NS_REQUIRES_SUPER;`). `Props::Shared` is defined as `std::shared_ptr<const Props>` at `packages/react-native/ReactCommon/react/renderer/core/Props.h:29`.

[^9]: Fabric iOS component discovery: the legacy `RCTViewManager` files are wrapped in `#ifndef RCT_REMOVE_LEGACY_ARCH` (e.g. `packages/react-native/Libraries/Image/RCTImageViewManager.mm:10` opens the guard, `:115` closes it). Fabric discovery happens through the plugin map at `packages/react-native/React/Fabric/Mounting/ComponentViews/RCTFabricComponentsPlugins.mm:19-43`, which maps a component name to a `<Name>Cls(void)` C function. Real-world example: `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.mm:1857-1860` (`Class<RCTComponentViewProtocol> RCTViewCls(void) { return RCTViewComponentView.class; }`). The `+ componentDescriptorProvider` class method is the pattern used by every built-in Fabric view (e.g. `packages/react-native/React/Fabric/Mounting/ComponentViews/Switch/RCTSwitchComponentView.mm:65-68`). External docs: "Fabric Native Components", [reactnative.dev/docs/next/fabric-native-components-introduction](https://reactnative.dev/docs/next/fabric-native-components-introduction).

[^10]: Android Fabric component generators: interface package + naming at `packages/react-native-codegen/src/generators/components/__tests__/__snapshots__/GeneratePropsJavaInterface-test.js.snap:14, 21` (`package com.facebook.react.viewmanagers; public interface <Name>ManagerInterface<T extends View> extends ViewManagerWithGeneratedInterface`); delegate at `packages/react-native-codegen/src/generators/components/__tests__/__snapshots__/GeneratePropsJavaDelegate-test.js.snap:14, 24` (`public class <Name>ManagerDelegate<T, U> extends BaseViewManagerDelegate<T, U>`). `SimpleViewManager<T>` itself lives at `packages/react-native/ReactAndroid/src/main/java/com/facebook/react/uimanager/SimpleViewManager.kt:21`.
