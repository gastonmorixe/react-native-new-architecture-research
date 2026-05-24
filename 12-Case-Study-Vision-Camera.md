---
title: "Case Study - react-native-vision-camera"
chapter: "12"
created_at: "2025-09-22T15:30:19-04:00"
updated_at: "2026-05-23T19:59:32-0400"
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
tags: [react-native, new-architecture, case-study, vision-camera, jsi, host-object, frame-processor, worklets, nitro, third-party]
audit:
  status: "verified"
  baseline_branch: "audit/baseline-2026-05-23"
  verified_branch: "audit/verified-2026-05-23"
  report: "_verification/chapters/12-vision-camera/report.md"
taillog:
  - "2025-09-22T15:30:19-04:00 | Initial draft (RN 0.81.4 era)"
  - "2026-05-23T16:13:51-0400 | Source-grounded audit pass against RN 0.86 (commit b32a6c9e9db) and Vision Camera v4.7.2 / v5.0.x. See _verification/chapters/12-vision-camera/report.md."
  - "2026-05-23T19:59:32-0400 | Recheck pass against VC clone HEAD 5a07d7ae (v5.0.10+7). Repo restructured to packages/ monorepo since v1 audit; v4 paths still resolve via `git show v4.7.2:...`. Fixed three v1 footnote bugs (worklets-plugin docs path, Android FrameHostObject.cpp:22 line number, TurboModule TODO count 7-not-4). Added V5 PR commit pin (30a8b5de) and v5 Frame.nitro.ts citation. See _verification/recheck/vc/report.md."
---

# Chapter 13: Case Study - `react-native-vision-camera`

To understand how the New Architecture meets real-world code, we will analyze `react-native-vision-camera`. This popular library provides a high-performance camera component and has been a benchmark for advanced JSI usage long before the New Architecture umbrella term existed. It is also a useful counter-example: a library can be New Architecture *compatible* without being implemented with Fabric or TurboModule primitives at all.

This case study tracks two snapshots:

-   **v4.7.2** (2025-09-02), the last release of the v4 line, contemporary with the original drafting of this book. The deep-dive below is grounded in this version.[^vc-v472]
-   **v5.0.0** (2026-04-16), a full rewrite onto Nitro Modules.[^vc-v500] Chapter 13 covers Nitro in depth, so here we only note what changed and why.

## What "New Architecture support" actually means in v4

The v4 line is the version everyone migrating to Fabric and Bridgeless had to make peace with for most of 2025. It is worth being precise about how it works, because the line between "supports the New Architecture" and "is implemented as a New Architecture module" is exactly the line that confuses most consumers.

-   **View Layer (legacy Paper view, Fabric-compatible via interop).** The `<Camera>` component is exported with `requireNativeComponent('CameraView')`, the legacy native component API.[^vc-nativeview] On iOS, `CameraViewManager` extends `RCTViewManager` and props are wired with `RCT_EXPORT_VIEW_PROPERTY` macros.[^vc-iosviewmanager] On Android, `CameraViewManager` extends `ViewGroupManager<CameraView>` and uses `@ReactProp` setters.[^vc-androidviewmanager] None of this is generated from a CodeGen spec. When the host app runs the New Architecture, the **Paper-on-Fabric interop layer** wraps these legacy managers so the view still renders inside the Fabric tree.[^rn-interop]
-   **Module Layer (legacy bridge module, Fabric-compatible via interop).** The imperative APIs (`takePhoto`, `startRecording`, camera permissions) are exposed by `CameraViewModule`, which extends `ReactContextBaseJavaModule` on Android with `@ReactMethod` annotations.[^vc-androidmodule] The iOS Swift class is registered through `RCT_EXTERN_REMAP_MODULE(CameraView, CameraViewManager, RCTViewManager)`.[^vc-iosmodulebridge] The JS side reaches the module via `NativeModules.CameraView`, not a CodeGen-generated spec.[^vc-jsmodule] The source even contains explicit TODOs waiting for TurboModules to land before the API can be cleaned up.[^vc-todo] These too are routed through the **TurboModule interop layer** when the host app is on Bridgeless.
-   **Frame Processor (true JSI, custom worklet runtime).** This is the only piece that is genuinely native to the JSI world. A JSI binding installs `global.VisionCameraProxy` and a Camera Frame is exposed to JS as a `jsi::HostObject`. The worklet runtime is a separate JS runtime running on a dedicated camera thread, not the React JS thread.

So the right one-liner is: in v4, Vision Camera *runs under* Fabric and Bridgeless because RN's interop layer wraps legacy modules and view managers. It is not itself Fabric or TurboModule code.

## Advanced JSI: The Frame Processor

The most distinctive feature of Vision Camera is its **Frame Processor**: a JavaScript function called for every frame of video, on a dedicated camera thread, with zero serialization. This is the use case that pushes JSI hard, and the reason the library matters as a case study even when the rest of it is legacy.

Here is how it works end to end.

**1. The JavaScript Worklet**

A developer provides a Frame Processor through the `useFrameProcessor` hook.[^vc-useFrameProcessor] The provided function must be a **`'worklet'`**.

```typescript
const frameProcessor = useFrameProcessor((frame) => {
  'worklet'
  // This code runs on a dedicated camera thread
  const faces = scanFaces(frame) // e.g., using a face detection library
  console.log(`Detected ${faces.length} faces.`)
}, [])
```

The `'worklet'` directive is picked up by the **`react-native-worklets-core/plugin`** Babel plugin, which Vision Camera's docs require you to add to `babel.config.js`.[^vc-workletsplugin] That plugin rewrites the function so it can be serialized and re-installed in a separate JSI runtime. Vision Camera does not use the Reanimated Babel plugin for this. Reanimated and `react-native-worklets-core` both implement the worklet concept but are distinct packages with their own runtimes. The two-runtime model is what lets the camera thread call user JS without going through the React JS thread at all.

(V5 switches the default worklets runtime to `react-native-worklets` from Software Mansion.[^vc-v5worklets] The user-facing `'worklet'` directive is unchanged.)

**2. The `Frame` Host Object**

The `frame` object passed to the worklet is not a plain JavaScript object. It is a **`jsi::HostObject`**. The C++ header on iOS confirms this:[^vc-frameHostObjectIos]

```cpp
// From Vision Camera's iOS FrameHostObject.h
class JSI_EXPORT FrameHostObject : public jsi::HostObject, public std::enable_shared_from_this<FrameHostObject> {
public:
  explicit FrameHostObject(Frame* frame) : _frame(frame), _baseClass(nullptr) {}

public:
  jsi::Value get(jsi::Runtime&, const jsi::PropNameID& name) override;
  std::vector<jsi::PropNameID> getPropertyNames(jsi::Runtime& rt) override;
  // ...
private:
  Frame* _frame;
  std::unique_ptr<jsi::Object> _baseClass;
};
```

The base contract here is RN's own. In `packages/react-native/ReactCommon/jsi/jsi/jsi.h`, `class HostObject` declares `virtual Value get(Runtime&, const PropNameID& name)`, and that is what Vision Camera overrides.[^rn-jsiHostObject]

The Android header has the same JSI base class but a different wrapper, because the underlying frame data is a Java object reached through fbjni:[^vc-frameHostObjectAndroid]

```cpp
// From Vision Camera's Android FrameHostObject.h
class JSI_EXPORT FrameHostObject : public jsi::HostObject, public std::enable_shared_from_this<FrameHostObject> {
public:
  explicit FrameHostObject(const jni::alias_ref<JFrame::javaobject>& frame);
  ~FrameHostObject();

public:
  jsi::Value get(jsi::Runtime&, const jsi::PropNameID& name) override;
  std::vector<jsi::PropNameID> getPropertyNames(jsi::Runtime& rt) override;
  // ...
private:
  jni::global_ref<JFrame> _frame;
  std::unique_ptr<jsi::Object> _baseClass;
};
```

On iOS the wrapped type is an Objective-C `Frame` around a `CMSampleBuffer`. On Android it is a `jni::global_ref` to a Java `JFrame` that owns the `android.media.Image` and any `HardwareBuffer`. Either way, the JavaScript side only ever sees a `HostObject` proxy.

**3. The Synchronous, Off-Thread Loop**

The complete flow:

1.  The native camera hardware captures a frame of video.
2.  Vision Camera's native code creates a `Frame` (iOS Obj-C class or Android Java class) holding the buffer.
3.  It then creates a `std::shared_ptr<FrameHostObject>` that wraps the `Frame`.
4.  On the dedicated camera queue (on iOS, `CameraQueues.videoQueue`, a `userInteractive`-QoS `DispatchQueue`),[^vc-videoQueue] the `AVCaptureVideoDataOutputSampleBufferDelegate` callback fires and invokes the user's worklet, passing the `FrameHostObject` as an argument.[^vc-videoFrameCallback]
5.  When the JavaScript code accesses a property like `frame.width`, JSI synchronously calls `get()` on the C++ `FrameHostObject`, which reads the width from the underlying `Frame` and returns it as a `jsi::Value`.

The whole pipeline happens for every frame, with no JSON serialization and no involvement from the main React JS thread. That direct, synchronous JSI access is what makes real-time per-frame work viable.

## What v5 changed (and why this case study still matters)

V5 (April 2026) is a full rewrite onto **Nitro Modules**, the library author's own framework that competes with TurboModules.[^vc-v500] Roughly 3,000 lines of hand-written JSI/C++ went away, and the public surface moved to Nitro-generated `HybridObject` bindings. The Frame Processor pipeline is now plain Swift and Kotlin sitting behind Nitro specs, and the `FrameHostObject` class no longer exists in the v5 source tree.[^vc-v5frame] The repo also restructured into a monorepo with worklets split into a separate npm package, `react-native-vision-camera-worklets`, that peer-depends on `react-native-worklets` (Software Mansion).[^vc-v5worklets-pkg]

For this book, the v4 case study is still the more instructive one. It shows how a mature library can ship a New-Arch-compatible release without rewriting its module and view layers, while still exploiting JSI for the one piece (`Frame`) where the performance budget actually demands it. Chapter 13 picks up where this leaves off and covers Nitro itself.

## Conclusion

`react-native-vision-camera` demonstrates two things at once. First, that "supports the New Architecture" can mean nothing more than "runs under the interop layer", and a working production library can ship that way for years. Second, that JSI is the part of the New Architecture you actually feel as a user. A `jsi::HostObject` exposing a camera frame to a worklet, on a dedicated thread, is the kind of capability you cannot build on top of the legacy bridge no matter how hard you try. Tracking these patterns is a practical way to understand how production codebases bridge the gap between the legacy architecture and the new one.

---

**Citations:**

[^vc-v472]: Vision Camera package metadata at the v4.7.2 tag: `tmp/react-native-vision-camera/package/package.json` (`"version": "4.7.2"`). v4.7.2 was tagged on 2025-09-02 (commit `82148d10`). The repo was restructured into a monorepo at v5.0.0 (PR #3735, commit `30a8b5de`, 2026-04-16),[^vc-v5pr] which deleted the `package/` directory and moved the library into `packages/react-native-vision-camera/`. All v4 paths cited below resolve under `git show v4.7.2:<path>` even though they no longer exist at the clone's current `main` HEAD.

[^vc-v500]: V5 release notes: <https://github.com/mrousavy/react-native-vision-camera/releases/tag/v5.0.0> ("Fully rewritten to Nitro Modules"; "~3,000 lines of hand-written JSI/C++ deleted"). V5 docs: <https://visioncamera.margelo.com>. The "~3,000" figure checks out: summing pure-deletion `.cpp/.h/.hpp/.mm` files in the V5 PR diff (`git show 30a8b5de --stat`) gives 2,970 lines.

[^vc-v5pr]: V5 PR (`feat: V5. (#3735)`): commit `30a8b5de43dd21ecbb5500d1b01903de3ab42f36`, dated 2026-04-16. The tag `v5.0.0` is one commit later (`ef61e1b7`, `chore: release 5.0.0`).

[^vc-nativeview]: `tmp/react-native-vision-camera/package/src/NativeCameraView.ts:64` (`requireNativeComponent<NativeCameraViewProps>('CameraView')`). There is no `codegenNativeComponent` spec anywhere under `package/src/`.

[^vc-iosviewmanager]: `tmp/react-native-vision-camera/package/ios/React/CameraViewManager.swift:14` (`final class CameraViewManager: RCTViewManager`). Props are listed in `ios/React/CameraViewManager.m` via `RCT_EXPORT_VIEW_PROPERTY` macros.

[^vc-androidviewmanager]: `tmp/react-native-vision-camera/package/android/src/main/java/com/mrousavy/camera/react/CameraViewManager.kt:19` (`class CameraViewManager : ViewGroupManager<CameraView>()`). Props are declared with `@ReactProp`.

[^rn-interop]: RN provides automatic interop wrappers so that legacy view managers and bridge modules keep working when the host app enables Fabric and Bridgeless. The view-side wrapper lives under `packages/react-native/ReactCommon/react/renderer/components/legacyviewmanagerinterop/` (see `UnstableLegacyViewManagerAutomaticComponentDescriptor.{h,cpp}`); the module-side wrapper is `packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTInteropTurboModule.{h,mm}`.

[^vc-androidmodule]: `tmp/react-native-vision-camera/package/android/src/main/java/com/mrousavy/camera/react/CameraViewModule.kt:39` (`class CameraViewModule(reactContext: ReactApplicationContext) : ReactContextBaseJavaModule(reactContext)`). Methods are annotated `@ReactMethod`, the legacy bridge contract.

[^vc-iosmodulebridge]: `tmp/react-native-vision-camera/package/ios/React/CameraViewManager.m:14` (`RCT_EXTERN_REMAP_MODULE(CameraView, CameraViewManager, RCTViewManager)`). The Swift class is registered as a legacy bridge module/view-manager hybrid, not a TurboModule.

[^vc-jsmodule]: `tmp/react-native-vision-camera/package/src/NativeCameraModule.ts:8` (`export const CameraModule = NativeModules.CameraView`). `NativeModules` is the legacy bridge accessor, not `TurboModuleRegistry.get`.

[^vc-todo]: For example: `tmp/react-native-vision-camera/package/android/src/main/java/com/mrousavy/camera/react/CameraViewModule.kt:117` (`// TODO: ... Hopefully TurboModules allows that`) and `package/ios/React/CameraViewManager.swift:43` (`// Wait for TurboModules?`). Seven such TODO/"wait for TurboModules" comments exist across v4.7.2's `package/` tree (`grep -niE 'TODO.*TurboModules?|Wait for TurboModules|Hopefully TurboModules'`): three in Kotlin (`CameraView.kt:36`, `CameraViewManager.kt:158`, `CameraViewManager.kt:166`, plus the Module one above), two in Swift (`CameraView.swift:16`, `CameraViewManager.swift:43`), and one in TSX (`Camera.tsx:229`). Two further notes in `src/index.ts:34` and `src/frame-processors/VisionCameraProxy.ts:91` reference "CxxTurboModule" as the eventual replacement.

[^vc-useFrameProcessor]: `tmp/react-native-vision-camera/package/src/hooks/useFrameProcessor.ts:42` (`export function useFrameProcessor(frameProcessor: (frame: Frame) => void, dependencies: DependencyList): ReadonlyFrameProcessor`).

[^vc-workletsplugin]: Official Vision Camera docs: `tmp/react-native-vision-camera/docs/docs/guides/FRAME_PROCESSORS.mdx:48-63` ("Frame Processors require `react-native-worklets-core` 1.0.0 or higher... add the plugin to your `babel.config.js`: `['react-native-worklets-core/plugin']`"). The docs site sits at the repo root, not under `package/`. At runtime, `tmp/react-native-vision-camera/package/src/frame-processors/VisionCameraProxy.ts:60` lazy-loads the package (`require('react-native-worklets-core')`). The v4 example app (`tmp/react-native-vision-camera/example/babel.config.js`) also registers `react-native-reanimated/plugin` for unrelated Reanimated animations; that plugin is not what compiles VC's frame-processor worklets.

[^vc-v5worklets]: V5 release notes (same link as [^vc-v500]): "Default Worklets runtime switched ... to `react-native-worklets` (Software Mansion) instead of `react-native-worklets-core`. You can now mutate Reanimated SharedValues directly from a Frame Processor."

[^vc-frameHostObjectIos]: `tmp/react-native-vision-camera/package/ios/FrameProcessors/FrameHostObject.h:19-35`.

[^rn-jsiHostObject]: `packages/react-native/ReactCommon/jsi/jsi/jsi.h:222` (class declaration), line 239 (`virtual Value get(Runtime&, const PropNameID& name)`).

[^vc-frameHostObjectAndroid]: `tmp/react-native-vision-camera/package/android/src/main/cpp/frameprocessors/FrameHostObject.h:20-37`. Constructor body is on `FrameHostObject.cpp:22` (`FrameHostObject::FrameHostObject(... frame) : _frame(make_global(frame)), _baseClass(nullptr) {}`).

[^vc-videoQueue]: `tmp/react-native-vision-camera/package/ios/Core/CameraQueues.swift:21-25` (declares `videoQueue`); `ios/Core/CameraSession+Configuration.swift:111` (`videoOutput.setSampleBufferDelegate(self, queue: CameraQueues.videoQueue)`).

[^vc-videoFrameCallback]: `tmp/react-native-vision-camera/package/ios/Core/CameraSession.swift:268-294` (`captureOutput(_:didOutput:from:)` → `onVideoFrame(...)` → `delegate.onFrame(...)`).

[^vc-v5frame]: V5 `Frame` is a Nitro `HybridObject` spec, not a `jsi::HostObject`: `tmp/react-native-vision-camera/packages/react-native-vision-camera/src/specs/instances/Frame.nitro.ts:73-99` (`interface Frame extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> { readonly width: number; readonly height: number; readonly pixelFormat: PixelFormat; ... getPlanes(): FramePlane[]; getPixelBuffer(): ArrayBuffer; getNativeBuffer(): NativeBuffer; ... }`). The Nitro tool generates the JSI and JNI bindings from this spec; no hand-written `HostObject` survives. `git grep -l 'FrameHostObject' HEAD` returns empty at the v5 working tree.

[^vc-v5worklets-pkg]: `tmp/react-native-vision-camera/packages/react-native-vision-camera-worklets/package.json` (peerDeps include `"react-native-worklets": "*"`, not `react-native-worklets-core`). The v5 example babel config is `tmp/react-native-vision-camera/apps/simple-camera/babel.config.js:3` (`plugins: ['react-native-worklets/plugin']`). The main `react-native-vision-camera@5.x` package's peer deps are `react-native-nitro-modules` and `react-native-nitro-image`; worklets is opt-in via the sibling package.
