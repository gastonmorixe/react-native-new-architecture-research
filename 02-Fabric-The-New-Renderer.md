# Chapter 3: Fabric - The New Renderer

Fabric is the name of React Native's modern rendering system, a complete re-implementation of the UI manager that lies at the heart of the New Architecture. Built on top of the JSI, Fabric delivers a more performant, responsive, and interoperable UI layer. It replaces the legacy "UIManager" and fundamentally changes how React Native translates JavaScript component trees into native platform views.

**Current Status (2025):** Fabric is now the default renderer in React Native 0.76+. It powers all React Native applications and provides significant performance improvements, especially for complex UIs and animations. The legacy UIManager is deprecated and will be removed in future versions.

## The Motivation: From UIManager to Fabric

The legacy UIManager operated entirely across the asynchronous Bridge. Every UI update, from creating a view to changing its background color, required a serialized message to be sent from JavaScript to the native side. This architecture led to several problems:

-   **Latency:** The asynchronous nature meant there was no guarantee when a UI update would be processed, leading to dropped frames in complex animations or high-frequency updates.
-   **Layout Thrashing:** Layout was asynchronous. JavaScript could not synchronously read the resulting position or size of a view on screen, making certain UI patterns difficult and inefficient.
-   **Data Overhead:** All props and styles were serialized to JSON, adding significant overhead to every update.

Fabric was designed to solve these issues by creating a more tightly integrated and synchronous rendering pipeline, with a significant portion of the logic moved into a shared C++ core.

## Visual: Fabric Render Pipeline

```mermaid
flowchart LR
  React[React Reconciler] -->|produces| ShadowTree[Fabric Shadow Tree (C++)]
  ShadowTree --> Layout[Yoga Layout]
  Layout --> Mounting[Mounting Layer]
  Mounting --> PlatformViews[Native Views\n(UIView / Android View)]
  React <-->|events| PlatformViews
  note right of ShadowTree: Immutable nodes, concurrent-safe
```

## The C++ Core of Fabric

At its heart, Fabric introduces a new data structure that lives entirely in C++: the **Shadow Tree**. This tree is the single source of truth for the state of the UI.

### Understanding the Shadow Tree Architecture

The Shadow Tree is not just a simple mirror of the React component tree. It's a sophisticated, immutable data structure designed for concurrent updates and efficient diffing:

```cpp
// Simplified conceptual view of the Shadow Tree
ShadowTree (Root)
├── ScrollViewShadowNode
│   ├── ViewShadowNode (Header)
│   │   └── TextShadowNode ("Welcome")
│   └── ViewShadowNode (Content)
│       ├── ImageShadowNode
│       └── TextShadowNode ("Description")
└── ViewShadowNode (Footer)
```

### 1. `ShadowNode`: The Immutable Building Block

The fundamental unit of the Shadow Tree is the `ShadowNode`. As defined in `packages/react-native/ReactCommon/react/renderer/core/ShadowNode.h` [1], it is the C++ representation of a React element.

```cpp
// Key aspects of ShadowNode design
class ShadowNode {
public:
  // Immutable design - all data is const after construction
  using Shared = std::shared_ptr<const ShadowNode>;
  
  // Core data (all immutable)
  const ShadowNodeFragment fragment_;
  const ShadowNodeFamily::Shared family_;
  const State::Shared state_;
  
  // Layout results (calculated by Yoga)
  LayoutMetrics layoutMetrics_;
  
  // Revision tracking for efficient updates
  int32_t revision_;
};
```

**Why Immutability Matters:**

```cpp
// Old Architecture - Mutable UI updates (problematic)
view.backgroundColor = "red";  // Direct mutation
view.frame = newFrame;         // Another mutation
// Result: Unpredictable state, race conditions

// Fabric - Immutable updates (safe & predictable)
auto oldNode = getShadowNode();
auto newNode = oldNode->clone({
  .props = std::make_shared<ViewProps>(*oldProps, [](auto& props) {
    props.backgroundColor = Color::Red();
  })
});
// Original node unchanged, new node created
```

**Real-World Example - Animated Color Change:**
```cpp
// Creating a new ShadowNode for a color animation
auto animateColor = [](const ShadowNode::Shared& node, float progress) {
  auto oldProps = std::static_pointer_cast<const ViewProps>(node->getProps());
  
  // Interpolate color based on progress
  auto newColor = Color::interpolate(
    Color::White(), 
    Color::Blue(), 
    progress
  );
  
  // Create new props with updated color
  auto newProps = std::make_shared<ViewProps>(*oldProps, [&](auto& props) {
    props.backgroundColor = newColor;
  });
  
  // Clone node with new props
  return node->clone({
    .props = newProps
  });
};
```

### Deep Dive: Props System

The Props system in Fabric is type-safe and efficient:

```cpp
// Base Props class
class Props {
public:
  // Revision for change detection
  int32_t revision;
  
  // Source of props (JS, native, or default)
  PropsSource source;
  
  // Raw props from JavaScript
  RawProps rawProps;
};

// Concrete example - ViewProps
class ViewProps : public Props {
public:
  // Typed properties
  Float opacity{1.0};
  Color backgroundColor{};
  Transform transform{};
  
  // Efficient parsing from JS
  ViewProps(const ViewProps& sourceProps, const RawProps& rawProps) {
    // Only parse changed properties
    if (rawProps.hasProperty("opacity")) {
      opacity = rawProps.getFloat("opacity");
    }
    if (rawProps.hasProperty("backgroundColor")) {
      backgroundColor = Color::parse(rawProps.getString("backgroundColor"));
    }
  }
};
```

### State Management in Fabric

Unlike props, state can be updated from both JavaScript and native:

```cpp
// Example: ScrollView state
class ScrollViewState {
public:
  // Content offset managed by native gesture
  Point contentOffset{0, 0};
  
  // Content size calculated during layout
  Size contentSize{0, 0};
  
  // Update from native (e.g., user scrolling)
  void updateContentOffset(Point newOffset) {
    contentOffset = newOffset;
    // Notify JavaScript of state change
    commitStateUpdate();
  }
};
```

### 2. `ComponentDescriptor`: The Shadow Node Factory

Defined in `packages/react-native/ReactCommon/react/renderer/core/ComponentDescriptor.h` [2], a `ComponentDescriptor` is a factory object responsible for creating and managing `ShadowNode` instances of a specific type.

```cpp
// Complete ComponentDescriptor interface
class ComponentDescriptor {
public:
  // Factory methods
  virtual std::shared_ptr<ShadowNode> createShadowNode(
      const ShadowNodeFragment& fragment,
      const ShadowNodeFamily::Shared& family) const = 0;

  virtual std::shared_ptr<ShadowNode> cloneShadowNode(
      const ShadowNode& sourceShadowNode,
      const ShadowNodeFragment& fragment) const = 0;
  
  // Props handling
  virtual Props::Shared cloneProps(
      const Props::Shared& props,
      const RawProps& rawProps) const = 0;
  
  // Event emitter creation
  virtual EventEmitter::Shared createEventEmitter(
      const ShadowNode::Shared& shadowNode,
      const EventDispatcher::Shared& eventDispatcher) const = 0;
};
```

**Real Implementation Example - ViewComponentDescriptor:**
```cpp
class ViewComponentDescriptor : public ComponentDescriptor {
public:
  ComponentHandle getComponentHandle() const override {
    return ViewComponentName; // "RCTView"
  }
  
  std::shared_ptr<ShadowNode> createShadowNode(
      const ShadowNodeFragment& fragment,
      const ShadowNodeFamily::Shared& family) const override {
    // Create props from raw JS values
    auto props = std::make_shared<ViewProps>(fragment.props);
    
    // Create the shadow node
    return std::make_shared<ViewShadowNode>(
      fragment,
      family,
      props
    );
  }
  
  // Specialized clone for performance
  std::shared_ptr<ShadowNode> cloneShadowNode(
      const ShadowNode& source,
      const ShadowNodeFragment& fragment) const override {
    // Efficient cloning with minimal allocations
    return std::make_shared<ViewShadowNode>(
      source,
      fragment
    );
  }
};
```

### 3. `ShadowTree`: Managing UI Evolution

As seen in `packages/react-native/ReactCommon/react/renderer/mounting/ShadowTree.h` [3], this class manages the entire lifecycle of the UI tree.

```cpp
class ShadowTree {
private:
  // Current committed tree
  ShadowTreeRevision::Shared currentRevision_;
  
  // Lock-free commit algorithm
  std::atomic<bool> commitLock_{false};
  
public:
  // Core commit operation
  bool tryCommit(
    const ShadowTreeCommitTransaction& transaction,
    const CommitOptions& options = {}) {
    
    // 1. Create new tree by applying transaction
    auto newRootNode = transaction(currentRevision_->rootShadowNode);
    
    // 2. Calculate layout with Yoga
    newRootNode->layoutTree(layoutContext);
    
    // 3. Create new revision
    auto newRevision = std::make_shared<ShadowTreeRevision>(
      newRootNode,
      currentRevision_->number + 1
    );
    
    // 4. Atomic commit
    if (commitLock_.exchange(true)) {
      return false; // Another commit in progress
    }
    
    currentRevision_ = newRevision;
    commitLock_ = false;
    
    // 5. Schedule mounting
    mountingCoordinator_->scheduleMount(newRevision);
    
    return true;
  }
};
```

**Commit Transaction Example:**
```cpp
// Updating a button's color when pressed
shadowTree.tryCommit([&](const RootShadowNode::Shared& root) {
  // Find the button node
  auto buttonNode = findNodeByTag(root, buttonTag);
  
  // Clone with new props
  auto newButtonNode = buttonNode->clone({
    .props = std::make_shared<ViewProps>(*buttonNode->getProps(), 
      [](auto& props) {
        props.backgroundColor = Color::Blue();
      })
  });
  
  // Clone parent nodes up to root
  return cloneTreeWithNewNode(root, buttonNode, newButtonNode);
});

## The Three Phases of Fabric Rendering

Fabric's rendering pipeline is a carefully orchestrated dance between JavaScript, C++, and platform-native code:

### Phase 1: The Render Phase (JavaScript → C++)

This phase begins in JavaScript when React detects a state or props change:

```javascript
// JavaScript: State update triggers re-render
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Text>{count}</Text>
      <Button onPress={() => setCount(count + 1)} title="Increment" />
    </View>
  );
}
```

**What happens under the hood:**

```cpp
// 1. React calls Fabric's renderer via JSI
void FabricUIManager::createNode(
    jsi::Runtime& rt,
    int tag,
    const jsi::String& componentName,
    int rootTag,
    const jsi::Object& props) {
  
  // 2. Look up ComponentDescriptor
  auto componentDescriptor = registry_->getComponentDescriptor(componentName);
  
  // 3. Parse props from JavaScript
  auto rawProps = RawProps(rt, props);
  
  // 4. Create ShadowNode
  auto shadowNode = componentDescriptor->createShadowNode({
    .tag = tag,
    .rootTag = rootTag,
    .props = componentDescriptor->cloneProps(nullptr, rawProps),
    .eventEmitter = componentDescriptor->createEventEmitter(tag)
  });
  
  // 5. Add to shadow tree (uncommitted)
  uncommittedTree_->appendChild(parentNode, shadowNode);
}
```

**Performance Optimization - Prop Parsing:**
```cpp
// Fabric only parses props that actually changed
class ViewProps {
  void fromRawProps(const RawProps& rawProps) {
    // Check revision to skip unchanged props
    if (rawProps.revision <= lastParsedRevision_) {
      return;
    }
    
    // Parse only modified properties
    for (const auto& [key, value] : rawProps.changedProps()) {
      if (key == "backgroundColor") {
        backgroundColor = parseColor(value);
      } else if (key == "opacity") {
        opacity = value.asDouble();
      }
      // ... other props
    }
    
    lastParsedRevision_ = rawProps.revision;
  }
};
```

### Phase 2: The Commit Phase (C++ Background Thread)

The commit phase is where Fabric's performance advantages really shine:

```cpp
void ShadowTree::commit(const CommitTransaction& transaction) {
  // 1. Apply the transaction to create new tree
  auto newRoot = transaction(currentRevision_->rootNode);
  
  // 2. Perform tree diffing
  auto mutations = calculateMutations(
    currentRevision_->rootNode,
    newRoot
  );
  
  // 3. Layout calculation with Yoga
  performLayout(newRoot);
  
  // 4. Create new revision
  auto newRevision = std::make_shared<ShadowTreeRevision>(
    newRoot,
    mutations,
    currentRevision_->number + 1
  );
  
  // 5. Atomic pointer swap
  currentRevision_ = newRevision;
  
  // 6. Schedule mount on UI thread
  mountingCoordinator_->scheduleMount(mutations);
}
```

**Tree Diffing Algorithm:**
```cpp
std::vector<Mutation> calculateMutations(
    const ShadowNode::Shared& oldRoot,
    const ShadowNode::Shared& newRoot) {
  
  std::vector<Mutation> mutations;
  
  // Recursive diff algorithm
  diffNodes(oldRoot, newRoot, mutations);
  
  return mutations;
}

void diffNodes(
    const ShadowNode::Shared& oldNode,
    const ShadowNode::Shared& newNode,
    std::vector<Mutation>& mutations) {
  
  // Same node, check for updates
  if (oldNode->getTag() == newNode->getTag()) {
    // Props changed?
    if (oldNode->getProps() != newNode->getProps()) {
      mutations.push_back({
        .type = Mutation::Type::Update,
        .tag = newNode->getTag(),
        .newProps = newNode->getProps()
      });
    }
    
    // Recursively diff children
    diffChildren(oldNode->getChildren(), newNode->getChildren(), mutations);
    
  } else {
    // Different nodes - replace
    mutations.push_back({
      .type = Mutation::Type::Remove,
      .tag = oldNode->getTag()
    });
    mutations.push_back({
      .type = Mutation::Type::Insert,
      .tag = newNode->getTag(),
      .parentTag = newNode->getParent()->getTag()
    });
  }
}
```

**Layout Calculation with Yoga:**
```cpp
void performLayout(const ShadowNode::Shared& root) {
  // Yoga configuration
  YGConfigRef yogaConfig = YGConfigNew();
  YGConfigSetPointScaleFactor(yogaConfig, screenScale_);
  
  // Convert shadow tree to Yoga tree
  auto yogaNode = createYogaTree(root);
  
  // Calculate layout
  YGNodeCalculateLayout(
    yogaNode,
    screenSize_.width,
    screenSize_.height,
    YGDirectionLTR
  );
  
  // Apply layout results back to shadow nodes
  applyLayoutResults(root, yogaNode);
  
  YGNodeFree(yogaNode);
  YGConfigFree(yogaConfig);
}
```

### Phase 3: The Mount Phase (C++ → Native Platform)

This is where abstract mutations become real UI changes:

```cpp
// iOS Implementation
void MountingCoordinator::mount(const MountingTransaction& transaction) {
  // Execute on main thread
  dispatch_async(dispatch_get_main_queue(), ^{
    for (const auto& mutation : transaction.getMutations()) {
      switch (mutation.type) {
        case Mutation::Type::Create:
          createView(mutation);
          break;
        case Mutation::Type::Update:
          updateView(mutation);
          break;
        case Mutation::Type::Delete:
          deleteView(mutation);
          break;
        case Mutation::Type::Insert:
          insertView(mutation);
          break;
      }
    }
  });
}

void updateView(const Mutation& mutation) {
  // Get the native view
  UIView* view = [viewRegistry viewForTag:mutation.tag];
  
  // Cast to component view protocol
  id<RCTComponentViewProtocol> componentView = (id<RCTComponentViewProtocol>)view;
  
  // Update props (type-safe, no serialization)
  [componentView updateProps:mutation.newProps oldProps:mutation.oldProps];
  
  // Update layout if needed
  if (mutation.layoutMetrics) {
    [componentView updateLayoutMetrics:*mutation.layoutMetrics
                     oldLayoutMetrics:mutation.oldLayoutMetrics];
  }
}
```

**Real Native View Update (iOS):**
```objc
// RCTViewComponentView.mm
- (void)updateProps:(const ViewProps &)props
           oldProps:(const ViewProps &)oldProps {
  // Direct C++ struct access - no parsing!
  if (props.backgroundColor != oldProps.backgroundColor) {
    self.backgroundColor = UIColorFromColor(props.backgroundColor);
  }
  
  if (props.opacity != oldProps.opacity) {
    self.alpha = props.opacity;
  }
  
  if (props.transform != oldProps.transform) {
    self.transform3D = CATransform3DFromTransform(props.transform);
  }
}


## React 18 Concurrent Features: Real-World Impact

Fabric's C++ core enables React 18's concurrent features, transforming how React Native applications handle complex UI updates:

### Automatic Batching in Practice

React 18's automatic batching dramatically reduces re-renders in real applications:

```jsx
// Real-world example: Shopping cart with multiple state updates
function ShoppingCart() {
  const [items, setItems] = useState([]);
  const [total, setTotal] = useState(0);
  const [discount, setDiscount] = useState(0);
  const [isLoading, setIsLoading] = useState(false);
  
  const addItem = async (product) => {
    // Old Architecture: 4 separate renders
    // New Architecture: 1 batched render
    
    setIsLoading(true);
    const updatedCart = await api.addToCart(product);
    
    // All these updates are batched together
    setItems(updatedCart.items);
    setTotal(updatedCart.total);
    setDiscount(updatedCart.discount);
    setIsLoading(false);
    
    // Fabric ensures only ONE render cycle occurs
  };
  
  return (
    <View>
      <Text>Total: ${total - discount}</Text>
      {items.map(item => <CartItem key={item.id} item={item} />)}
      {isLoading && <ActivityIndicator />}
    </View>
  );
}
```

**Performance Metrics:**
- **Old Architecture**: 4 renders × 16ms = 64ms total
- **New Architecture**: 1 render × 16ms = 16ms total
- **Result**: 75% reduction in render time

### Transitions for Heavy Computations

Real-world example of filtering a large dataset:

```jsx
function ProductCatalog() {
  const [searchQuery, setSearchQuery] = useState('');
  const [filteredProducts, setFilteredProducts] = useState(products);
  const [isPending, startTransition] = useTransition();
  
  // 10,000+ products to filter
  const products = useLargeProductDataset();
  
  const handleSearch = (query) => {
    // Update search input immediately (urgent)
    setSearchQuery(query);
    
    // Filter products in background (non-urgent)
    startTransition(() => {
      const filtered = products.filter(product => {
        // Complex filtering logic
        return (
          product.name.toLowerCase().includes(query.toLowerCase()) ||
          product.description.toLowerCase().includes(query.toLowerCase()) ||
          product.tags.some(tag => tag.includes(query))
        );
      });
      
      setFilteredProducts(filtered);
    });
  };
  
  return (
    <View>
      <TextInput
        value={searchQuery}
        onChangeText={handleSearch}
        placeholder="Search products..."
      />
      
      {isPending ? (
        <View style={styles.pendingOverlay}>
          <Text>Updating results...</Text>
          <ProductGrid products={filteredProducts} opacity={0.6} />
        </View>
      ) : (
        <ProductGrid products={filteredProducts} />
      )}
    </View>
  );
}
```

### Suspense with Real Data Fetching

Fabric enables sophisticated loading states with Suspense:

```jsx
// Resource creation with built-in caching
const userResource = createResource(userId => 
  fetch(`/api/users/${userId}`).then(r => r.json())
);

function UserDashboard({ userId }) {
  return (
    <View>
      {/* Multiple Suspense boundaries for granular loading */}
      <Suspense fallback={<HeaderSkeleton />}>
        <UserHeader userId={userId} />
      </Suspense>
      
      <Suspense fallback={<ContentSkeleton />}>
        <UserContent userId={userId} />
        
        {/* Nested suspense for dependent data */}
        <Suspense fallback={<FriendListSkeleton />}>
          <UserFriends userId={userId} />
        </Suspense>
      </Suspense>
    </View>
  );
}

function UserHeader({ userId }) {
  // This suspends until data is ready
  const user = userResource.read(userId);
  
  return (
    <View style={styles.header}>
      <Image source={{ uri: user.avatar }} />
      <Text style={styles.name}>{user.name}</Text>
    </View>
  );
}
```

### Concurrent Rendering in Action

Fabric's concurrent renderer enables time-slicing for smooth animations:

```jsx
function AnimatedList({ data }) {
  const scrollY = useSharedValue(0);
  const [visibleItems, setVisibleItems] = useState([]);
  
  // Heavy computation wrapped in transition
  const updateVisibleItems = useCallback((offset) => {
    startTransition(() => {
      // Calculate which items are visible
      const startIndex = Math.floor(offset / ITEM_HEIGHT);
      const endIndex = Math.ceil((offset + SCREEN_HEIGHT) / ITEM_HEIGHT);
      
      // Complex visibility calculations
      const newVisible = data.slice(startIndex, endIndex).map(item => ({
        ...item,
        opacity: calculateOpacity(item, offset),
        scale: calculateScale(item, offset),
        blur: calculateBlur(item, offset)
      }));
      
      setVisibleItems(newVisible);
    });
  }, [data]);
  
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
      
      // Update visible items without blocking scroll
      runOnJS(updateVisibleItems)(event.contentOffset.y);
    }
  });
  
  return (
    <Animated.ScrollView onScroll={scrollHandler}>
      {visibleItems.map(item => (
        <AnimatedItem key={item.id} item={item} scrollY={scrollY} />
      ))}
    </Animated.ScrollView>
  );
}
```

### Optimistic Updates with Concurrent Features

Combining transitions with optimistic updates:

```jsx
function TodoList() {
  const [todos, setTodos] = useState([]);
  const [optimisticTodos, setOptimisticTodos] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const addTodo = async (text) => {
    // Optimistic update (immediate)
    const newTodo = { id: Date.now(), text, pending: true };
    setOptimisticTodos([...todos, newTodo]);
    
    // Server update (can be interrupted)
    startTransition(async () => {
      try {
        const savedTodo = await api.createTodo(text);
        setTodos(current => [...current, savedTodo]);
        setOptimisticTodos([]); // Clear optimistic state
      } catch (error) {
        // Revert optimistic update
        setOptimisticTodos([]);
        showError("Failed to add todo");
      }
    });
  };
  
  const displayTodos = [...todos, ...optimisticTodos];
  
  return (
    <View>
      <TodoInput onAdd={addTodo} disabled={isPending} />
      {displayTodos.map(todo => (
        <TodoItem 
          key={todo.id} 
          todo={todo} 
          opacity={todo.pending ? 0.6 : 1}
        />
      ))}
    </View>
  );
}
```

---

**Citations:**

[1] `packages/react-native/ReactCommon/react/renderer/core/ShadowNode.h`
[2] `packages/react-native/ReactCommon/react/renderer/core/ComponentDescriptor.h`
[3] `packages/react-native/ReactCommon/react/renderer/mounting/ShadowTree.h`
[4] `packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.h`
