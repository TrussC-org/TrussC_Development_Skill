# TrussC Node System Reference

## Node (Base Class)

All scene objects inherit from Node. Nodes form a tree (scene graph).

### Lifecycle

```cpp
virtual void setup();      // Called once before first update/draw
virtual void update();     // Every frame
virtual void draw();       // Render this node
virtual void cleanup();    // On destroy
```

### Children

```cpp
addChild(Ptr child, bool keepGlobalPosition = false);
insertChild(size_t index, Ptr child, bool keepGlobalPosition = false);
removeChild(Ptr child);
removeAllChildren();
getParent();
getChildren();              // const vector<Ptr>&
getChildCount();
```

When `keepGlobalPosition=true`, child's screen position is preserved when reparenting.

**addChild to a new parent automatically removes from old parent.**

### State

```cpp
setActive(bool);            // false: update/draw/events all skipped
getActive();
setVisible(bool);           // false: only draw skipped
getVisible();
destroy();                  // Mark for deferred removal (see below)
isDead();                   // true if destroy() was called
enableEvents();             // Required for mouse events
disableEvents();
isMouseOver();              // O(1), updated each frame
```

Callbacks:
```cpp
virtual void onActiveChanged(bool active);
virtual void onVisibleChanged(bool visible);
virtual void onChildAdded(Ptr child);
virtual void onChildRemoved(Ptr child);
```

### Destroying Nodes

**Never call `removeChild()` during iteration** (e.g., inside update/draw loops) — it mutates the children vector and causes crashes.

Use `destroy()` instead:
```cpp
child->destroy();  // Marks as dead, actual removal happens later (after frame)
```

- `destroy()` sets a dead flag — `isDead()` returns true immediately
- The framework removes dead nodes **outside** the update/draw loop, so the vector stays stable
- `isDead()` is useful for skipping already-destroyed objects in your own logic
- `cleanup()` is called on the node when it's actually removed

**Pattern:**
```cpp
// In update — safe to mark for removal
for (auto& child : getChildren()) {
    if (shouldRemove(child)) {
        child->destroy();  // Safe — removed after this frame
    }
}

// Filter dead objects in your own containers
if (someNode && !someNode->isDead()) {
    // still valid
}
```

### Transform

```cpp
// Position
setPos(float x, float y, float z = 0);
setPos(Vec3 pos);
setX(float); setY(float); setZ(float);
getPos();  getX();  getY();  getZ();

// Rotation (radians)
setRot(float radians);      // 2D Z-axis rotation
getRot();
setEuler(float pitch, float yaw, float roll);  // 3D
getEuler();

// Scale
setScale(float uniform);
setScale(float sx, float sy, float sz = 1);
getScale();
```

### Coordinate Conversion

```cpp
// Vec3 overload (preferred)
Vec3 localToGlobal(const Vec3& local);
Vec3 globalToLocal(const Vec3& global);

// Out-param overload (legacy)
void globalToLocal(float gx, float gy, float& lx, float& ly);
void localToGlobal(float lx, float ly, float& gx, float& gy);

getMouseX();  getMouseY();       // Mouse in local space
getPMouseX(); getPMouseY();      // Previous frame
getLocalMatrix();                // Cached
getGlobalMatrix();               // Includes parents
```

### Mouse Events (override, return true to consume)

```cpp
virtual bool onMousePress(Vec2 local, int button);
virtual bool onMouseRelease(Vec2 local, int button);
virtual bool onMouseMove(Vec2 local);
virtual bool onMouseDrag(Vec2 local, int button);
virtual bool onMouseScroll(Vec2 local, Vec2 scroll);
virtual void onMouseEnter();
virtual void onMouseLeave();
```

### Keyboard Events (broadcast to all active nodes)

```cpp
virtual bool onKeyPress(int key);
virtual bool onKeyRelease(int key);
```

### Timers (protected)

```cpp
uint64_t callAfter(double seconds, function<void()> cb);
uint64_t callEvery(double interval, function<void()> cb);
void cancelTimer(uint64_t id);
void cancelAllTimers();
```

### Draw Hooks

```cpp
virtual void beginDraw();    // Before draw() + children
virtual void draw();         // Your rendering code
virtual void drawChildren(); // Default: draws all children
virtual void endDraw();      // After draw() + children (overlays)
```

Order: pushMatrix → beginDraw → **resetStyle** → draw → drawChildren → **resetStyle** → endDraw → popMatrix

**Style Isolation:** `resetStyle()` is called automatically before `draw()` and `endDraw()`. Each Node starts from default style (white color, fill enabled, no stroke). Parent and sibling styles never leak — always call `setColor()` etc. at the start of your `draw()`.

---

## RectNode (extends Node)

2D rectangle with width/height and hit testing.

### Size

```cpp
setSize(float w, float h);
setSize(float size);         // Square
setWidth(float); setHeight(float);
getWidth(); getHeight(); getSize();
setRect(float x, float y, float w, float h);  // Position + size

virtual void onSizeChanged();  // Override for reactions
```

### Bounds

```cpp
getLeft();   // 0
getRight();  // width
getTop();    // 0
getBottom(); // height
```

### Clipping

```cpp
setClipping(bool enabled);   // Scissor clip children to bounds
isClipping();
```

### Event Members

```cpp
Event<MouseEventArgs> mousePressed;
Event<MouseEventArgs> mouseReleased;
Event<MouseDragEventArgs> mouseDragged;
Event<ScrollEventArgs> mouseScrolled;
```

### Drawing Helpers

```cpp
drawRectFill();              // Filled rect
drawRectStroke();            // Outline rect
drawRectFillAndStroke();     // Both
```

---

## ScrollContainer

Scrollable container for content larger than viewport.

```cpp
setContent(Node::Ptr content);
getContent();

// Scroll position
getScrollX(); getScrollY(); getScroll();
setScrollX(float); setScrollY(float); setScroll(float x, float y);

// Bounds
getMaxScrollX(); getMaxScrollY();
updateScrollBounds();        // Call when content size changes

// Settings
setHorizontalScrollEnabled(bool);  // Default: false
setVerticalScrollEnabled(bool);    // Default: true
setScrollSpeed(float);             // Default: 20.0
```

Constructor: `enableEvents()` + `setClipping(true)` by default.

---

## ScrollBar

Visual indicator for ScrollContainer.

```cpp
ScrollBar(ScrollContainer* container, ScrollBar::Vertical or ::Horizontal);
void updateFromContainer();  // Call in update()

setBarColor(Color);
setBarWidth(float);
setMargin(float);
```

Draggable. Auto-hides when no scrolling needed.

---

## LayoutMod (auto-layout)

Arrange children vertically or horizontally.

```cpp
auto layout = node->addMod<LayoutMod>(LayoutDirection::Vertical, 10.0f);
layout->setCrossAxis(AxisMode::Fill);    // Children fill perpendicular axis
layout->setMainAxis(AxisMode::Content);  // Parent shrinks to fit children
layout->setPadding(5);
layout->updateLayout();                  // Call after adding/removing children
```

AxisMode: `None` (no resize), `Fill` (stretch to parent), `Content` (parent fits children)

---

## TweenMod (animation)

```cpp
auto tween = node->addMod<TweenMod>();
tween->moveTo(100, 200)
     .scaleTo(2.0f)
     .duration(1.0f)
     .delay(0.5f)
     .ease(EaseType::CubicInOut)
     .start();
tween->complete.listen([]() { /* done */ });
```

Supports: moveTo/By/From, scaleTo/By/From, rotateTo/By/From (radians), quaternion rotation.

---

## Event System

### Event<T> + EventListener (RAII)

**Always prefer Event/EventListener over raw `function<>` callbacks.** EventListener auto-disconnects on destruction, preventing dangling references.

```cpp
// === Emitter side: declare Event as public member ===
class MyWidget : public RectNode {
public:
    Event<CropAspect> clicked;       // Event with argument
    Event<void> doneEvent;           // Event without argument

protected:
    bool onMousePress(Vec2 pos, int button) override {
        clicked.notify(aspect_);     // Fire with arg (passed by reference)
        doneEvent.notify();          // Fire void event
        return true;
    }
};

// === Listener side: store EventListener as member ===
class MyPanel : public RectNode {
public:
    MyPanel() {
        widget_ = make_shared<MyWidget>();
        // listen() returns EventListener — MUST be stored
        clickListener_ = widget_->clicked.listen([this](CropAspect& a) {
            handleClick(a);
        });
    }

private:
    MyWidget::Ptr widget_;
    EventListener clickListener_;   // RAII: auto-disconnects on destruction
};
```

### Key Rules

1. **Store the EventListener** — if it goes out of scope, the listener disconnects immediately
2. **Event<T> callback receives T&** (reference) — `listen([](bool& val) { ... })`
3. **Event<void> callback receives nothing** — `listen([]() { ... })`
4. **notify(arg)** fires all listeners — copies entries before iteration (safe for listener removal during notify)
5. **Multiple listeners per event** — `listen()` can be called multiple times, all fire on notify
6. **Thread-safe** — Event uses recursive_mutex internally

### Chaining Events (bubbling up)

Widgets fire events → parent listens and re-fires its own events:

```cpp
// CropPanel listens to child OrientationToggle, re-fires as its own event
orientListener_ = orientToggle_->orientationChanged.listen([this](bool& landscape) {
    orientationChanged.notify(landscape);  // Bubble up to CropView
});
```

### Built-in Node Events

```cpp
// RectNode has built-in events (use alongside override methods):
Event<MouseEventArgs> mousePressed;
Event<MouseEventArgs> mouseReleased;
Event<MouseDragEventArgs> mouseDragged;
Event<ScrollEventArgs> mouseScrolled;
```

### Event Args

```cpp
struct MouseEventArgs { float x, y; int button; bool shift, ctrl, alt, super; };
struct MouseDragEventArgs { float x, y, deltaX, deltaY; int button; };
struct ScrollEventArgs { float scrollX, scrollY; };
struct KeyEventArgs { int key; bool isRepeat, shift, ctrl, alt, super; };
struct ResizeEventArgs { int width, height; };
```

---

## UI Widget Design Patterns

**Reference implementation: `src/crop/` (CropWidgets.h, CropPanel.h, CropView.h)**

### Rule: Every UI element is a RectNode child

Never draw UI elements (text, buttons, separators) directly in a parent's `draw()`. They won't scroll, won't receive events, and become unmaintainable.

```cpp
// BAD: drawing text in parent's draw()
void draw() override {
    drawBitmapString("Label", 10, 20);  // Won't scroll, no hit testing
}

// GOOD: text as a child RectNode
class TextLabel : public RectNode {
    void draw() override {
        font_->drawString(text, xPad, getHeight() / 2, Left, Center);
    }
};
content_->addChild(make_shared<TextLabel>("Label", &font_));
```

### Constructor vs setup() initialization

```
Constructor: create shared_ptr members, set sizes, wire Event listeners
setup():     addChild() calls only (requires shared_ptr to be complete)
```

Members used in `setSize()` / `update()` MUST be initialized in the constructor, because `setSize()` can be called before `setup()` runs.

### Panel Pattern (ScrollContainer + LayoutMod)

```cpp
class MyPanel : public RectNode {
    MyPanel() {
        scrollContainer_ = make_shared<PlainScrollContainer>();
        content_ = make_shared<RectNode>();
        scrollContainer_->setContent(content_);
        scrollBar_ = make_shared<ScrollBar>(scrollContainer_.get(), ScrollBar::Vertical);

        // LayoutMod: auto-stack children vertically
        contentLayout_ = content_->addMod<LayoutMod>(LayoutDirection::Vertical, 2);
        contentLayout_->setCrossAxis(AxisMode::Fill);    // children width = parent
        contentLayout_->setMainAxis(AxisMode::Content);  // parent height = sum of children
        contentLayout_->setPadding(9, 0, 12, 0);

        // Create widgets here...
    }

    void setup() override {
        addChild(scrollContainer_);
        content_->addChild(widget1_);
        content_->addChild(widget2_);
    }

    void setSize(float w, float h) override {
        RectNode::setSize(w, h);
        scrollContainer_->setRect(0, 0, w, h);
        content_->setWidth(w - 12);  // scrollbar space
        contentLayout_->updateLayout();  // Must call manually after size change
    }

    void update() override {
        scrollContainer_->updateScrollBounds();
        scrollBar_->updateFromContainer();
    }

    void draw() override {
        // Only non-scrolling elements (background, border)
        setColor(0.09f, 0.09f, 0.11f); fill(); drawRect(0, 0, getWidth(), getHeight());
    }
};
```

### LayoutMod notes

- `updateLayout()` is NOT automatic — call it in `setSize()` and after adding/removing children
- `addMod<LayoutMod>(...)` returns `LayoutMod*` (raw pointer) — Node owns the Mod, safe as long as Node lives
- `earlyUpdate()` exists but is empty — no auto-relayout on child size changes

### PlainScrollContainer

`ScrollContainer` draws a default background. Use `PlainScrollContainer` (defined in FolderTree.h) to suppress it:

```cpp
class PlainScrollContainer : public ScrollContainer {
public:
    void draw() override {}  // No background
};
```

### Font rendering

Always use `Font` over `drawBitmapString` for UI text:

```cpp
Font font_;
loadJapaneseFont(font_, 12);  // or font_.load(TC_FONT_SANS, 14);
font_.drawString(text, x, y, Left, Center);  // Alignment: Left/Center/Right, Top/Center/Bottom/Baseline
```

### Widget-to-parent communication

Use `Event<T>` for child → parent communication. Never call parent methods directly.

```
Widget (Event emitter)  →  Panel (listens, re-fires)  →  View (listens, acts)  →  App (listens)
```

Each layer stores `EventListener` members for RAII lifecycle management.
