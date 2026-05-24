---
title: "Conclusion and future"
chapter: "08"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:13:17-0400"
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
tags: [react-native, new-architecture, conclusion, summary, roadmap]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/08-conclusion/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:13:17-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Fixed two broken citation URLs, updated React 18 framing to React 19 (current), clarified the Bridgeless-default-in-0.74 nuance, replaced the single em-dash, added inline source citations for the runtime claims. See _verification/chapters/08-conclusion/report.md."
---

# Chapter 9: Conclusion and Future

Over the course of this report, we have dissected the React Native New Architecture, a multi-year, foundational overhaul designed to carry the framework into the future. The journey from the original, Bridge-based architecture to the modern, JSI-based system was a monumental undertaking driven by the need to solve real-world performance and scalability limitations.

## A Summary of Key Advancements

The New Architecture is not a single feature, but a collection of interconnected components that work in concert:

-   **The JavaScript Interface (JSI)** is the new foundation, replacing the asynchronous Bridge with a direct, synchronous, and efficient C++ API. It eliminates the JSON serialization the old Bridge needed and lets JavaScript and the native runtime exchange handles to each other's objects without copying.[^3]

-   **Fabric** is the modern UI renderer built on the JSI. By moving the Shadow Tree to C++ and adding a multi-priority scheduler that backs Concurrent React, it provides a more responsive and fluid user interface, capable of handling complex updates without blocking the main thread.[^4]

-   **TurboModules** are the next-generation native modules. By leveraging the JSI for synchronous method calls and enabling lazy loading, they dramatically improve app startup time and the performance of native function calls. Every TurboModule is a `jsi::HostObject`, so a JS-side lookup resolves directly into a C++ method call.[^5]

-   **CodeGen** is the automated tooling that provides compile-time type safety. By generating native interface code from a single JavaScript source of truth (the `*NativeModule.ts` / `*NativeComponent.ts` specs), it eliminates an entire class of common runtime errors and makes developing native integrations more robust and maintainable. Generators emit C++ headers, JNI bindings, Java specs, and Objective-C++ stubs from one schema.[^6]

-   **Bridgeless Mode**, the runtime where the legacy message-queue Bridge is not initialized, has been the default *when the New Architecture is enabled* since React Native 0.74.[^1] In practice, most apps first ran in this mode after upgrading to 0.76, since that is the release where the New Architecture itself became the default.[^2] The legacy architecture is being phased out, not retroactively removed from existing apps: an opt-out flag (`newArchEnabled=false` on Android, `RCT_NEW_ARCH_ENABLED=0` for `pod install` on iOS) still exists, and the legacy Bridge remains in the codebase as a compatibility layer for un-migrated modules.[^2]

-   **Concurrent React features** are now fully supported, bringing automatic batching, transitions, and Suspense to React Native applications, and aligning the development experience with web React. Note that React Native's required React version has moved on since the New Architecture launched: 0.76 shipped with React 18, 0.78 (Feb 2025) bumped the peer dependency to React 19, and the current `main` (RN 0.86) pins `"react": "^19.2.3"`.[^7]

## The Impact: A More Performant and Capable Framework

The performance analysis shows that these architectural changes have yielded significant, measurable results. App startup is faster, UI interactions in complex apps are smoother, and the latency of JS-to-native calls has been reduced by orders of magnitude. While the primary goal was to build a better foundation, the performance benefits for most real-world applications are undeniable.

## The Future of React Native

With the New Architecture the default in React Native 0.76+ (shipped 2024-10-23), the foundation is set for the next era of React Native development. Bridgeless mode has been the default *within* the New Architecture since 0.74, and the legacy Bridge remains only for interop via compatibility layers. We can expect to see:

-   **Deeper Integration with Concurrent React:** Features like Suspense for data fetching, which were previously difficult to implement, can now be fully realized, leading to more sophisticated and user-friendly loading and transition experiences.
-   **Further Performance Optimizations:** With the core bottlenecks removed, the community and the React Native team can now focus on more granular performance tuning at the JSI, Fabric, and TurboModule levels.
-   **A More Stable Developer Experience:** The move towards type-safe, spec-driven development will continue, making the creation of robust native modules and components more accessible and reliable.
-   **Web Alignment:** Active development is underway to align React Native more closely with web React, including a well-defined event loop modelled after the HTML spec, DOM-style Node and layout APIs (`getBoundingClientRect`, traversal), and Yoga-driven layout conformance.[^8]
-   **Enhanced Tooling:** The ecosystem continues to evolve with tools like Nitro providing alternative approaches to native module development, pushing the boundaries of performance and developer experience. See Chapter 14 for a detailed treatment.

In conclusion, the React Native New Architecture is a resounding success. It has addressed the core limitations of the past and established a powerful, performant, and scalable foundation that will allow the framework to evolve and thrive for years to come. For any developer in the React Native ecosystem, understanding and embracing this new paradigm is no longer optional. It is the path forward.[^9]

---

**Further Reading:**

- The official React Native documentation is the best source for the latest information on the New Architecture and its APIs. [https://reactnative.dev/docs/getting-started](https://reactnative.dev/docs/getting-started)

**Citations:**

[^1]: "React Native 0.74 - Yoga 3.0, Bridgeless New Architecture, and more". React Native Blog (2024-04-22). Section "New Architecture: Bridgeless by Default" states: "In this release, we are making Bridgeless Mode the default when the New Architecture is enabled." [https://reactnative.dev/blog/2024/04/22/release-0.74](https://reactnative.dev/blog/2024/04/22/release-0.74)
[^2]: "React Native 0.76 - New Architecture by default, React Native DevTools, and more". React Native Blog (2024-10-23). [https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture](https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture). Companion post "New Architecture is here" covers the bridge-removal rationale and the gradual-migration / opt-out story: [https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here).
[^3]: JSI `Runtime` class declared in `packages/react-native/ReactCommon/jsi/jsi/jsi.h:705` (`class JSI_EXPORT Runtime : public IRuntime`). Header-only C++ API with synchronous `evaluateJavaScript`, `getProperty`, `setProperty`, `call`, etc.
[^4]: Shadow Tree implemented in `packages/react-native/ReactCommon/react/renderer/core/ShadowNode.h`. The multi-priority scheduler that powers concurrent rendering lives in `packages/react-native/ReactCommon/react/renderer/runtimescheduler/RuntimeScheduler_Modern.h:21` (`class RuntimeScheduler_Modern final : public RuntimeSchedulerBase`).
[^5]: `packages/react-native/ReactCommon/react/nativemodule/core/ReactCommon/TurboModule.h:46`: `class JSI_EXPORT TurboModule : public jsi::HostObject`. The `get(jsi::Runtime&, const jsi::PropNameID&)` override returns a `jsi::Value` synchronously.
[^6]: Generators in `packages/react-native-codegen/src/generators/modules/`: `GenerateModuleH.js` (C++), `GenerateModuleJavaSpec.js`, `GenerateModuleJniCpp.js`, `GenerateModuleJniH.js`, `GenerateModuleObjCpp/`. All driven from `*NativeModule.{ts,js.flow}` specs parsed by `packages/react-native-codegen/src/parsers/`.
[^7]: React peer dependency in `packages/react-native/package.json`: `"react": "^19.2.3"`. React 19 was first required by RN 0.78 ("**Deps:** Fix peer dependencies on React types to React 19" in CHANGELOG-0.7x.md's v0.78.0 section, released 2025-02-19).
[^8]: Event loop alignment documented in `packages/react-native/ReactCommon/react/renderer/runtimescheduler/__docs__/README.md`, which links the implementation to the [HTML spec event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) and the [well-defined event loop RFC](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0744-well-defined-event-loop.md). DOM-style APIs (`getBoundingClientRect`, etc.) live in `packages/react-native/src/private/webapis/dom/`.
[^9]: "About the New Architecture". React Native Documentation. [https://reactnative.dev/architecture/landing-page](https://reactnative.dev/architecture/landing-page).
