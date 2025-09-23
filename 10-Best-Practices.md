# Chapter 11: Best Practices

Adopting the New Architecture effectively involves more than just avoiding mistakes; it requires embracing new patterns and best practices. This chapter provides a list of recommendations for writing high-quality, performant code.

## General Practices

-   **Do** migrate one module or component at a time. The New Architecture is designed for incremental adoption. This allows you to isolate issues and learn the new patterns in a controlled way.
-   **Don't** attempt a "big bang" migration of a large, complex application all at once. This makes debugging extremely difficult.
-   **Do** keep your JavaScript specs as the single source of truth. All native implementations should strictly follow the CodeGen-generated interfaces.
-   **Do** audit your third-party dependencies early and often. Prefer libraries that explicitly state support for the New Architecture.
-   **Do** consider higher-level tooling where appropriate. For teams that want to simplify the module creation process or extract maximum performance, alternative frameworks like **Nitro** (see Chapter 14) provide abstractions on top of the standard tools and may be a good fit.

## JSI and C++ Practices

-   **Do** use C++ for performance-critical, cross-platform logic. The JSI makes it easier than ever to write a function once in C++ and use it from both iOS and Android, reducing code duplication and improving performance.
-   **Don't** put platform-specific code (e.g., `UIKit` or Android SDK calls) in your core C++ modules. Keep the C++ layer platform-agnostic and use platform-specific adapter code to call into it.
-   **Do** handle threading carefully. If a JSI call needs to perform work off the main thread, dispatch it to a background queue and use the `CallInvoker` to return the result to the JavaScript thread.

## TurboModule Practices

-   **Do** keep synchronous methods trivial and fast. They are perfect for fetching simple, pre-calculated values that are already in memory. [1]
-   **Don't** ever perform I/O (network, disk), heavy computation, or any potentially long-running task in a synchronous TurboModule method. This will block the JS thread and freeze your app. [1]
-   **Do** use Promises for any method that is asynchronous or might take more than a millisecond to complete. This is the standard and expected pattern for async work. [1]
-   **Don't** rely on callbacks. While they may be supported by the interop layer, Promises are the forward-looking standard for async operations in TurboModules.

## Fabric Component Practices

-   **Do** use the `updateProps` method efficiently. It is a diffing mechanism. Use it only to apply the changed properties to your native view. [2]
-   **Don't** perform expensive allocations or setup work inside `updateProps`. Place one-time setup code in the view's native initializer. [2]
-   **Do** use commands for one-off, imperative actions. If you need to trigger an action on a native view (e.g., `focus()`, `clear()`), defining it as a command is much more efficient than changing a prop to trigger the effect. [3]
-   **Don't** use props for ephemeral, event-like triggers. For example, to trigger an animation, it's better to use a command like `myView.startAnimation()` than to toggle a prop like `shouldAnimate={true}`. [3]
-   **Do** colocate your component's related files. A good pattern is to have a single directory containing the JS spec, the iOS implementation (`.mm`), and the Android implementation (`.java`/`.kt`) for a single component. [2]

---

**Citations:**

[1] "Turbo Native Modules". React Native Documentation. [https://reactnative.dev/docs/next/turbo-native-modules-introduction](https://reactnative.dev/docs/next/turbo-native-modules-introduction)
[2] "Fabric Native Components: iOS". React Native Documentation. [https://reactnative.dev/docs/next/fabric-native-components-introduction](https://reactnative.dev/docs/next/fabric-native-components-introduction)
[3] "Fabric Component Native Commands". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/fabric-component-native-commands](https://reactnative.dev/docs/next/the-new-architecture/fabric-component-native-commands)
