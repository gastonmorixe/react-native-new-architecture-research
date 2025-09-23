# Chapter 5: CodeGen - The Automation Engine

The New Architecture's goals of performance and type safety are enabled by the JSI, Fabric, and TurboModules. However, the glue that binds them together and makes the system practical for developers is **CodeGen**. CodeGen is a build-time tool that automates the creation of the boilerplate interface code required to connect the JavaScript world to the native C++ core, as detailed in the official "Using Codegen" guide.[^2]

Its source code can be found in the repository at `packages/react-native-codegen/` [1].

## The Motivation: Eliminating Manual Glue Code

In the legacy architecture, connecting JavaScript to native required developers to write and maintain two separate interfaces by hand. This manual process was tedious and notoriously error-prone:

-   A typo in a method name would lead to a runtime "red box" error.
-   A mismatched argument type could cause a crash or silent failure.
-   There was no single source of truth; the JavaScript and native code could easily drift out of sync.

CodeGen was created to solve this problem by making the JavaScript spec the **single source of truth** and automating the generation of the native-side interface code.

## The CodeGen Pipeline

The CodeGen process is a classic compiler pipeline that runs during the application's build phase.

**1. Parsing**

The process begins with a developer-written JavaScript or TypeScript file containing a `Spec`.

**2. Schema Generation**

The parser transforms the spec file into a single, language-agnostic JSON object that represents the complete API contract. This intermediate representation must conform to the `CodegenSchema.d.ts` format.

**3. Code Generation**

Finally, this JSON schema is passed to the **generators** (`src/generators/`), which use templates to output strings of platform-specific code.

### A Concrete Example

Let's trace the process for a simple TurboModule.

**Step 1: The Developer writes the Spec in TypeScript.**

```typescript
// NativeMyModule.ts
import type { TurboModule } from 'react-native/Libraries/TurboModule/RCTExport';
import { TurboModuleRegistry } from 'react-native/Libraries/TurboModule/TurboModuleRegistry';

export interface Spec extends TurboModule {
  readonly PI: number;
  getSyncValue(a: number, b: string): string;
  getValueWithPromise(): Promise<boolean>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

**Step 2: CodeGen parses this and generates the C++ Spec Header (`...Spec.h`).**

CodeGen produces a C++ abstract class that represents this interface. The native C++ implementation will inherit from this class.

```cpp
// A simplified version of the generated MyModuleSpec.h

#pragma once

#include <ReactCommon/TurboModule.h>
#include <jsi/jsi.h>

namespace facebook::react {

class JSI_EXPORT NativeMyModuleSpecJSI : public TurboModule {
protected:
  NativeMyModuleSpecJSI(std::shared_ptr<CallInvoker> jsInvoker);

public:
  // Notice the methods from the spec are now C++ virtual functions
  virtual jsi::Value getSyncValue(jsi::Runtime& rt, jsi::Value a, jsi::Value b) = 0;
  virtual jsi::Value getValueWithPromise(jsi::Runtime& rt) = 0;

  // The constant is also exposed
  virtual double get_PI() = 0;
};

} // namespace facebook::react
```

**Step 3: The developer implements the generated interface.**

The native C++ or Objective-C++ code must now provide a concrete implementation for the pure virtual functions defined in the generated header. If the signatures do not match (e.g., if a method is missing or an argument type is wrong), the C++ compiler will throw an error.

## The Benefit: Compile-Time Safety

This automated process is the key to the New Architecture's type safety. By generating the native interfaces from a single JavaScript source of truth, it guarantees that the native implementation cannot drift out of sync with the JavaScript API. This shifts a whole class of fragile, runtime-dependent errors into robust, easy-to-diagnose compile-time errors, making the development of native integrations significantly more reliable and maintainable.

---

**Citations:**

[^1]: `packages/react-native-codegen/`
[^2]: "Using Codegen". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen)
