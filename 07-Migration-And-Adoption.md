# Chapter 7: Migration and Adoption

Understanding the new architecture is the first step; the next is adopting it. For developers with existing React Native applications, this involves a clear migration path. For new applications, the New Architecture is enabled by default as of React Native 0.76, so the process involves learning the new, modern APIs.

This chapter serves as a practical guide to enabling the New Architecture and migrating legacy native modules and components. For a complete, in-depth guide, developers should always refer to the official React Native documentation [1].

## Step 1: Prerequisites and Enabling the New Architecture

Before migrating, ensure your project is updated to a recent version of React Native (0.76+ is recommended). The process of enabling the New Architecture is a simple configuration change.

**For Android:**

In your `android/gradle.properties` file, add or set the following flag:

```properties
newArchEnabled=true
```

**For iOS:**

In your `ios/Podfile`, find the `use_react_native!` call and add the `:new_arch_enabled` flag:

```ruby
use_react_native!(
  :path => config[:reactNativePath],
  # Enable the New Architecture for iOS
  :new_arch_enabled => true
)
```

After updating the `Podfile`, you must run `pod install` from the `ios` directory. With these flags in place, the project will build with Fabric and TurboModules enabled.

## Step 2: Migrating a Legacy Native Module to a TurboModule

The migration process for a native module centers on creating a formal spec and updating the native class to conform to the generated interface. The official documentation provides a detailed guide for this process [2].

**1. Create the JavaScript Spec**

First, define the module's interface in a TypeScript file (e.g., `NativeMyModule.ts`). This file is the new source of truth.

```typescript
import type { TurboModule } from 'react-native/Libraries/TurboModule/RCTExport';
import { TurboModuleRegistry } from 'react-native/Libraries/TurboModule/TurboModuleRegistry';

export interface Spec extends TurboModule {
  getValue(options: { a: string }): Promise<string>;
  getSyncValue(): string;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

**2. Configure CodeGen**

In your library's `package.json`, add a `codegenConfig` section to tell the build system where to find your spec.

**3. Update the Native Implementation**

The native classes must now implement the interface that CodeGen generates from the spec, removing the old `RCT_EXPORT_` macros.

## Step 3: Migrating a Legacy View Manager to a Fabric Component

Migrating a UI component follows a similar pattern: define a spec, configure CodeGen, and update the native implementation. The official documentation also provides a specific guide for Fabric components [3].

**1. Create the JavaScript Spec**

Define the component's props in a TypeScript file.

```typescript
import type { ViewProps } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

export interface MyComponentProps extends ViewProps {
  color: string;
}

export default codegenNativeComponent<MyComponentProps>('MyComponent');
```

**2. Update the Native Implementation**

The native view class (e.g., on iOS) should now inherit from `RCTViewComponentView`. Instead of using `RCT_EXPORT_VIEW_PROPERTY`, it will implement the `updateProps` method, which receives a C++ `Props` object and applies the updates to the native view.

## Step 4: Audit Dependencies

A critical part of the migration is to check third-party libraries that use native code. Most popular libraries have been updated to support the New Architecture, but older or less-maintained ones may not be compatible. It is essential to check the documentation for each library and update to versions that are explicitly New Architecture-ready.

---

**Citations:**

[1] "Migrating to the New Architecture". React Native Documentation. [https://reactnative.dev/docs/new-architecture-intro](https://reactnative.dev/docs/new-architecture-intro)
[2] "TurboModules". React Native Documentation. [https://reactnative.dev/docs/new-architecture-turbomodules](https://reactnative.dev/docs/new-architecture-turbomodules)
[3] "Fabric Native Components". React Native Documentation. [https://reactnative.dev/docs/new-architecture-fabric-components](https://reactnative.dev/docs/new-architecture-fabric-components)