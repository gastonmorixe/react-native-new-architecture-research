# Chapter 16: Additional Resources and Checklists

This chapter consolidates useful checklists and resource lists from various guides to serve as a quick reference.

## Common Migration Challenges

When migrating an existing application, be aware of these common challenges:

1.  **Third-party Library Compatibility**: Not all third-party native libraries have been updated for the New Architecture. This is often the biggest hurdle. Always check a library's documentation for compatibility.
2.  **Learning Curve**: Understanding the new concepts, especially the JSI, C++ types, and the new threading model, requires a learning investment.
3.  **Build Configuration**: The New Architecture requires specific flags and build setups (`gradle.properties`, `Podfile`). Incorrect configuration is a common source of errors.
4.  **Testing Coverage**: Migrating can introduce subtle behavioral changes. Thorough end-to-end testing is critical to ensure all features work as expected.

## Recommended Development Tools

The following tools can be helpful when working with the New Architecture:

-   **`react-native-builder-bob`**: A scaffolding tool that can help create the boilerplate for a new native module or component library.
-   **`RN-Tester`**: The official React Native tester app within the main repository. It contains numerous examples of New Architecture components and modules that can be used as a reference.
-   **Community Example Repositories**: Searching GitHub for `RNNewArchitectureLibraries` or similar can reveal community-maintained examples of migrated libraries.

## Library Migration Checklist

For authors of libraries that contain native code, the migration process involves a few extra steps to ensure consumers of your library have a smooth experience.

-   **[ ] Ship CodeGen Specs:** Your TypeScript or Flow spec files must be included in your published npm package.
-   **[ ] Add `codegenConfig` to `package.json`:** This is essential so the consuming app's build process knows how to run CodeGen on your library's specs.[^2]
-   **[ ] Implement Both Architectures (Optional but Recommended):** For maximum compatibility, support both the legacy and New Architecture. You can use the `BuildConfig.IS_NEW_ARCHITECTURE_ENABLED` flag on Android or the `RCTIsNewArchitectureEnabled()` utility on iOS to conditionally register the correct module implementation.[^1]
-   **[ ] Provide a `Podspec` for iOS:** The `.podspec` file for your library must correctly declare its dependencies and source files.
-   **[ ] Provide `build.gradle` for Android:** Your Android library's `build.gradle` must be configured to handle CodeGen and link the native C++ code.[^3]
-   **[ ] Document Compatibility:** Clearly state in your `README.md` which versions of your library support the New Architecture and what versions of React Native they are compatible with.[^4]
-   **[ ] Add CI for Both Architectures:** Set up your continuous integration to build and test your library's example app with both the New Architecture enabled and disabled to prevent regressions.[^4]

---

**Citations:**

[^1]: `packages/react-native/React/Base/RCTUtils.h`
[^2]: "Using Codegen". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen)
[^3]: "React Native Gradle Plugin". React Native Documentation. [https://reactnative.dev/docs/next/architecture/gradle-plugin](https://reactnative.dev/docs/next/architecture/gradle-plugin)
[^4]: "About the New Architecture". React Native Documentation. [https://reactnative.dev/docs/next/architecture/landing-page](https://reactnative.dev/docs/next/architecture/landing-page)
