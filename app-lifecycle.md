# TrussC App Lifecycle & Utilities

## App Class

App inherits from RectNode. Entry point:

```cpp
int main() {
    WindowSettings settings;
    settings.setSize(1280, 720).setTitle("My App");
    runApp<MyApp>(settings);
}

class MyApp : public App {
    void setup() override { }
    void update() override { }
    void draw() override { }
};
```

### Virtual Methods

```cpp
// Lifecycle
virtual void setup()
virtual void update()
virtual void draw()
virtual void cleanup()
virtual void exit()                    // Before cleanup, save settings here

// Keyboard
virtual void keyPressed(int key)
virtual void keyReleased(int key)

// Mouse (screen coordinates)
virtual void mousePressed(Vec2 pos, int button)
virtual void mouseReleased(Vec2 pos, int button)
virtual void mouseMoved(Vec2 pos)
virtual void mouseDragged(Vec2 pos, int button)
virtual void mouseScrolled(Vec2 delta)

// Window / IO
virtual void windowResized(int width, int height)
virtual void filesDropped(const vector<string>& files)
```

Events dispatch to both App virtual methods AND the node scene graph.

---

## Window Management

### WindowSettings

```cpp
WindowSettings settings;
settings.width = 1280;
settings.height = 720;
settings.title = "My App";
settings.highDpi = true;           // Retina support
settings.pixelPerfect = false;     // Logical coords vs framebuffer pixels
settings.sampleCount = 4;          // MSAA
settings.fullscreen = false;
```

### Runtime

```cpp
setWindowTitle(string)
setWindowSize(int w, int h)
setFullscreen(bool); toggleFullscreen(); isFullscreen()
getWindowWidth(); getWindowHeight(); getWindowSize() -> Vec2
getAspectRatio()
getDpiScale()                      // 2.0 on Retina
```

---

## Frame Timing

### FPS Control

```cpp
setFps(fps)                        // Sync update/draw
setIndependentFps(updateFps, drawFps)  // Independent rates

// Special constants
VSYNC   = -1.0f    // Match monitor refresh
EVENT_DRIVEN = 0.0f // Only on redraw()

redraw(count = 1)                  // Trigger draw frames
```

**Typical pattern for apps with UI:**
```cpp
setIndependentFps(VSYNC, 0);      // Update at vsync, draw only on redraw()
```

### Timing Queries

```cpp
getElapsedTime()                   // Seconds since start
getDeltaTime()                     // Seconds since last frame
getFrameRate()                     // FPS (10-frame average)
getFrameCount()                    // Total update() calls
getUpdateCount()
getDrawCount()
```

---

## Input State

### Mouse

```cpp
getGlobalMouseX(); getGlobalMouseY()
getGlobalPMouseX(); getGlobalPMouseY()  // Previous frame
getMousePos() -> Vec2
isMousePressed()
getMouseButton()                   // -1 if none
```

### Keyboard

```cpp
isKeyPressed(key)                  // Is key currently held?
```

Key constants:
```
KEY_SPACE, KEY_ESCAPE, KEY_ENTER, KEY_TAB, KEY_BACKSPACE, KEY_DELETE
KEY_LEFT, KEY_RIGHT, KEY_UP, KEY_DOWN
KEY_F1 ... KEY_F12
KEY_LEFT_SHIFT, KEY_RIGHT_SHIFT, KEY_LEFT_CONTROL, KEY_LEFT_ALT, KEY_LEFT_SUPER
MOUSE_BUTTON_LEFT (0), MOUSE_BUTTON_RIGHT (1), MOUSE_BUTTON_MIDDLE (2)
```

---

## Logging

```cpp
logNotice()  << "info message";
logWarning() << "warning message";
logError()   << "error message";

logNotice("TAG") << "tagged message";
```

**Never use cout** — stdout is reserved for MCP communication. Logs go to stderr.

LogLevel: Verbose, Notice, Warning, Error, Fatal, Silent

---

## Math

### Constants

```cpp
TAU          // 6.283... (full rotation, 2π) — USE THIS
HALF_TAU     // π
QUARTER_TAU  // π/2
PI           // deprecated, use HALF_TAU
```

### Vectors

```cpp
Vec2(x, y)
Vec3(x, y, z)
Vec4(x, y, z, w)

// Vec2/Vec3 methods
v.length(); v.lengthSquared()
v.normalized(); v.normalize()      // normalize() mutates
v.dot(other)
v.distance(other)
v.lerp(target, t)
v.angle()                         // Radians from +x (Vec2)
v.rotated(radians)                // Vec2
v.perpendicular()                 // Vec2

Vec2::fromAngle(radians, length)  // Static constructor
```

### Utility Functions

```cpp
lerp(a, b, t)
clamp(value, min, max)
map(value, inMin, inMax, outMin, outMax)
radians(degrees)
degrees(radians)
```

---

## Threading

### Thread Class

```cpp
class MyWorker : public Thread {
    void threadedFunction() override {
        while (isThreadRunning()) {
            // do work
            sleep(100);  // milliseconds
        }
    }
};

MyWorker worker;
worker.startThread();
worker.stopThread();       // Signal stop
worker.waitForThread();    // Block until done
worker.isThreadRunning();

Thread::isCurrentThreadTheMainThread();
Thread::sleep(milliseconds);
```

### Thread Safety Rules

1. **Never sleep inside a mutex lock** — deadlock
2. **Use lock_guard<mutex>** for shared data
3. **Pixels OK in background** — Image/Texture/Fbo are main-thread only
4. **Use ThreadChannel** for safe cross-thread message passing

---

## 3D Projection

```cpp
setupScreenOrtho()                 // 2D mode
setupScreenPerspective(fovDeg=45)  // 3D
setupScreenFov(fovDeg)             // 0=ortho, >0=perspective

setNearClip(float); setFarClip(float)

worldToScreen(Vec3) -> Vec3        // (screenX, screenY, depth)
screenToWorld(Vec2, worldZ=0) -> Vec3
```

---

## File Utilities

```cpp
// Data path (relative to exe + data root)
setDataPathRoot(path)
getDataPath(filename)
setDataPathToResources()           // macOS bundle Resources/

// Path parsing
getFileName(path)                  // "dir/test.txt" → "test.txt"
getBaseName(path)                  // "dir/test.txt" → "test"
getFileExtension(path)             // "dir/test.txt" → "txt"
getParentDirectory(path)

// Filesystem
fileExists(path)
directoryExists(path)
createDirectory(path)              // Creates parents too
listDirectory(path)                // vector<string>
joinPath(dir, file)
getAbsolutePath(path)
```

---

## String Utilities

```cpp
toString(value)
toString(value, precision)
toInt(str); toInt64(str)
toFloat(str); toDouble(str)
toBool(str)                        // "true"/"1"/"yes" → true
toHex(value); toBinary(value)
```
