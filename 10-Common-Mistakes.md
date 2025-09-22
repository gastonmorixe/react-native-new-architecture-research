# Chapter 10: Common Mistakes and Misunderstandings

The New Architecture introduces powerful concepts, but also new ways for things to go wrong. Understanding common pitfalls can save significant debugging time. This chapter outlines frequent mistakes and misunderstandings grouped by component.

## General Misunderstandings

**Mistake: Assuming the New Architecture automatically makes existing code faster.**

-   **Why it's a mistake:** Simply enabling the New Architecture flag does not optimize legacy components. An old, Bridge-based native module will still communicate via a compatibility layer. The performance benefits are only realized when a module is migrated to a TurboModule or a component to a Fabric Component.
-   **How to avoid:** Understand that performance gains require active migration of your native code to use the new, JSI-based APIs.

**Mistake: Ignoring third-party library compatibility.**

-   **Why it's a mistake:** Enabling the New Architecture while using a third-party native library that does not support it is a common source of crashes and unexpected behavior. The library may rely on the old Bridge, which is not fully available in Bridgeless mode.
-   **How to avoid:** Before migrating, audit all native dependencies. Check their documentation for "New Architecture," "Fabric," or "TurboModule" support. Update to compatible versions or find alternatives.

## JSI and C++ Mistakes

**Mistake: Blocking the JavaScript thread with a long-running synchronous method.**

-   **Why it's a mistake:** The ability to create synchronous methods is powerful, but it's also dangerous. If a synchronous native method performs file I/O, network requests, or heavy computation, it will completely block the JS thread, freezing your app's UI.
-   **How to avoid:** Reserve synchronous methods for only the most trivial, fast operations (e.g., reading a pre-calculated value from memory). For everything else, use Promises.

**Mistake: Ignoring JSI thread safety.**

-   **Why it's a mistake:** A `jsi::Runtime` instance is not thread-safe. Calling JSI methods for the same runtime from multiple threads without proper locking will lead to memory corruption and crashes.
-   **How to avoid:** Ensure all JSI interactions for a given runtime happen on a single, dedicated thread. If you need to pass data between threads, use thread-safe mechanisms and dispatch back to the JSI thread to interact with JavaScript.

**Mistake: Performing complex operations in a `HostObject` destructor.**

-   **Why it's a mistake:** The destructor for a C++ `HostObject` is called by the JavaScript engine's garbage collector. This can happen on any thread, at any time, and it is explicitly unsafe to make any JSI calls from within the destructor.
-   **How to avoid:** The destructor should only be used for trivial cleanup of the C++ object's own memory. If you need to perform more complex cleanup that involves other systems or the JSI, you must implement an explicit `invalidate()` or `cleanup()` method on your Host Object that you call from JavaScript before releasing its reference.

## TurboModule Mistakes

**Mistake: Module name mismatch.**

-   **Why it's a mistake:** JavaScript finds a TurboModule by its name. If the name you provide in `TurboModuleRegistry.getEnforcing<Spec>('MyModule')` does not exactly match the name provided in the native implementation (e.g., in the `ReactPackage` on Android or the `getTurboModule` registration on iOS), the module will not be found, and your app will crash on startup.
-   **How to avoid:** Ensure the module name is identical across the JS spec and all native registration points.

## Fabric Component Mistakes

**Mistake: Treating `updateProps` as a constructor or a one-time setup.**

-   **Why it's a mistake:** The `updateProps` method on a native component view will be called many times throughout the component's lifecycle—every time a prop changes. Performing expensive, one-time setup work inside it is inefficient.
-   **How to avoid:** Use the view's native initializer (e.g., `init` on iOS) for one-time setup. Use `updateProps` only for applying the new properties to the view.

**Mistake: Passing frequently changing values as props.**

-   **Why it's a mistake:** While Fabric is much more efficient than the old UIManager, every prop change still results in a call from C++ to the native view. If you are passing down values from an animation that updates every frame (e.g., from `useAnimatedStyle` in Reanimated), this can still create significant traffic.
-   **How to avoid:** For high-frequency updates like animations, use libraries like Reanimated that are designed to perform the animations directly on the UI thread, bypassing the React render cycle entirely. For one-off imperative actions, use commands instead of props.

## CodeGen Mistakes

**Mistake: Forgetting to rebuild after changing a spec.**

-   **Why it's a mistake:** CodeGen runs during the build process. If you change a JavaScript spec file (e.g., add a new method to a TurboModule), the native interface code is not automatically updated. You must re-run the build (`pod install` for iOS, or a Gradle build for Android) to regenerate the native files.
-   **How to avoid:** After any change to a `Spec` file, perform a clean build of your native project to ensure the generated code is up-to-date.

**Mistake: Incorrect `codegenConfig` in `package.json`.**

-   **Why it's a mistake:** If the `jsSrcsDir` or other paths in the `codegenConfig` are incorrect, the CodeGen script won't find your spec files and will not generate any code, leading to compilation errors.
-   **How to avoid:** Double-check all paths and package names in the `codegenConfig` section to ensure they are correct.
