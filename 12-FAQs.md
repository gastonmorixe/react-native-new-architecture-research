# Chapter 12: Frequently Asked Questions (FAQs)

This chapter addresses common high-level questions about the React Native New Architecture.

**1. Do I have to migrate to the New Architecture?**

For now, the legacy architecture is still supported. However, as of React Native 0.76, the New Architecture is the default for new projects. All future development of React Native core and new React features (like Concurrent Rendering) will be built for the New Architecture. It is highly recommended to migrate to stay current and ensure the future viability of your application.

**2. Is the Bridge completely gone in the New Architecture?**

In **Bridgeless Mode** (the default for new architecture apps since RN 0.74), the original message-queue Bridge is no longer initialized or used. However, an **interoperability layer** still exists to support legacy native modules. So, while the old performance-bottleneck Bridge is gone, the system can still communicate with old modules.

**3. Can I still use my old native modules and components?**

Yes, for the most part. The interoperability layer allows legacy `RCTBridgeModule`s and `ViewManager`s to function. However, they will not see the performance benefits of the new architecture. For full compatibility and performance, they should be migrated to TurboModules and Fabric Components.

**4. Do I need to learn C++ to use the New Architecture?**

Not necessarily to *use* it, but it is highly beneficial if you are *writing* native modules. You can write TurboModules and Fabric Components using only Objective-C or Java/Kotlin, as CodeGen handles the C++ interface generation. However, understanding the C++ layer (especially JSI concepts like `HostObject`) is crucial for debugging and for unlocking advanced performance patterns, such as writing a single, cross-platform module in C++.

**5. What is the difference between a TurboModule and a Fabric Component?**

-   **TurboModules** are for non-UI native functionality. They replace the old `NativeModules`. Examples include accessing the device's calendar, camera, or file system.
-   **Fabric Components** are for UI. They replace the old `ViewManager`s. They are native views that are rendered on the screen, like a map view or a custom video player.

**6. Why is my new TurboModule `undefined` in JavaScript?**

This is a common issue during migration. The most frequent causes are:

-   **Name Mismatch:** The name used in `TurboModuleRegistry.getEnforcing('MyModule')` in JavaScript does not exactly match the name registered in the native code (e.g., in the `ReactPackage` on Android).
-   **CodeGen Not Run:** You have created or changed the JS `Spec` file but have not rebuilt the app. You must re-run `pod install` or rebuild your Gradle project to trigger CodeGen and link the new module.
-   **Module Not Registered:** The module has not been properly registered in your Android `ReactPackage` or your iOS `Podfile`.

**7. Is the New Architecture always faster than the old one?**

For most real-world applications, yes. The benefits in app startup (from lazy loading), UI responsiveness (from Fabric), and native call speed (from JSI) are significant. However, in some very simple, static rendering scenarios, the overhead of the new architecture's more complex machinery can make it appear marginally slower than the legacy system. The true benefits shine in complex applications with many native interactions and dynamic UIs.
