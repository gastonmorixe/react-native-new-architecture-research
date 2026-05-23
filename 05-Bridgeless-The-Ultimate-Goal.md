# Chapter 6: Bridgeless - The Ultimate Goal

The introduction of the JSI, Fabric, and TurboModules provides the foundational pieces to replace the legacy architecture. However, for a time, both the old and new systems could coexist in a single application to provide backward compatibility. The ultimate goal of the new architecture, however, was to completely remove the old one. This final state is known as **Bridgeless Mode**.

This mode was first made available as an opt-in with React Native 0.73 and became the default for New Architecture-enabled apps in React Native 0.74 [1].

## What is Bridgeless Mode?

Bridgeless mode is not a new component or a separate piece of technology. It is the architectural state where the original, asynchronous message-based Bridge is **no longer initialized or used**. In this mode, all communication between JavaScript and the native platform flows exclusively through the JavaScript Interface (JSI).

In a non-bridgeless (but still New Architecture-enabled) app, the Bridge might still exist to support legacy native modules that haven't been migrated to TurboModules. In Bridgeless mode, this compatibility layer is still present, but the core message queue and its associated overhead are gone. All interactions, whether with a modern TurboModule or a legacy module via the interop layer, are ultimately funneled through the synchronous, direct JSI path.

## Visual: Bridgeless vs Non‑Bridgeless

```mermaid
flowchart TB
  subgraph NA[New Architecture Enabled]
    JSI[JSI Runtime]
    TM[TurboModules]
    Interop[Interop Layer]
    Legacy[RCTBridgeModule (legacy)]
  end

  JSI --> TM
  JSI --> Interop --> Legacy

  note left of NA: Bridgeless default since 0.74 when NA is enabled\nLegacy Bridge process is not initialized
```

## The Benefits of a Bridgeless World

Running without the Bridge unlocks the full potential of the New Architecture, leading to several key benefits:

1.  **Improved Performance:** This is the most significant advantage. By removing the need to initialize the Bridge and its modules at startup, apps launch faster. By eliminating the serialization and queueing of messages, communication latency is drastically reduced, leading to more responsive UI and faster data exchange.

2.  **Simplified Architecture and Threading:** Without the Bridge, the mental model for developers becomes simpler. There is no longer a complex, asynchronous boundary to reason about. The threading model is more direct, with a clear path from the JavaScript thread to the native UI and background threads via the JSI.

3.  **Unlocking Modern React Features:** The asynchronous nature of the legacy Bridge was fundamentally incompatible with modern React features like Concurrent Rendering and Suspense. Bridgeless mode, with its synchronous JSI foundation, allows these features to work in React Native just as they do in React for the web.

## The Interoperability Layer

The React Native team understood that a hard cutover to the new architecture was not feasible for the vast ecosystem of existing apps and third-party libraries. To facilitate a gradual migration, an interoperability layer was created.

This layer allows legacy, Bridge-based native modules to continue functioning in a Bridgeless world. When JavaScript attempts to call a method on a legacy module, the JSI intercepts this call and routes it through a compatibility layer that mimics the behavior of the old Bridge, but without the global message queue. While this allows for backward compatibility, it does not provide the performance benefits of true TurboModules. The long-term goal is for the entire ecosystem to migrate to the TurboModule and Fabric APIs to reap the full benefits of the New Architecture.

In conclusion, Bridgeless mode represents the fulfillment of the New Architecture's promise: a faster, more efficient, and more capable React Native. It marks a definitive shift away from the original design and sets a new foundation for the future of the framework.

---

**Citations:**

[1] "Announcing React Native 0.74". React Native Blog. [https://reactnative.dev/blog/2024/05/22/react-native-074](https://reactnative.dev/blog/2024/05/22/react-native-074)
