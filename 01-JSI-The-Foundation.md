# Chapter 2: JSI - The Foundation

At the very heart of the New Architecture lies the **JavaScript Interface (JSI)**. It is the foundational layer that replaces the old, asynchronous Bridge and makes the performance and interoperability goals of the new architecture possible. The JSI is not a specific implementation but rather a lightweight, engine-agnostic C++ API that allows for direct, synchronous, and bidirectional communication between JavaScript and a native language.

Its definition can be found in the repository here: `packages/react-native/ReactCommon/jsi/jsi/jsi.h` [1]. Understanding the JSI is crucial because every other component of the New Architecture—Fabric, TurboModules, and even CodeGen—is built upon it.

## The Core Principles of JSI

The JSI was designed to solve the core problems of the Bridge: serialization overhead and asynchronous communication. It achieves this through a few key principles:

1.  **Shared Memory, Not Messages:** Instead of serializing data into JSON strings and sending them across a message queue, JSI allows the JavaScript and native worlds to hold direct references to each other's objects.
2.  **Synchronous by Default:** With direct references, JavaScript can invoke a method on a native object (and vice-versa) and get a result back immediately, within the same thread of execution.
3.  **C++ as the Lingua Franca:** The JSI itself is defined as a C++ abstract interface. This allows React Native to interact with any JavaScript engine (Hermes, V8, JavaScriptCore) simply by providing a C++ implementation that conforms to the JSI standard.

## The Key JSI C++ Components

A deep dive into `jsi.h` reveals the primary C++ classes and concepts that constitute the JSI API.

### 1. `jsi::Runtime` - The Execution Context

The `jsi::Runtime` class is the central nervous system of the JSI. It represents an instance of a JavaScript VM and is the primary entry point for all JSI operations. It is passed as the first argument to nearly every JSI function.

### 2. `jsi::HostObject` - Exposing Native Objects to JavaScript

The `jsi::HostObject` is the most powerful feature of the JSI. It allows you to expose a C++ object to the JavaScript world as if it were a regular JavaScript object.

To do this, a C++ class must inherit from `jsi::HostObject` and override key virtual methods:

```cpp
// From jsi.h
class JSI_EXPORT HostObject {
public:
  virtual ~HostObject();
  virtual Value get(Runtime&, const PropNameID& name);
  virtual void set(Runtime&, const PropNameID& name, const Value& value);
  virtual std::vector<PropNameID> getPropertyNames(Runtime& rt);
};
```

When JavaScript code attempts to access a property on a Host Object (e.g., `myNativeObject.someProperty`), the JSI intercepts this call and invokes the C++ `get` method.

### 3. `jsi::Function` and `HostFunctionType`

To expose a single native function to JavaScript, you use `jsi::Function::createFromHostFunction`. This factory method takes a `jsi::HostFunctionType`, which is a `std::function` with the following signature:

```cpp
// From jsi.h
using HostFunctionType = std::function<
    Value(Runtime& rt, const Value& thisVal, const Value* args, size_t count)>;
```

### 4. `jsi::Value` - The Universal Data Type

The `jsi::Value` is the currency of the JSI. It is a C++ tagged union that can represent any JavaScript value: `undefined`, `null`, `boolean`, `number`, `string`, `object`, etc. This is the key to eliminating serialization.

### 5. JSI Handles and Pointers

JSI types like `jsi::Object` and `jsi::String` are handles that refer to the actual data on the JS engine's heap. They have **move-only semantics**, a C++ pattern that prevents expensive, accidental copies.

### 6. `jsi::PropNameID` - Efficient Property Lookups

Instead of raw strings, the JSI uses `jsi::PropNameID` to represent property names. The runtime interns these names, allowing for much faster property lookups.

## Real-World Example: Vision Camera's Frame Processor

The power of `jsi::HostObject` is perfectly demonstrated by the `react-native-vision-camera` library (see Chapter 13). Its Frame Processor feature works by:

1.  Capturing a frame of video from the camera in native code.
2.  Wrapping the raw pixel buffer in a C++ `Frame` object.
3.  Wrapping that `Frame` object in a `FrameHostObject` which inherits from `jsi::HostObject`.
4.  Passing this `FrameHostObject` to a JavaScript worklet function.

When the developer's JavaScript code accesses `frame.width`, it is synchronously calling the `get()` method on the C++ `FrameHostObject`, which returns the width of the video frame with zero serialization overhead. This enables real-time video analysis directly in JavaScript, a feat impossible under the old architecture.

---

**Citations:**

[1] `packages/react-native/ReactCommon/jsi/jsi/jsi.h`
