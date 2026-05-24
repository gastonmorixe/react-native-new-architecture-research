---
title: "Additional Resources and Checklists"
chapter: "15"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:13:59-04:00"
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
tags: [react-native, new-architecture, checklists, migration, resources, codegen, library-authors]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/15-resources/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:13:59-04:00 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Fixed iOS symbol name (RCTIsNewArchEnabled, not RCTIsNewArchitectureEnabled). Added v0.82 opt-out-removal context to dual-architecture guidance. Repaired two broken reactnative.dev URLs. Added RN-Tester path and noted RNNewArchitectureLibraries archive status. See _verification/chapters/15-resources/report.md."
---

# Chapter 16: Additional Resources and Checklists

This chapter consolidates useful checklists and resource lists from various guides to serve as a quick reference.

## Common Migration Challenges

When migrating an existing application, be aware of these common challenges:

1.  **Third-party Library Compatibility**: Not all third-party native libraries have been updated for the New Architecture. This is often the biggest hurdle. Always check a library's documentation for compatibility.
2.  **Learning Curve**: Understanding the new concepts, especially the JSI, C++ types, and the new threading model, requires a learning investment.
3.  **Build Configuration**: The New Architecture requires specific flags and build setups (`gradle.properties`, `Podfile`). Incorrect configuration is a common source of errors. On React Native 0.82 and newer, `newArchEnabled=false` in `gradle.properties` is rejected by the React Native Gradle Plugin with an explicit error, and `RCT_NEW_ARCH_ENABLED=0` in the iOS environment is ignored (the plugin still prints a warning).[^5]
4.  **Testing Coverage**: Migrating can introduce subtle behavioral changes. Thorough end-to-end testing is critical to ensure all features work as expected.

## Recommended Development Tools

The following tools can be helpful when working with the New Architecture:

-   **`react-native-builder-bob`**: A scaffolding tool that can help create the boilerplate for a new native module or component library. Maintained by Callstack.[^6]
-   **`RN-Tester`**: The official React Native tester app within the main repository, at `packages/rn-tester/`. It bundles three end-to-end New Architecture examples that are useful as references: `NativeModuleExample/` (Objective-C++ TurboModule), `NativeCxxModuleExample/` (pure C++ TurboModule, also runs on Android via JNI), and `NativeComponentExample/` (Fabric native component).[^7]
-   **Community Example Repositories**: The original step-by-step gallery lived at [`react-native-community/RNNewArchitectureLibraries`](https://github.com/react-native-community/RNNewArchitectureLibraries). That repo was archived in March 2024, so use it for the *recipes*, not as a moving target. The maintained successor is the working-group repo [`reactwg/react-native-new-architecture`](https://github.com/reactwg/react-native-new-architecture); its `docs/` folder has been folded into the official docs but the discussion archive there is still the best place to find migration war stories from library authors.[^8]

## Library Migration Checklist

For authors of libraries that contain native code, the migration process involves a few extra steps to ensure consumers of your library have a smooth experience.

-   **[ ] Ship CodeGen Specs:** Your TypeScript or Flow spec files must be included in your published npm package. CodeGen reads each linked library's `codegenConfig.jsSrcsDir` at build time, so if the specs are excluded by `.npmignore` or `files`, the consuming app's build will fail to generate the glue code.
-   **[ ] Add `codegenConfig` to `package.json`:** This is essential so the consuming app's build process knows how to run CodeGen on your library's specs.[^2] The schema is `{ name, type, jsSrcsDir, android, ios, includesGeneratedCode }`, modeled on the Android side at `packages/gradle-plugin/shared/src/main/kotlin/com/facebook/react/model/ModelCodegenConfig.kt`.[^9] See `packages/react-native-popup-menu-android/package.json` for a single-platform example or `packages/react-native/package.json` for the multi-library shape (`codegenConfig.libraries[]`).
-   **[ ] Implement Both Architectures (Only if You Support RN ≤ 0.81):** Before React Native 0.82, library authors used the `BuildConfig.IS_NEW_ARCHITECTURE_ENABLED` flag on Android and the `RCTIsNewArchEnabled()` utility on iOS to register the correct module implementation at runtime.[^1] Starting with React Native 0.82, opt-out from the New Architecture was removed: on iOS `RCTIsNewArchEnabled()` is hardcoded to return `YES`,[^1] and on Android the React Native Gradle Plugin hardcodes the `IS_NEW_ARCHITECTURE_ENABLED` `buildConfigField` to `"true"`.[^10] So this checklist item only matters if your library still ships releases for users pinned to RN ≤ 0.81; for libraries that already require RN ≥ 0.82, drop the conditional and ship the New Arch implementation only.
-   **[ ] Provide a `Podspec` for iOS:** The `.podspec` file for your library must correctly declare its dependencies and source files. Working references in RN itself: `packages/rn-tester/NativeModuleExample/ScreenshotManager.podspec`, `NativeComponentExample/MyNativeView.podspec`, `NativeCxxModuleExample/NativeCxxModuleExample.podspec`.
-   **[ ] Provide `build.gradle` for Android:** Your Android library's `build.gradle` must apply the React Native Gradle Plugin so CodeGen runs and the generated C++ JNI sources link into your library's CMake build.[^3] `packages/react-native-popup-menu-android/android/build.gradle` is the closest in-tree example.
-   **[ ] Document Compatibility:** Clearly state in your `README.md` which versions of your library support the New Architecture and what versions of React Native they are compatible with.[^4] If you still ship pre-0.82 support, document the minimum RN version each major library version targets.
-   **[ ] Add CI for the Versions You Actually Support:** Set up CI to build and test your library's example app against every (RN version × architecture) combination you advertise. For libraries that only target RN ≥ 0.82, that means just one matrix axis (the New Architecture); for libraries that still claim RN ≤ 0.81 support, exercise both arches on those older versions to catch regressions.

---

**Citations:**

[^1]: `packages/react-native/React/Base/RCTUtils.h:18` declares `RCT_EXTERN BOOL RCTIsNewArchEnabled(void);`. The implementation in `packages/react-native/React/Base/RCTUtils.mm:46-49` is hardcoded to `return YES;` since commit `83e6eaf693f` ("Prevent users from opting-out of the New Architecture", PR #53026, first released in v0.82.0-rc.0, 2025-08-05).
[^2]: "Using Codegen". React Native Documentation. [https://reactnative.dev/docs/next/the-new-architecture/using-codegen](https://reactnative.dev/docs/next/the-new-architecture/using-codegen)
[^3]: "React Native Gradle Plugin". React Native Documentation. [https://reactnative.dev/docs/react-native-gradle-plugin](https://reactnative.dev/docs/react-native-gradle-plugin)
[^4]: "About the New Architecture". React Native Documentation. [https://reactnative.dev/architecture/landing-page](https://reactnative.dev/architecture/landing-page)
[^5]: `packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/ReactRootProjectPlugin.kt:61-84` (Android error on `newArchEnabled=false`) and the v0.82 CHANGELOG entry "**New Architecture:** Add warning if `RCT_NEW_ARCH_ENABLED` is set to 0" (commit `7d0bef2f25`).
[^6]: `react-native-builder-bob`. Callstack. [https://github.com/callstack/react-native-builder-bob](https://github.com/callstack/react-native-builder-bob)
[^7]: RN-Tester native examples: `packages/rn-tester/NativeModuleExample/`, `packages/rn-tester/NativeCxxModuleExample/`, `packages/rn-tester/NativeComponentExample/`.
[^8]: `react-native-community/RNNewArchitectureLibraries` (archived 2024-03-20): [https://github.com/react-native-community/RNNewArchitectureLibraries](https://github.com/react-native-community/RNNewArchitectureLibraries). Successor working group: [https://github.com/reactwg/react-native-new-architecture](https://github.com/reactwg/react-native-new-architecture).
[^9]: `packages/gradle-plugin/shared/src/main/kotlin/com/facebook/react/model/ModelCodegenConfig.kt:10-16` (`data class ModelCodegenConfig(name, type, jsSrcsDir, android, includesGeneratedCode)`).
[^10]: `packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/utils/AgpConfiguratorUtils.kt:63` injects `buildConfigField("boolean", "IS_NEW_ARCHITECTURE_ENABLED", "true")` unconditionally. Hardcoded by commit `d5d21d06149` ("Remove possibility to newArchEnabled=false in 0.82", PR #53025, first released in v0.82.0-rc.0, 2025-08-05).
