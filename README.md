---
title: "A comprehensive report on the React Native New Architecture"
chapter: "README"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T16:14:43-0400"
session_id: "audit-worker-readme"
host_info:
  hostname: "macbookpro.home.arpa"
  user: "gaston"
  os: "macOS 26.5 (..)"
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
tags: [react-native, new-architecture, readme, table-of-contents, overview]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/readme/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:14:43-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db). Refreshed the stale '0.82+' version pin to '0.86' and verified link integrity for all 16 chapter files plus the five Mermaid diagrams referenced in the intro. See _verification/chapters/readme/report.md."
---

# A Comprehensive Report on the React Native New Architecture

> A deep-dive research report on the React Native New Architecture, covering core concepts, practical guides, real-world case studies, and more.

This document serves as the central hub for a detailed, 16-chapter report on the significant evolution in the React Native framework. The research is conducted by analyzing the official React Native repository, authoritative documentation, and real-world libraries to provide a comprehensive and up-to-date understanding of the new architecture.

**Updated for React Native 0.76+ (audit pass 2026-05):** This documentation has been thoroughly updated to reflect the current state of the New Architecture, which is now enabled by default in all React Native projects.[^1] The information is verified against the React Native `main` branch at commit `b32a6c9e9db` (v0.86.0-rc.1 era) and official documentation.

Tip: Many chapters now include Mermaid diagrams (rendered on GitHub) to visualize the Bridge vs JSI call paths, Fabric’s render pipeline, TurboModule CodeGen, Bridgeless interop, and the migration flow.

---

## 📚 Table of Contents

### Part 1: Core Architecture

*   [Chapter 1: Introduction - The "Why"](./00-Introduction-The-Why.md)
    *   A deep dive into the limitations of the original architecture and the vision for the new one.
*   [Chapter 2: JSI - The Foundation](./01-JSI-The-Foundation.md)
    *   An exploration of the JavaScript Interface (JSI) and its role as the cornerstone of the new architecture.
*   [Chapter 3: Fabric - The New Renderer](./02-Fabric-The-New-Renderer.md)
    *   A detailed look at the new rendering system that replaces UIManager.
*   [Chapter 4: Turbo Modules - The New Native Modules](./03-Turbo-Modules-The-New-Native-Modules.md)
    *   Understanding the next generation of native modules and their benefits.
*   [Chapter 5: CodeGen - The Automation Engine](./04-CodeGen-The-Automation-Engine.md)
    *   An analysis of the tooling that automates the generation of "glue code."
*   [Chapter 6: Bridgeless - The Ultimate Goal](./05-Bridgeless-The-Ultimate-Goal.md)
    *   Investigating the progress and implications of running React Native without the original Bridge.

### Part 2: Practical Application & Guides

*   [Chapter 7: Migration and Adoption](./06-Migration-And-Adoption.md)
    *   A practical guide for developers on migrating their apps and libraries.
*   [Chapter 8: Performance Analysis](./07-Performance-Analysis.md)
    *   A review of benchmarks and performance improvements.
*   [Chapter 9: Conclusion and Future](./08-Conclusion-And-Future.md)
    *   A summary of the key advancements and a look at the future roadmap.
*   [Chapter 10: Common Mistakes and Misunderstandings](./09-Common-Mistakes.md)
    *   A guide to frequent errors and pitfalls when working with the New Architecture.
*   [Chapter 11: Best Practices](./10-Best-Practices.md)
    *   A list of do's and don'ts for writing performant and maintainable code.
*   [Chapter 12: Frequently Asked Questions (FAQs)](./11-FAQs.md)
    *   A list of common high-level questions and their answers.

### Part 3: Real-World Examples & Tooling

*   [Chapter 13: Case Study - `react-native-vision-camera`](./12-Case-Study-Vision-Camera.md)
    *   An analysis of how a production library blends legacy view managers with JSI frame processors while preparing for the New Architecture.
*   [Chapter 14: Alternative Tooling - Nitro](./13-Alternative-Tooling-Nitro.md)
    *   An exploration of a high-level framework that provides an alternative developer experience for creating native modules.
*   [Chapter 15: Tutorial - Building a Native Module and Component](./14-Tutorial-Building-From-Scratch.md)
    *   A hands-on tutorial building a simple Fabric component and TurboModule from scratch.
*   [Chapter 16: Additional Resources and Checklists](./15-Additional-Resources-and-Checklists.md)
    *   A collection of useful checklists and resources for migration and development.

---

## Footnotes

[^1]: The New Architecture is hard-wired on in the current React Native source. On iOS, `NewArchitectureHelper.new_arch_enabled` in `packages/react-native/scripts/cocoapods/new_architecture.rb:162` returns `true` unconditionally. On Android, the Gradle plugin at `packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/ReactRootProjectPlugin.kt:65` rejects `newArchEnabled=false` with a warning that says "Setting `newArchEnabled=false` in your `gradle.properties` file is not supported anymore since React Native 0.82. (...) The application will run with the New Architecture enabled by default." The default was originally flipped for new apps in v0.76.0 (2024-10-23, see `CHANGELOG-0.7x.md:1151`).
