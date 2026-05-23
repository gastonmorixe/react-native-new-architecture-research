# Chapter 13: Case Study - `react-native-vision-camera`

To understand how the New Architecture is applied in a complex, real-world library, we will analyze `react-native-vision-camera`. This popular library provides a high-performance camera component and is widely regarded as a benchmark for modern React Native development. It perfectly illustrates how Fabric, TurboModules, and advanced JSI concepts are used together.

## Standard New Architecture Components

Vision Camera has evolved significantly and now provides comprehensive New Architecture support (2025):[^1]

-   **View Layer (Fabric Component):** The `<Camera>` component is now implemented as a proper Fabric component with full New Architecture support. It uses CodeGen-generated specs and provides optimal performance with the new renderer.

-   **Module Layer (TurboModule):** The imperative APIs such as `takePhoto`, `startRecording`, and camera permissions are now implemented as TurboModules, providing synchronous access and better performance. The module leverages JSI for direct communication between JavaScript and native code.

-   **Frame Processor (Advanced JSI):** The frame processor system represents one of the most sophisticated uses of JSI in the React Native ecosystem, enabling real-time video processing directly in JavaScript.

## Advanced JSI: The Frame Processor

The most innovative feature of Vision Camera is its **Frame Processor**. This feature allows a developer to write a JavaScript function that can synchronously operate on every single frame of video, in real-time. This enables use cases like facial recognition, QR code scanning, and applying live filters directly in JS. This would be impossible under the old architecture, and it serves as a masterclass in advanced JSI usage.

Here is how it works, from end to end:

**1. The JavaScript Worklet**

A developer provides a Frame Processor using the `useFrameProcessor` hook. Crucially, the provided function must be a **`'worklet'`**.

```typescript
const frameProcessor = useFrameProcessor((frame) => {
  'worklet'
  // This code runs on a dedicated camera thread
  const faces = scanFaces(frame) // e.g., using a face detection library
  console.log(`Detected ${faces.length} faces.`)
}, [])
```

The `'worklet'` directive is a signal to the `react-native-reanimated` Babel plugin, which transforms the function so it can be executed on a different thread from the main React JS thread.

**2. The `Frame` Host Object**

The `frame` object passed to the worklet is not a plain JavaScript object. It is a **`jsi::HostObject`**. The C++ header file for this object, found in the library's source at `package/android/src/main/cpp/frameprocessors/FrameHostObject.h`, confirms this:[^4]

```cpp
// From Vision Camera's FrameHostObject.h
class JSI_EXPORT FrameHostObject : public jsi::HostObject, public std::enable_shared_from_this<FrameHostObject> {
public:
  explicit FrameHostObject(Frame* frame) : _frame(frame) {}

public:
  jsi::Value get(jsi::Runtime&, const jsi::PropNameID& name) override;
  // ...
private:
  Frame* _frame;
};
```

This `FrameHostObject` is a C++ class that wraps a pointer to the actual `Frame` data (which contains the raw pixel buffer from the camera).

**3. The Synchronous, Off-Thread Loop**

Here is the complete flow:

1.  The native camera hardware captures a frame of video.
2.  Vision Camera's native C++ code creates a `Frame` object containing this data.
3.  It then creates a `std::shared_ptr<FrameHostObject>` that wraps the `Frame`.
4.  On a dedicated camera thread (not the UI or JS thread), the native code invokes the user's JavaScript worklet, passing the `FrameHostObject` as an argument.
5.  When the JavaScript code accesses a property like `frame.width`, the JSI synchronously calls the `get()` method on the C++ `FrameHostObject`, which retrieves the width from the underlying `Frame` data and returns it as a `jsi::Value`.

This entire process happens for every single frame, with minimal overhead, no serialization, and no involvement from the main React JS thread. It is this direct, synchronous communication, enabled by the JSI, that allows for real-time video processing in JavaScript.

## Conclusion

`react-native-vision-camera` demonstrates how a mature library can progressively adopt pieces of the New Architecture. Its UI and imperative modules still use legacy entry points while remaining compatible with Fabric builds, and its Frame Processor system pushes the JSI to its full potential with `HostObject`s to enable high-performance, real-time features that were previously unthinkable. Tracking these patterns is a practical way to understand how production codebases bridge the gap between the legacy architecture and the new one.

---

**Citations:**

[^1]: `package/ios/React/CameraViewManager.swift`
[^2]: `package/src/NativeCameraView.ts`
[^3]: `package/android/src/main/java/com/mrousavy/camera/react/CameraViewModule.kt`
[^4]: `package/android/src/main/cpp/frameprocessors/FrameHostObject.h`
