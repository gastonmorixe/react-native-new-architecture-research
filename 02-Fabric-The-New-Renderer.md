# Chapter 3: Fabric - The New Renderer

Fabric is the name of React Native's modern rendering system, a complete re-implementation of the UI manager that lies at the heart of the New Architecture. Built on top of the JSI, Fabric delivers a more performant, responsive, and interoperable UI layer. It replaces the legacy "UIManager" and fundamentally changes how React Native translates JavaScript component trees into native platform views.

## The Motivation: From UIManager to Fabric

The legacy UIManager operated entirely across the asynchronous Bridge. Every UI update, from creating a view to changing its background color, required a serialized message to be sent from JavaScript to the native side. This architecture led to several problems:

-   **Latency:** The asynchronous nature meant there was no guarantee when a UI update would be processed, leading to dropped frames in complex animations or high-frequency updates.
-   **Layout Thrashing:** Layout was asynchronous. JavaScript could not synchronously read the resulting position or size of a view on screen, making certain UI patterns difficult and inefficient.
-   **Data Overhead:** All props and styles were serialized to JSON, adding significant overhead to every update.

Fabric was designed to solve these issues by creating a more tightly integrated and synchronous rendering pipeline, with a significant portion of the logic moved into a shared C++ core.

## The C++ Core of Fabric

At its heart, Fabric introduces a new data structure that lives entirely in C++: the **Shadow Tree**. This tree is the single source of truth for the state of the UI.

#### 1. `ShadowNode`: The Building Block

The fundamental unit of the Shadow Tree is the `ShadowNode`. As defined in `packages/react-native/ReactCommon/react/renderer/core/ShadowNode.h` [1], it is the C++ representation of a React element. Each `ShadowNode` is an immutable object that contains all the information needed to render a component:

-   **Props:** A shared pointer to a `Props` object.
-   **Children:** A list of shared pointers to its child `ShadowNode`s.
-   **State:** A pointer to a `State` object for stateful components.

Because the entire UI tree now exists in C++, layout calculation and view diffing can happen on a background thread without ever crossing the JS/native boundary.

#### 2. `ComponentDescriptor`: The Shadow Node Factory

Defined in `packages/react-native/ReactCommon/react/renderer/core/ComponentDescriptor.h` [2], a `ComponentDescriptor` is a factory object responsible for creating and managing `ShadowNode` instances of a specific type. For each component (like `View` or `Image`), there is a corresponding `ComponentDescriptor` subclass that knows how to create and clone its specific `ShadowNode`.

```cpp
// From ComponentDescriptor.h
class ComponentDescriptor {
 public:
  // ...
  virtual std::shared_ptr<ShadowNode> createShadowNode(
      const ShadowNodeFragment& fragment,
      const ShadowNodeFamily::Shared& family) const = 0;

  virtual std::shared_ptr<ShadowNode> cloneShadowNode(
      const ShadowNode& sourceShadowNode,
      const ShadowNodeFragment& fragment) const = 0;
  // ...
};
```

#### 3. `ShadowTree`: The Tree of Record

As seen in `packages/react-native/ReactCommon/react/renderer/mounting/ShadowTree.h` [3], this class manages the entire lifecycle of the UI tree. Its most important responsibility is managing **revisions**. A `ShadowTreeRevision` is an immutable snapshot of the root `ShadowNode` at a given point in time. The core operation is `commit`, which takes the current revision and a transaction function to produce a new, updated revision.

## The Three Phases of Fabric Rendering

Fabric's rendering pipeline can be broken down into three distinct phases:

**1. The Render Phase**

This phase begins in JavaScript. When a React component's state changes, React creates a new tree of React Elements. The Fabric renderer intercepts this and, using the JSI, communicates with the C++ core to build a corresponding tree of `ShadowNode`s via their `ComponentDescriptor`s.

**2. The Commit Phase**

Once the new Shadow Tree is constructed, the commit phase begins. This phase is responsible for two critical tasks:

-   **Tree Diffing:** The new Shadow Tree is compared ("diffed") against the previously committed tree to determine the minimal set of changes required.
-   **Layout Calculation:** The Yoga layout engine is run on the new Shadow Tree to calculate the final position and size of every single node. Crucially, this all happens in C++ on a background thread.

After layout is complete, the `ShadowTree` class finalizes the new `ShadowTreeRevision` and schedules it for mounting.

**3. The Mount Phase**

This is where the C++ world meets the host platform. The `MountingCoordinator` takes the layout-complete Shadow Tree and translates it into a series of atomic operations to manipulate the actual native views (`UIView`, `android.view.View`).

On iOS, this involves creating and updating instances of `RCTViewComponentView` (or its subclasses), defined in `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.h` [4]. This native `UIView` has methods that are called by the mounting layer:

```objc
// From RCTComponentViewProtocol.h, which RCTViewComponentView implements
- (void)updateProps:(const facebook::react::Props::Shared &)props
           oldProps:(const facebook::react::Props::Shared &)oldProps;

- (void)updateLayoutMetrics:(const facebook::react::LayoutMetrics &)layoutMetrics
           oldLayoutMetrics:(const facebook::react::LayoutMetrics &)oldLayoutMetrics;
```

-   `updateProps`: Receives the new C++ `Props` object. The native view is responsible for casting this to its specific props type and applying the changes (e.g., `self.backgroundColor = ...`).
-   `updateLayoutMetrics`: Receives the `LayoutMetrics` calculated by Yoga. The native view uses this to update its `frame` and `transform`.

Because this phase is synchronous and operates on a pre-calculated layout, it eliminates the layout thrashing and asynchronicity issues of the old UIManager, resulting in a much smoother and more predictable rendering process.

---

**Citations:**

[1] `packages/react-native/ReactCommon/react/renderer/core/ShadowNode.h`
[2] `packages/react-native/ReactCommon/react/renderer/core/ComponentDescriptor.h`
[3] `packages/react-native/ReactCommon/react/renderer/mounting/ShadowTree.h`
[4] `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.h`