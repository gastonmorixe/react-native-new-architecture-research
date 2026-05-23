# Chapter 2: JSI - The Foundation

At the very heart of the New Architecture lies the **JavaScript Interface (JSI)**. It is the foundational layer that replaces the old, asynchronous Bridge and makes the performance and interoperability goals of the new architecture possible. The JSI is not a specific implementation but rather a lightweight, engine-agnostic C++ API that allows for direct, synchronous, and bidirectional communication between JavaScript and a native language.

Its definition can be found in the repository here: `packages/react-native/ReactCommon/jsi/jsi/jsi.h` [1]. Understanding the JSI is crucial because every other component of the New Architecture—Fabric, TurboModules, and even CodeGen—is built upon it.

**Current Status (2025):** JSI is now the standard communication layer in React Native 0.76+. It powers all new native modules and components, providing significant performance improvements over the legacy Bridge architecture.

## Visual: JSI Call Path and Data Flow

```mermaid
flowchart LR
    JS[JS Runtime\n(Hermes / V8 / JSC)] -->|calls| JSI[JSI C++ API]
    JSI --> Host[HostObject / Function]
    Host --> Cpp[C++ Impl]
    Cpp --> Platform[iOS / Android APIs]
    JS <-->|refs| JSI
    classDef core fill:#e6f4ff,stroke:#1f6feb,color:#0b2e13
    class JSI,Host,Cpp core
```

## The Core Principles of JSI

The JSI was designed to solve the core problems of the Bridge: serialization overhead and asynchronous communication. It achieves this through a few key principles:

### 1. Shared Memory, Not Messages

Instead of serializing data into JSON strings and sending them across a message queue, JSI allows the JavaScript and native worlds to hold direct references to each other's objects.

**Bridge Architecture (Old):**
```javascript
// JavaScript creates an object
const data = { users: [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }] };

// Bridge serializes to JSON string: '{"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]}'
// ~100 bytes sent across Bridge
// Native deserializes JSON back to NSDictionary/HashMap
```

**JSI Architecture (New):**
```cpp
// JavaScript and Native share the same memory reference
// No serialization, no copying, just a pointer
jsi::Object data = // ... reference to JS object
// Direct access to properties without any conversion
```

### 2. Synchronous by Default

With direct references, JavaScript can invoke a method on a native object (and vice-versa) and get a result back immediately, within the same thread of execution.

**Real-World Impact:**
```javascript
// Old Bridge - Forced Asynchronous Pattern
class LocationManager {
  async getCurrentLocation() {
    return new Promise((resolve) => {
      NativeModules.Location.getCurrentPosition((location) => {
        resolve(location); // ~50-100ms later
      });
    });
  }
}

// New JSI - Synchronous When Needed
class LocationManager {
  getCurrentLocationSync() {
    // Direct synchronous call - immediate result
    return NativeLocation.getCurrentPositionSync();
  }
  
  // Still supports async for long operations
  async getCurrentLocationAsync() {
    return NativeLocation.getCurrentPositionAsync();
  }
}
```

### 3. C++ as the Lingua Franca

The JSI itself is defined as a C++ abstract interface. This allows React Native to interact with any JavaScript engine (Hermes, V8, JavaScriptCore) simply by providing a C++ implementation that conforms to the JSI standard.

**Engine Agnostic Design:**
```cpp
// JSI works with any JavaScript engine
namespace facebook {
namespace jsi {
  // Abstract interface - no engine-specific code
  class Runtime {
  public:
    virtual Value evaluateJavaScript(
      const std::shared_ptr<const Buffer>& buffer,
      const std::string& sourceURL) = 0;
    // ... other virtual methods
  };
}
}

// Engine-specific implementations
class HermesRuntime : public jsi::Runtime { /* Hermes implementation */ };
class V8Runtime : public jsi::Runtime { /* V8 implementation */ };
class JSCRuntime : public jsi::Runtime { /* JavaScriptCore implementation */ };
```

## The Key JSI C++ Components

A deep dive into `jsi.h` reveals the primary C++ classes and concepts that constitute the JSI API.

### 1. `jsi::Runtime` - The Execution Context

The `jsi::Runtime` class is the central nervous system of the JSI. It represents an instance of a JavaScript VM and is the primary entry point for all JSI operations.

```cpp
// Key Runtime operations
class Runtime {
public:
  // Execute JavaScript code
  Value evaluateJavaScript(const std::shared_ptr<const Buffer>& buffer, 
                          const std::string& sourceURL);
  
  // Access global object
  Object global();
  
  // Create new values
  Value createValueFromJsonUtf8(const uint8_t* json, size_t length);
  
  // Clone values (deep copy)
  Value cloneValue(const Value& value);
  
  // Check value types
  bool isArray(const Object&) const;
  bool isFunction(const Object&) const;
  
  // ... many more operations
};
```

**Practical Example - Evaluating JavaScript:**
```cpp
void executeScript(jsi::Runtime& rt) {
  // Create JavaScript code
  const char* code = R"(
    function greet(name) {
      return `Hello, ${name}!`;
    }
    greet('React Native');
  )";
  
  // Evaluate the code
  auto result = rt.evaluateJavaScript(
    std::make_shared<jsi::StringBuffer>(code),
    "greeting.js"
  );
  
  // result now contains "Hello, React Native!"
  std::string greeting = result.getString(rt).utf8(rt);
}
```

### 2. `jsi::HostObject` - The Bridge Between Worlds

The `jsi::HostObject` is the most powerful feature of the JSI. It allows you to expose a C++ object to the JavaScript world as if it were a regular JavaScript object.

```cpp
// Complete HostObject interface
class JSI_EXPORT HostObject {
public:
  virtual ~HostObject();
  
  // Called when JS accesses a property: obj.prop
  virtual Value get(Runtime& rt, const PropNameID& name);
  
  // Called when JS sets a property: obj.prop = value
  virtual void set(Runtime& rt, const PropNameID& name, const Value& value);
  
  // Return all available properties
  virtual std::vector<PropNameID> getPropertyNames(Runtime& rt);
};
```

**Real-World Example - Native File System Access:**
```cpp
class FileSystemHostObject : public jsi::HostObject {
private:
  std::string basePath_;
  
public:
  FileSystemHostObject(const std::string& basePath) : basePath_(basePath) {}
  
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "readFile") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "readFile"), 1,
        [this](jsi::Runtime& rt, const jsi::Value&, const jsi::Value* args, size_t) {
          if (args[0].isString()) {
            std::string filename = args[0].getString(rt).utf8(rt);
            std::string content = readFileSync(basePath_ + "/" + filename);
            return jsi::String::createFromUtf8(rt, content);
          }
          throw jsi::JSError(rt, "Filename must be a string");
        });
    }
    
    if (propName == "writeFile") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "writeFile"), 2,
        [this](jsi::Runtime& rt, const jsi::Value&, const jsi::Value* args, size_t) {
          std::string filename = args[0].getString(rt).utf8(rt);
          std::string content = args[1].getString(rt).utf8(rt);
          writeFileSync(basePath_ + "/" + filename, content);
          return jsi::Value::undefined();
        });
    }
    
    if (propName == "listFiles") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "listFiles"), 0,
        [this](jsi::Runtime& rt, const jsi::Value&, const jsi::Value*, size_t) {
          auto files = listDirectory(basePath_);
          auto array = jsi::Array(rt, files.size());
          for (size_t i = 0; i < files.size(); i++) {
            array.setValueAtIndex(rt, i, jsi::String::createFromUtf8(rt, files[i]));
          }
          return array;
        });
    }
    
    return jsi::Value::undefined();
  }
  
  void set(jsi::Runtime& rt, const jsi::PropNameID& name, const jsi::Value& value) override {
    // Could implement property setters here
    throw jsi::JSError(rt, "FileSystem properties are read-only");
  }
};

// Usage in JavaScript:
// const files = fs.listFiles();
// const content = fs.readFile('data.json');
// fs.writeFile('output.txt', 'Hello World');
```

### 3. `jsi::Function` and `HostFunctionType`

To expose a single native function to JavaScript, you use `jsi::Function::createFromHostFunction`.

```cpp
// HostFunctionType signature
using HostFunctionType = std::function<
    Value(Runtime& rt, const Value& thisVal, const Value* args, size_t count)>;
```

**Practical Example - High-Performance Math Library:**
```cpp
void installMathModule(jsi::Runtime& rt) {
  auto mathObj = jsi::Object(rt);
  
  // Fast matrix multiplication
  auto matrixMultiply = jsi::Function::createFromHostFunction(rt,
    jsi::PropNameID::forAscii(rt, "matrixMultiply"), 2,
    [](jsi::Runtime& rt, const jsi::Value& thisVal, 
       const jsi::Value* args, size_t count) -> jsi::Value {
      
      if (count != 2) {
        throw jsi::JSError(rt, "matrixMultiply expects 2 arguments");
      }
      
      // Direct access to JavaScript arrays - no serialization!
      auto matrixA = args[0].asObject(rt).asArray(rt);
      auto matrixB = args[1].asObject(rt).asArray(rt);
      
      size_t rowsA = matrixA.length(rt);
      size_t colsB = matrixB.getValueAtIndex(rt, 0)
                             .asObject(rt).asArray(rt).length(rt);
      
      // Perform multiplication in C++ for maximum performance
      auto result = jsi::Array(rt, rowsA);
      
      // ... actual matrix multiplication logic ...
      
      return result;
    });
  
  // Fast Fourier Transform
  auto fft = jsi::Function::createFromHostFunction(rt,
    jsi::PropNameID::forAscii(rt, "fft"), 1,
    [](jsi::Runtime& rt, const jsi::Value& thisVal,
       const jsi::Value* args, size_t count) -> jsi::Value {
      
      auto signal = args[0].asObject(rt).asArray(rt);
      size_t n = signal.length(rt);
      
      // Perform FFT in C++ - 100x faster than JavaScript
      std::vector<std::complex<double>> data(n);
      for (size_t i = 0; i < n; i++) {
        data[i] = signal.getValueAtIndex(rt, i).asNumber();
      }
      
      performFFT(data); // Native FFT implementation
      
      // Return complex number array
      auto result = jsi::Array(rt, n);
      for (size_t i = 0; i < n; i++) {
        auto complexNum = jsi::Object(rt);
        complexNum.setProperty(rt, "real", data[i].real());
        complexNum.setProperty(rt, "imag", data[i].imag());
        result.setValueAtIndex(rt, i, complexNum);
      }
      
      return result;
    });
  
  mathObj.setProperty(rt, "matrixMultiply", matrixMultiply);
  mathObj.setProperty(rt, "fft", fft);
  
  rt.global().setProperty(rt, "NativeMath", mathObj);
}

// JavaScript usage:
// const result = NativeMath.matrixMultiply([[1,2],[3,4]], [[5,6],[7,8]]);
// const spectrum = NativeMath.fft([1,2,3,4,5,6,7,8]);
```

### 4. `jsi::Value` - The Universal Data Type

The `jsi::Value` is the currency of the JSI. It is a C++ tagged union that can represent any JavaScript value.

```cpp
// Value type checking and conversion
void processValue(jsi::Runtime& rt, const jsi::Value& value) {
  if (value.isUndefined()) {
    // Handle undefined
  } else if (value.isNull()) {
    // Handle null
  } else if (value.isBool()) {
    bool b = value.getBool();
  } else if (value.isNumber()) {
    double n = value.getNumber();
  } else if (value.isString()) {
    std::string s = value.getString(rt).utf8(rt);
  } else if (value.isObject()) {
    auto obj = value.getObject(rt);
    
    if (obj.isArray(rt)) {
      auto arr = obj.getArray(rt);
      size_t length = arr.length(rt);
      
      // Iterate array without serialization
      for (size_t i = 0; i < length; i++) {
        auto element = arr.getValueAtIndex(rt, i);
        // Process element...
      }
    } else if (obj.isFunction(rt)) {
      auto func = obj.getFunction(rt);
      // Can call JavaScript functions from C++!
      auto result = func.call(rt, jsi::Value(42));
    }
  }
}
```

**Creating Complex Values:**
```cpp
// Create a complex object structure
jsi::Value createUserObject(jsi::Runtime& rt, const User& user) {
  auto userObj = jsi::Object(rt);
  
  // Basic properties
  userObj.setProperty(rt, "id", jsi::Value(user.id));
  userObj.setProperty(rt, "name", jsi::String::createFromUtf8(rt, user.name));
  userObj.setProperty(rt, "age", jsi::Value(user.age));
  
  // Nested object
  auto address = jsi::Object(rt);
  address.setProperty(rt, "street", jsi::String::createFromUtf8(rt, user.street));
  address.setProperty(rt, "city", jsi::String::createFromUtf8(rt, user.city));
  userObj.setProperty(rt, "address", address);
  
  // Array of tags
  auto tags = jsi::Array(rt, user.tags.size());
  for (size_t i = 0; i < user.tags.size(); i++) {
    tags.setValueAtIndex(rt, i, jsi::String::createFromUtf8(rt, user.tags[i]));
  }
  userObj.setProperty(rt, "tags", tags);
  
  // Method
  auto greet = jsi::Function::createFromHostFunction(rt,
    jsi::PropNameID::forAscii(rt, "greet"), 0,
    [name = user.name](jsi::Runtime& rt, const jsi::Value&, 
                       const jsi::Value*, size_t) -> jsi::Value {
      return jsi::String::createFromUtf8(rt, "Hello, I'm " + name);
    });
  userObj.setProperty(rt, "greet", greet);
  
  return userObj;
}
```

### 5. JSI Handles and Move Semantics

JSI types are handles with **move-only semantics** to prevent expensive copies:

```cpp
// ❌ Won't compile - no copy constructor
jsi::Object obj1 = jsi::Object(rt);
jsi::Object obj2 = obj1; // Error!

// ✅ Move semantics
jsi::Object obj1 = jsi::Object(rt);
jsi::Object obj2 = std::move(obj1); // obj1 is now invalid

// ✅ Pass by reference for reading
void readObject(jsi::Runtime& rt, const jsi::Object& obj) {
  auto value = obj.getProperty(rt, "key");
}

// ✅ Return by move
jsi::Object createObject(jsi::Runtime& rt) {
  auto obj = jsi::Object(rt);
  obj.setProperty(rt, "created", true);
  return obj; // Implicit move
}

// Working with collections
std::vector<jsi::Value> collectValues(jsi::Runtime& rt) {
  std::vector<jsi::Value> values;
  
  // Must use move to add to vector
  values.push_back(jsi::Value(42));
  values.push_back(jsi::String::createFromUtf8(rt, "hello"));
  values.push_back(jsi::Object(rt));
  
  return values; // Vector is moved
}
```

### 6. `jsi::PropNameID` - Optimized Property Access

Property names are interned for performance:

```cpp
// Efficient property access patterns
class OptimizedHostObject : public jsi::HostObject {
private:
  // Pre-intern frequently used property names
  struct PropertyNames {
    jsi::PropNameID width;
    jsi::PropNameID height;
    jsi::PropNameID data;
    jsi::PropNameID transform;
    
    PropertyNames(jsi::Runtime& rt) :
      width(jsi::PropNameID::forAscii(rt, "width")),
      height(jsi::PropNameID::forAscii(rt, "height")),
      data(jsi::PropNameID::forAscii(rt, "data")),
      transform(jsi::PropNameID::forAscii(rt, "transform")) {}
  };
  
  std::unique_ptr<PropertyNames> propNames_;
  
public:
  OptimizedHostObject(jsi::Runtime& rt) {
    propNames_ = std::make_unique<PropertyNames>(rt);
  }
  
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    // Fast comparison using interned IDs
    if (name.compare(rt, propNames_->width) == 0) {
      return jsi::Value(1920);
    } else if (name.compare(rt, propNames_->height) == 0) {
      return jsi::Value(1080);
    } else if (name.compare(rt, propNames_->data) == 0) {
      // Return large data without serialization
      return getLargeDataArray(rt);
    }
    
    return jsi::Value::undefined();
  }
};
```

## Real-World Example: Vision Camera's Frame Processor

The power of `jsi::HostObject` is perfectly demonstrated by the `react-native-vision-camera` library (see Chapter 13). Its Frame Processor feature works by:

1.  Capturing a frame of video from the camera in native code.
2.  Wrapping the raw pixel buffer in a C++ `Frame` object.
3.  Wrapping that `Frame` object in a `FrameHostObject` which inherits from `jsi::HostObject`.
4.  Passing this `FrameHostObject` to a JavaScript worklet function.

When the developer's JavaScript code accesses `frame.width`, it is synchronously calling the `get()` method on the C++ `FrameHostObject`, which returns the width of the video frame with zero serialization overhead. This enables real-time video analysis directly in JavaScript, a feat impossible under the old architecture.

**Performance Impact:** Vision Camera can process ~30 MB frame buffers at 60 FPS (approximately 2 GB of data per second) with minimal overhead thanks to JSI's direct memory access. This would be impossible with the legacy Bridge's serialization approach.

## Advanced JSI Patterns

### 1. Lazy Property Evaluation

Compute expensive properties only when accessed:

```cpp
class ImageHostObject : public jsi::HostObject {
private:
  std::string imagePath_;
  mutable std::optional<ImageMetadata> cachedMetadata_;
  mutable std::mutex metadataMutex_;
  
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "metadata") {
      // Lazy load and cache metadata
      std::lock_guard<std::mutex> lock(metadataMutex_);
      if (!cachedMetadata_.has_value()) {
        cachedMetadata_ = loadImageMetadata(imagePath_);
      }
      
      auto meta = jsi::Object(rt);
      meta.setProperty(rt, "width", cachedMetadata_->width);
      meta.setProperty(rt, "height", cachedMetadata_->height);
      meta.setProperty(rt, "format", jsi::String::createFromUtf8(rt, cachedMetadata_->format));
      return meta;
    }
    
    if (propName == "pixels") {
      // Return a proxy object that loads pixels on demand
      return createPixelAccessor(rt, imagePath_);
    }
    
    return jsi::Value::undefined();
  }
};
```

### 2. Streaming Data with JSI

Handle large datasets without loading everything into memory:

```cpp
class StreamHostObject : public jsi::HostObject {
private:
  std::unique_ptr<DataStream> stream_;
  
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "read") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "read"), 1,
        [this](jsi::Runtime& rt, const jsi::Value&, 
               const jsi::Value* args, size_t) -> jsi::Value {
          size_t bytes = args[0].asNumber();
          auto data = stream_->read(bytes);
          
          if (data.empty()) {
            return jsi::Value::null();
          }
          
          // Return typed array for efficient binary data
          auto arrayBuffer = jsi::ArrayBuffer(rt, data.size());
          memcpy(arrayBuffer.data(rt), data.data(), data.size());
          
          auto uint8Array = jsi::Object::createFromHostObject(
            rt, std::make_shared<TypedArrayHostObject>(arrayBuffer));
          
          return uint8Array;
        });
    }
    
    if (propName == "position") {
      return jsi::Value(static_cast<double>(stream_->position()));
    }
    
    return jsi::Value::undefined();
  }
};
```

### 3. Bidirectional Event Systems

Create event emitters that work seamlessly between JS and Native:

```cpp
class EventEmitterHostObject : public jsi::HostObject {
private:
  std::unordered_map<std::string, std::vector<jsi::Function>> listeners_;
  std::mutex listenersMutex_;
  
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "on") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "on"), 2,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          std::string event = args[0].getString(rt).utf8(rt);
          auto callback = args[1].getObject(rt).getFunction(rt);
          
          std::lock_guard<std::mutex> lock(listenersMutex_);
          listeners_[event].push_back(std::move(callback));
          
          return jsi::Value::undefined();
        });
    }
    
    if (propName == "emit") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "emit"), -1, // Variable args
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t count) -> jsi::Value {
          if (count < 1) return jsi::Value::undefined();
          
          std::string event = args[0].getString(rt).utf8(rt);
          
          std::lock_guard<std::mutex> lock(listenersMutex_);
          auto it = listeners_.find(event);
          if (it != listeners_.end()) {
            for (auto& listener : it->second) {
              // Call each listener with the remaining arguments
              listener.call(rt, args + 1, count - 1);
            }
          }
          
          return jsi::Value::undefined();
        });
    }
    
    return jsi::Value::undefined();
  }
  
  // Call from native code
  void emitFromNative(jsi::Runtime& rt, const std::string& event, 
                      std::vector<jsi::Value> args) {
    std::lock_guard<std::mutex> lock(listenersMutex_);
    auto it = listeners_.find(event);
    if (it != listeners_.end()) {
      for (auto& listener : it->second) {
        listener.call(rt, args.data(), args.size());
      }
    }
  }
};
```

### 4. Memory-Mapped Files with JSI

Direct access to large files without loading into memory:

```cpp
class MappedFileHostObject : public jsi::HostObject {
private:
  std::unique_ptr<MemoryMappedFile> file_;
  
public:
  MappedFileHostObject(const std::string& path) {
    file_ = std::make_unique<MemoryMappedFile>(path);
  }
  
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "size") {
      return jsi::Value(static_cast<double>(file_->size()));
    }
    
    if (propName == "getUint8") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "getUint8"), 1,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          size_t offset = args[0].asNumber();
          if (offset >= file_->size()) {
            throw jsi::JSError(rt, "Offset out of bounds");
          }
          return jsi::Value(file_->data()[offset]);
        });
    }
    
    if (propName == "slice") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "slice"), 2,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          size_t start = args[0].asNumber();
          size_t end = args[1].asNumber();
          
          if (start >= file_->size() || end > file_->size() || start > end) {
            throw jsi::JSError(rt, "Invalid slice range");
          }
          
          // Create ArrayBuffer view without copying
          auto buffer = jsi::ArrayBuffer(rt, file_->data() + start, end - start);
          return buffer;
        });
    }
    
    return jsi::Value::undefined();
  }
};
```

### 5. Thread-Safe Native Modules

Handle concurrent access from multiple threads:

```cpp
class ThreadSafeCounterHostObject : public jsi::HostObject {
private:
  std::atomic<int64_t> counter_{0};
  mutable std::shared_mutex callbacksMutex_;
  std::vector<jsi::Function> changeCallbacks_;
  
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "value") {
      return jsi::Value(static_cast<double>(counter_.load()));
    }
    
    if (propName == "increment") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "increment"), 0,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value*, size_t) -> jsi::Value {
          int64_t newValue = counter_.fetch_add(1) + 1;
          notifyListeners(rt, newValue);
          return jsi::Value(static_cast<double>(newValue));
        });
    }
    
    if (propName == "onChange") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "onChange"), 1,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          auto callback = args[0].getObject(rt).getFunction(rt);
          
          std::unique_lock<std::shared_mutex> lock(callbacksMutex_);
          changeCallbacks_.push_back(std::move(callback));
          
          return jsi::Value::undefined();
        });
    }
    
    return jsi::Value::undefined();
  }
  
private:
  void notifyListeners(jsi::Runtime& rt, int64_t newValue) {
    std::shared_lock<std::shared_mutex> lock(callbacksMutex_);
    auto value = jsi::Value(static_cast<double>(newValue));
    for (auto& callback : changeCallbacks_) {
      callback.call(rt, value);
    }
  }
};
```

### 6. Performance Profiling with JSI

Create native performance monitoring tools:

```cpp
class PerformanceMonitor : public jsi::HostObject {
private:
  struct Timing {
    std::chrono::high_resolution_clock::time_point start;
    std::chrono::high_resolution_clock::time_point end;
    std::string name;
  };
  
  std::vector<Timing> timings_;
  std::unordered_map<std::string, std::chrono::high_resolution_clock::time_point> activeMarks_;
  
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    auto propName = name.utf8(rt);
    
    if (propName == "mark") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "mark"), 1,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          std::string markName = args[0].getString(rt).utf8(rt);
          activeMarks_[markName] = std::chrono::high_resolution_clock::now();
          return jsi::Value::undefined();
        });
    }
    
    if (propName == "measure") {
      return jsi::Function::createFromHostFunction(rt,
        jsi::PropNameID::forAscii(rt, "measure"), 2,
        [this](jsi::Runtime& rt, const jsi::Value&,
               const jsi::Value* args, size_t) -> jsi::Value {
          std::string name = args[0].getString(rt).utf8(rt);
          std::string startMark = args[1].getString(rt).utf8(rt);
          
          auto it = activeMarks_.find(startMark);
          if (it != activeMarks_.end()) {
            Timing timing;
            timing.name = name;
            timing.start = it->second;
            timing.end = std::chrono::high_resolution_clock::now();
            timings_.push_back(timing);
            
            auto duration = std::chrono::duration_cast<std::chrono::microseconds>
                           (timing.end - timing.start).count();
            
            return jsi::Value(duration / 1000.0); // Return milliseconds
          }
          
          return jsi::Value::undefined();
        });
    }
    
    return jsi::Value::undefined();
  }
};
```

---

**Citations:**

[1] `packages/react-native/ReactCommon/jsi/jsi/jsi.h`
