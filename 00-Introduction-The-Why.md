# Chapter 1: Introduction - The "Why"

The React Native New Architecture is arguably the most significant evolution in the framework's history. While it became the default in the React Native 0.76 release in 2024 [2], its story begins much earlier. The vision was first shared with the community in a 2018 presentation at React Conf [1]. This marked the beginning of a multi-year effort to fundamentally re-imagine the communication layer between JavaScript and the native host platform.

This was not a minor refactoring but a deep, foundational change designed to address systemic performance bottlenecks and unlock new capabilities for the framework. To fully appreciate the new architecture, one must first understand the limitations of the system it replaced: the original architecture, centered around a component known as "the Bridge."

## The Old Architecture: A Story of the Bridge

In the original architecture, the entire communication system between the JavaScript thread (where the application's business logic runs) and the Native thread (where the host platform's UI and services run) was funneled through a single, asynchronous message queue: **the Bridge**.

This design, while functional, had several inherent limitations that became more pronounced as React Native applications grew in complexity:

1.  **Asynchronous Communication:** Every call from JavaScript to a native module was, by necessity, asynchronous. When JavaScript needed to invoke a native function (e.g., to access the camera or display a native view), it would place a serialized message onto the Bridge. The native side would eventually pick up this message, execute the function, and, if a result was needed, send a message back. This round-trip introduced latency and made direct, synchronous interaction impossible.

2.  **Serialization Overhead:** All data sent across the Bridge had to be serialized into a JSON string. This meant that complex JavaScript objects had to be converted to strings, sent over, and then parsed back into their native equivalents (e.g., `NSDictionary` on iOS or `ReadableMap` on Android). This process of serialization and deserialization added significant computational overhead, especially for large data payloads or frequent updates.

3.  **Congestion and Bottlenecks:** The Bridge acted as a single lane of traffic. High-frequency updates, such as tracking a user's gesture on the screen, could flood the bridge with messages, leading to dropped frames, UI stutter, and a generally unresponsive feel. The batched nature of the bridge meant there were no guarantees about when a message would be processed, making fine-grained control difficult.

## The Vision: A New Foundation for React Native

The vision for the New Architecture was to tear down these limitations and create a more performant, tightly integrated, and capable framework. The core goals, as articulated by the React Native team and community collaborators, were:

1.  **Performance:** To dramatically improve performance by enabling direct, synchronous communication between JavaScript and the native platform, eliminating the serialization overhead of the Bridge.

2.  **Type Safety:** To create a system where the "contract" between JavaScript and native code is strictly defined and enforced at compile time. This reduces runtime errors and makes cross-language refactoring safer.

3.  **Concurrency:** To align React Native with the latest advancements in the React ecosystem, such as concurrent rendering, enabling more fluid and responsive user interfaces.

4.  **Improved Interoperability:** To make it easier and more efficient to integrate React Native with existing native codebases and to create complex, high-performance native UI components.

To achieve this vision, the team built the new architecture on three core pillars, which the subsequent chapters of this report will explore in detail:

*   **The JavaScript Interface (JSI):** The new foundational layer that replaces the Bridge, allowing for direct, synchronous method calls between the two realms.
*   **Fabric:** The new rendering system that leverages JSI to create a more responsive and efficient UI layer.
*   **TurboModules:** The next generation of native modules, also built on JSI, which are loaded on-demand and can be invoked synchronously from JavaScript.

---

**Citations:**

[1] Parashuram N, "React Native's New Architecture" (React Conf 2018). [https://www.youtube.com/watch?v=U-i-7I8uSJE](https://www.youtube.com/watch?v=U-i-7I8uSJE)
[2] "About the New Architecture". React Native Documentation. [https://reactnative.dev/docs/next/architecture/landing-page](https://reactnative.dev/docs/next/architecture/landing-page)
