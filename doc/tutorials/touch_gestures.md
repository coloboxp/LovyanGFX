# Touch Gesture System Guide

This guide explains how to implement a comprehensive touch gesture system in LovyanGFX.

## Gesture System Architecture

The gesture system consists of gesture detection, tracking, and event handling components:

```cpp
class TouchGestureSystem {
protected:
    struct TouchPoint {
        int16_t x, y;           // Current position
        int16_t start_x, start_y; // Starting position
        int16_t last_x, last_y;   // Previous position
        uint32_t start_time;      // Touch start time
        uint32_t last_time;       // Last update time
        bool active;              // Touch point active
    };
    
    static const int MAX_TOUCHES = 5;  // Maximum touch points
    TouchPoint points[MAX_TOUCHES];
    LGFX* display;
    
    // Gesture thresholds
    struct {
        uint32_t tap_duration = 200;      // Max tap duration (ms)
        uint32_t long_press_duration = 500; // Long press duration
        int16_t swipe_threshold = 50;     // Min swipe distance
        int16_t zoom_threshold = 10;      // Min zoom distance
        int16_t rotation_threshold = 5;   // Min rotation angle
    } config;
    
public:
    TouchGestureSystem(LGFX* disp) : display(disp) {
        reset();
    }
    
    void reset() {
        for (int i = 0; i < MAX_TOUCHES; i++) {
            points[i].active = false;
        }
    }
};
```

## Gesture Detection

### Basic Touch Tracking

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    void update() {
        int16_t x, y;
        if (display->getTouch(&x, &y)) {
            // New touch point
            if (!points[0].active) {
                points[0].active = true;
                points[0].start_x = points[0].x = x;
                points[0].start_y = points[0].y = y;
                points[0].start_time = millis();
                onTouchStart(0, x, y);
            } else {
                // Update existing touch
                points[0].last_x = points[0].x;
                points[0].last_y = points[0].y;
                points[0].x = x;
                points[0].y = y;
                points[0].last_time = millis();
                onTouchMove(0, x, y);
            }
        } else if (points[0].active) {
            // Touch ended
            onTouchEnd(0, points[0].x, points[0].y);
            points[0].active = false;
        }
    }
    
protected:
    virtual void onTouchStart(int id, int16_t x, int16_t y) {}
    virtual void onTouchMove(int id, int16_t x, int16_t y) {}
    virtual void onTouchEnd(int id, int16_t x, int16_t y) {}
};
```

### Multi-Touch Support

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    void updateMultiTouch() {
        static touch_point_t raw_points[MAX_TOUCHES];
        uint8_t count = display->getTouchRaw(raw_points, MAX_TOUCHES);
        
        // Update active points
        for (int i = 0; i < count; i++) {
            int16_t x = raw_points[i].x;
            int16_t y = raw_points[i].y;
            
            if (!points[i].active) {
                // New touch point
                points[i].active = true;
                points[i].start_x = points[i].x = x;
                points[i].start_y = points[i].y = y;
                points[i].start_time = millis();
                onTouchStart(i, x, y);
            } else {
                // Update existing touch
                points[i].last_x = points[i].x;
                points[i].last_y = points[i].y;
                points[i].x = x;
                points[i].y = y;
                points[i].last_time = millis();
                onTouchMove(i, x, y);
            }
        }
        
        // End inactive points
        for (int i = count; i < MAX_TOUCHES; i++) {
            if (points[i].active) {
                onTouchEnd(i, points[i].x, points[i].y);
                points[i].active = false;
            }
        }
    }
};
```

## Gesture Recognition

### Single-Touch Gestures

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    void detectSingleTouchGestures() {
        if (!points[0].active) return;
        
        uint32_t duration = millis() - points[0].start_time;
        int16_t dx = points[0].x - points[0].start_x;
        int16_t dy = points[0].y - points[0].start_y;
        float distance = sqrt(dx * dx + dy * dy);
        
        // Detect tap or long press
        if (distance < config.swipe_threshold) {
            if (duration >= config.long_press_duration) {
                onLongPress(points[0].x, points[0].y);
            }
            return;
        }
        
        // Detect swipe direction
        if (distance >= config.swipe_threshold) {
            float angle = atan2(dy, dx) * 180 / PI;
            
            if (abs(dx) > abs(dy)) {
                // Horizontal swipe
                onSwipe(dx > 0 ? SwipeDirection::Right : SwipeDirection::Left);
            } else {
                // Vertical swipe
                onSwipe(dy > 0 ? SwipeDirection::Down : SwipeDirection::Up);
            }
        }
    }
    
protected:
    enum class SwipeDirection {
        Left, Right, Up, Down
    };
    
    virtual void onTap(int16_t x, int16_t y) {}
    virtual void onLongPress(int16_t x, int16_t y) {}
    virtual void onSwipe(SwipeDirection direction) {}
};
```

### Multi-Touch Gestures

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    void detectMultiTouchGestures() {
        if (!points[0].active || !points[1].active) return;
        
        // Calculate current and previous distances
        float curr_dx = points[1].x - points[0].x;
        float curr_dy = points[1].y - points[0].y;
        float prev_dx = points[1].last_x - points[0].last_x;
        float prev_dy = points[1].last_y - points[0].last_y;
        
        float curr_dist = sqrt(curr_dx * curr_dx + curr_dy * curr_dy);
        float prev_dist = sqrt(prev_dx * prev_dx + prev_dy * prev_dy);
        
        // Detect pinch/zoom
        float scale_change = curr_dist / prev_dist;
        if (abs(scale_change - 1.0f) > config.zoom_threshold / 100.0f) {
            onZoom(scale_change);
        }
        
        // Detect rotation
        float curr_angle = atan2(curr_dy, curr_dx);
        float prev_angle = atan2(prev_dy, prev_dx);
        float rotation = (curr_angle - prev_angle) * 180 / PI;
        
        if (abs(rotation) > config.rotation_threshold) {
            onRotate(rotation);
        }
    }
    
protected:
    virtual void onZoom(float scale) {}
    virtual void onRotate(float angle) {}
};
```

## Gesture Event Handling

### Event System

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    struct GestureEvent {
        enum Type {
            Tap,
            LongPress,
            Swipe,
            Zoom,
            Rotate
        } type;
        
        union {
            struct { int16_t x, y; } point;
            SwipeDirection swipe;
            float scale;
            float angle;
        } data;
    };
    
    std::vector<std::function<void(const GestureEvent&)>> listeners;
    
public:
    void addEventListener(std::function<void(const GestureEvent&)> listener) {
        listeners.push_back(listener);
    }
    
protected:
    void dispatchEvent(const GestureEvent& event) {
        for (auto& listener : listeners) {
            listener(event);
        }
    }
    
    // Override virtual handlers
    void onTap(int16_t x, int16_t y) override {
        GestureEvent event;
        event.type = GestureEvent::Tap;
        event.data.point = {x, y};
        dispatchEvent(event);
    }
    
    // Similar implementations for other gesture events
};
```

## Usage Example

```cpp
class MyApplication {
    LGFX display;
    TouchGestureSystem gestures;
    
public:
    MyApplication() : gestures(&display) {
        // Initialize display
        display.init();
        
        // Add gesture event listener
        gestures.addEventListener([this](const GestureEvent& event) {
            handleGesture(event);
        });
    }
    
    void handleGesture(const GestureEvent& event) {
        switch (event.type) {
            case GestureEvent::Tap:
                Serial.printf("Tap at %d,%d\n", 
                    event.data.point.x, 
                    event.data.point.y);
                break;
                
            case GestureEvent::Swipe:
                switch (event.data.swipe) {
                    case SwipeDirection::Left:
                        previousPage();
                        break;
                    case SwipeDirection::Right:
                        nextPage();
                        break;
                }
                break;
                
            case GestureEvent::Zoom:
                updateScale(event.data.scale);
                break;
                
            case GestureEvent::Rotate:
                updateRotation(event.data.angle);
                break;
        }
    }
    
    void update() {
        gestures.update();  // Update gesture detection
        // Other application updates
    }
};
```

## Performance Optimization

```cpp
class TouchGestureSystem {
    // ... previous code ...
    
    struct GestureState {
        uint32_t last_update;
        uint32_t update_interval;
        bool gesture_active;
        
        void setUpdateInterval(uint32_t interval) {
            update_interval = interval;
        }
        
        bool shouldUpdate() {
            uint32_t now = millis();
            if (now - last_update >= update_interval) {
                last_update = now;
                return true;
            }
            return false;
        }
    } state;
    
public:
    void setUpdateInterval(uint32_t interval) {
        state.setUpdateInterval(interval);
    }
    
    void update() {
        if (!state.shouldUpdate()) return;
        
        // Regular update code
    }
};
```

These examples demonstrate how to implement a comprehensive touch gesture system in LovyanGFX. Adapt them according to your specific needs and requirements. 