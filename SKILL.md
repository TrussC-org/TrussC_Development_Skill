---
name: trussc-dev
description: TrussC framework knowledge — API patterns, node system, graphics, build workflow, common pitfalls. Use when writing TrussC C++ code.
---

# TrussC Development Guide

Comprehensive reference for writing TrussC applications. See topic files for detailed API.

## Quick Reference Files

- [node-system.md](node-system.md) — Node, RectNode, ScrollContainer, events, timers, mods
- [graphics.md](graphics.md) — Drawing, colors, style, Image/Pixels/Texture/Fbo
- [app-lifecycle.md](app-lifecycle.md) — App setup, window, input, logging, threading, math

## Build Workflow

### New Source File → cmake re-configure

When you create a new `.cpp` or `.h` file under `src/`, you MUST run:

```bash
cmake --preset macos
```

TrussC uses `file(GLOB_RECURSE CONFIGURE_DEPENDS ...)`. Ninja picks up new files automatically, but Xcode does not. Always run `cmake --preset macos` after adding/deleting/renaming source files.

### Build

```bash
cmake --build build-macos
```

### projectGenerator

If `CMakeLists.txt` or `CMakePresets.json` need regeneration (e.g., adding/removing addons):

```bash
/Users/toru/Nextcloud/Make/TrussC/projectGenerator/projectGenerator.app/Contents/MacOS/projectGenerator --update /path/to/project
```

## Essential Patterns

### Namespace & Includes

```cpp
#include <TrussC.h>          // Angle brackets, not quotes
using namespace std;
using namespace tc;
// Omit std:: and tc:: prefixes everywhere
```

### Units & Constants

- **Angles:** TAU (= 2π, full rotation). NOT PI.
- **Colors:** 0.0–1.0 float range (not 0–255)
- **Logging:** `logNotice()`, `logWarning()`, `logError()` — NOT cout (stdout reserved for MCP)

### Event-Driven Rendering (redraw)

```cpp
setIndependentFps(VSYNC, 0);  // update runs at vsync, draw only on redraw()
```

Call `redraw()` whenever display changes: key/mouse input, data updates, async load completion, window resize.

### Thread Safety

| Class | Main Thread | Background Thread |
|-------|:-----------:|:-----------------:|
| Pixels | OK | OK (CPU only) |
| Texture | OK | NO (GPU) |
| Image | OK | NO (GPU sync) |
| Fbo | OK | NO (GPU) |
| Font | OK | NO (GPU atlas) |

**Pattern:** Load `Pixels` in background thread → transfer to main thread → create Texture/Image.

### Node Hierarchy Basics

```cpp
auto child = make_shared<RectNode>();
child->setSize(100, 50);
child->setPos(10, 20);
parent->addChild(child);           // Automatic transform inheritance
child->enableEvents();             // Required for mouse events
child->setActive(false);           // Hides + disables update/draw/events
```

Draw order: `beginDraw()` → `draw()` → `drawChildren()` → `endDraw()`

Children draw in their parent's local coordinate space. No automatic clipping (use `setClipping(true)` on RectNode for scissor).

### Mouse Events

Override `onMousePress(Vec2 local, int button)` etc. Return `true` to consume (stop propagation).

Events are in **local coordinates** of the node. `enableEvents()` is required.

```cpp
bool onMousePress(Vec2 local, int button) override {
    if (button == 0) { /* left click */ return true; }
    return RectNode::onMousePress(local, button);  // propagate
}
```

### Timers

```cpp
uint64_t id = callAfter(0.5, [this]() { /* fires once after 0.5s */ });
uint64_t id2 = callEvery(1.0, [this]() { /* fires every 1s */ });
cancelTimer(id);
```

Protected Node methods. Store ID to cancel later.

## UI Widget Design Patterns

See [node-system.md](node-system.md) § "UI Widget Design Patterns" for details on:
- Every UI element as a RectNode child (labels, separators, buttons)
- Event<T>/EventListener RAII for widget communication (no raw `function<>` callbacks)
- PlainScrollContainer + ScrollBar + LayoutMod panel pattern
- Constructor/setup() separation (create in constructor, addChild in setup)
- Font rendering for all text

## Common Pitfalls

1. **Letter keys are UPPERCASE** → `'A'` not `'a'`. sokol uses key codes, not ASCII. `if (key == 'a')` will never match.
2. **sleep() inside mutex lock** → deadlock. Always sleep OUTSIDE lock scope.
3. **Image in background thread** → crash. Use Pixels for background, Image for main thread only.
4. **Texture update twice per frame** → second call silently skipped (sokol limitation).
5. **Forgot enableEvents()** → node won't receive mouse/key events.
6. **Forgot redraw()** → screen doesn't update in event-driven mode.
7. **setLineWidth()** → doesn't exist in sokol. drawLine is always 1px. Use StrokeMesh for thick lines.
8. **std::map vs tc::map** → name collision with `using namespace tc`. Use `std::map` explicitly if needed, or avoid `using namespace tc` in that file.
9. **Copy Pixels/Image/Texture** → deleted copy constructor. Use `std::move()` or `Pixels::clone()`.
10. **Node without make_shared** → `addChild()` fails. Always create nodes with `make_shared<>()`.
11. **getGlobalPos()** → returns `Vec3` (origin of the node in global space). Shorthand for `localToGlobal(Vec3(0,0,0))`.
12. **removeChild() during iteration** → crashes (vector mutation). Use `destroy()` for deferred removal. Check `isDead()` to skip destroyed nodes.
13. **addChild() in constructor** → crash. `weak_from_this()` fails because `shared_ptr` isn't complete yet. Always `addChild()` in `setup()` override.
14. **Raw `function<>` callbacks** → no auto-cleanup, dangling risk. Use `Event<T>` + `EventListener` (RAII auto-disconnect on destruction).
15. **Drawing UI elements in parent draw()** → won't scroll, no hit testing. Make every UI element a RectNode child.
16. **LayoutMod auto-relayout** → doesn't happen. Call `updateLayout()` manually in `setSize()` and after child changes.
