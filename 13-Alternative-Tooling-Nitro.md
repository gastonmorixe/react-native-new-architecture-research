# Chapter 14: Alternative Tooling - Nitro

While the standard New Architecture tools (CodeGen, TurboModules, Fabric) provide the fundamental building blocks, the ecosystem is evolving with higher-level tooling designed to simplify the developer experience and push performance even further. One prominent example of this is **Nitro**.

Nitro is an opinionated, high-performance framework for building native modules that sits on top of the JSI, acting as an alternative and enhancement to the standard TurboModule workflow.[^1]

## Core Concepts of Nitro

Nitro's philosophy is centered around providing a truly object-oriented and type-safe bridge between JavaScript and native code.

### The `HybridObject`

The central abstraction in Nitro is the `HybridObject`. While a standard TurboModule is a singleton, a `HybridObject` is a true, instantiable C++ object that can be created and referenced from JavaScript.[^2] This allows for a more powerful, object-oriented programming model.

For example, instead of passing an ID or a file path to a native function, you can pass an actual native object instance:

```typescript
// With a TurboModule, you might do this:
const processedImagePath = await ImageEditor.crop(imagePath, rect);

// With Nitro's HybridObjects, you can do this:
const image = await ImageFactory.loadImage(imagePath);
const processedImage: Image = ImageEditor.crop(image, rect);
```

This allows developers to work with complex native object instances directly, which is a more intuitive and powerful paradigm.

### The `nitrogen` Code Generator

Nitro comes with its own code generator, `nitrogen`. It serves the same purpose as React Native's built-in `codegen`—reading a JavaScript spec and generating native interface code—but with a key philosophical difference:[^3]

-   **Standard `codegen`:** Runs as part of the **app's** build process. The app is responsible for generating the code for all its native dependencies.
-   **`nitrogen`:** Is designed to be run by the **library author**. The generated C++, Swift, and Kotlin interface code is then packaged and published with the library, ensuring that the consumer of the library has a stable, pre-generated interface to code against.

## Key Differentiators and Performance Claims

Nitro makes several claims about its advantages over the standard TurboModule system, backed by technical choices and benchmarks.

**1. Performance**

Nitro's documentation presents benchmarks showing its `HybridObject` method calls to be significantly faster than TurboModule method calls for high-throughput scenarios.[^4] It attributes this to two main architectural decisions:

-   **`jsi::NativeState` vs. `jsi::HostObject`:** Standard TurboModules are implemented using `jsi::HostObject`. Nitro's `HybridObject`s are built using `jsi::NativeState`. Nitro claims that `NativeState` allows for better caching of properties by the JavaScript runtime and provides more accurate memory pressure information to the garbage collector, leading to more efficient object management.[^4]
-   **Direct Swift & C++ Interop:** For iOS development, Nitro leverages the modern Swift 5.9 C++ interoperability features. This allows the JSI C++ layer to call Swift code directly, completely bypassing the Objective-C runtime. This eliminates a layer of bridging, reducing call overhead and simplifying the development process for Swift-focused teams.[^5]

**2. Richer Type Support**

Nitro's tooling is designed to support a wider range of types than the standard `codegen`, including tuples, variants (unions), and callbacks that can return values to the native side.[^6]

## Conclusion on Nitro

Nitro is not a replacement for the New Architecture. Rather, it is a sophisticated, opinionated layer on top of it. It offers a glimpse into the future of the ecosystem, where third-party tools can provide alternative developer experiences and push the boundaries of performance by making different trade-offs.

For developers seeking the absolute highest performance for JS-native communication, or for those who prefer a more object-oriented and Swift-centric development workflow on iOS, Nitro presents a compelling alternative to the standard TurboModule creation process.

---

**Citations:**

[^1]: `docs/docs/what-is-nitro.md`
[^2]: `docs/docs/how-to-build-a-nitro-module.md`
[^3]: `docs/docs/nitrogen.md`
[^4]: `docs/docs/comparison.md`
[^5]: `docs/docs/what-is-nitro.md#uses-jsi-native-state`
[^6]: `docs/docs/types/custom-types.md`
