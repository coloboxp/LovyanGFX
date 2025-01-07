# Touch Gesture System Guide

This guide explains how to implement a comprehensive touch gesture system in LovyanGFX, including gesture detection, tracking, and event handling.

## Gesture System Architecture

The gesture system consists of several components that work together to detect and handle various touch gestures:

```cpp
class TouchGestureSystem {
protected:
    // Structure to track individual touch points
    struct TouchPoint {
        int16_t x, y;           // Current position
        int16_t start_x, start_y; // Starting position for gesture
        int16_t last_x, last_y;   // Previous position for velocity
        uint32_t start_time;      // Touch start timestamp
        uint32_t last_time;       // Last update timestamp
        bool active;              // Whether point is being tracked
        
        // Calculate distance moved from start
        float getDistance() const {
            // Calculate Euclidean distance from start point using Pythagorean theorem
            // distance = √((x₂-x₁)² + (y₂-y₁)²)
            float dx = x - start_x;  // Change in x
            float dy = y - start_y;  // Change in y
            return sqrt(dx * dx + dy * dy);  // Hypotenuse length
        }
        
        // Calculate velocity in pixels per second
        float getVelocity() const {
            // Convert milliseconds to seconds for standard units
            float dt = (last_time - start_time) / 1000.0f;
            if (dt == 0) return 0;  // Prevent division by zero
            
            // Velocity = distance / time (pixels per second)
            return getDistance() / dt;
        }
    };
    
    static const int MAX_TOUCHES = 5;  // Maximum simultaneous touch points
    TouchPoint points[MAX_TOUCHES];    // Array of touch points
    LGFX* display;                     // Display reference
    
    // Gesture detection thresholds
    struct {
        uint32_t tap_duration = 200;      // 200ms maximum for tap
        uint32_t long_press_duration = 500; // 500ms for long press
        int16_t swipe_threshold = 50;     // 50px minimum swipe
        int16_t zoom_threshold = 10;      // 10% scale change
        int16_t rotation_threshold = 5;   // 5 degrees minimum
    } config;
    
public:
    TouchGestureSystem(LGFX* disp) : display(disp) {
        reset();
    }
    
    // Reset all touch tracking
    void reset() {
        for (int i = 0; i < MAX_TOUCHES; i++) {
            points[i].active = false;
        }
    }
    
    // Main update function - called every frame
    void update() {
        int16_t x, y;
        if (display->getTouch(&x, &y)) {
            // New touch point detected
            if (!points[0].active) {
                // Initialize new touch point
                points[0].active = true;
                points[0].start_x = points[0].x = x;
                points[0].start_y = points[0].y = y;
                points[0].start_time = millis();
                onTouchStart(0, x, y);
            } else {
                // Update existing touch
                points[0].last_x = points[0].x;
                points[0].last_y = points[0].y;
                points[0].last_time = millis();
                points[0].x = x;
                points[0].y = y;
                
                // Check for gestures
                detectGestures();
            }
        } else if (points[0].active) {
            // Touch released - check for tap gestures
            uint32_t duration = millis() - points[0].start_time;
            float distance = points[0].getDistance();
            
            if (distance < config.swipe_threshold) {
                if (duration < config.tap_duration) {
                    onTap(points[0].x, points[0].y);
                } else if (duration >= config.long_press_duration) {
                    onLongPress(points[0].x, points[0].y);
                }
            }
            
            points[0].active = false;
            onTouchEnd(0, points[0].x, points[0].y);
        }
    }
    
protected:
    // Detect various gestures based on touch movement
    void detectGestures() {
        TouchPoint& p = points[0];
        float distance = p.getDistance();
        float velocity = p.getVelocity();
        
        // Calculate movement vector components
        float dx = p.x - p.start_x;  // Horizontal displacement
        float dy = p.y - p.start_y;  // Vertical displacement
        
        // Detect swipe gestures when:
        // 1. Movement distance exceeds threshold
        // 2. Movement speed is above 0.5 pixels per second
        if (distance >= config.swipe_threshold && velocity > 0.5f) {
            SwipeDirection direction;
            // Compare absolute values to determine primary axis of movement
            if (abs(dx) > abs(dy)) {
                // Horizontal movement dominates - check direction
                direction = dx > 0 ? SwipeDirection::Right : SwipeDirection::Left;
            } else {
                // Vertical movement dominates - check direction
                direction = dy > 0 ? SwipeDirection::Down : SwipeDirection::Up;
            }
            onSwipe(direction, velocity);
        }
        
        // For multi-touch gestures (if supported)
        if (points[1].active) {
            detectMultiTouchGestures();
        }
    }
    
    // Handle multi-touch gestures like pinch and rotate
    void detectMultiTouchGestures() {
        TouchPoint& p1 = points[0];
        TouchPoint& p2 = points[1];
        
        // Calculate distances between touch points:
        // - curr_dist: current distance between points
        // - start_dist: distance at gesture start
        float curr_dist = getDistance(p1.x, p1.y, p2.x, p2.y);
        float start_dist = getDistance(p1.start_x, p1.start_y, 
                                     p2.start_x, p2.start_y);
        
        // Calculate scale factor as ratio of distances
        // scale > 1.0: pinch out (zoom in)
        // scale < 1.0: pinch in (zoom out)
        float scale = curr_dist / start_dist;
        
        // Detect significant pinch gestures
        // Convert threshold from pixels to scale factor percentage
        if (abs(scale - 1.0f) > config.zoom_threshold / 100.0f) {
            onPinch(scale);
        }
        
        // Calculate rotation angle:
        // 1. Get angle between points at current position
        // 2. Get angle between points at start position
        // 3. Subtract to get relative rotation
        float current_angle = getAngle(p1.x - p2.x, p1.y - p2.y);
        float start_angle = getAngle(p1.start_x - p2.start_x, 
                                    p1.start_y - p2.start_y);
        float rotation = current_angle - start_angle;
        
        // Detect significant rotation
        // threshold typically 5-10 degrees
        if (abs(rotation) > config.rotation_threshold) {
            onRotate(rotation);
        }
    }
    
    // Utility functions with detailed explanations
    float getDistance(int16_t x1, int16_t y1, int16_t x2, int16_t y2) {
        // Calculate Euclidean distance between two points
        // Uses Pythagorean theorem: d = √((x₂-x₁)² + (y₂-y₁)²)
        float dx = x2 - x1;  // Change in x
        float dy = y2 - y1;  // Change in y
        return sqrt(dx * dx + dy * dy);  // Hypotenuse length
    }
    
    float getAngle(float x, float y) {
        // Calculate angle in degrees from x-axis to point (x,y)
        // atan2 returns angle in radians (-π to π)
        // Convert to degrees by multiplying by 180/π
        return atan2(y, x) * 180.0f / PI;
    }
    
    // Virtual event handlers - override in derived classes
    virtual void onTouchStart(int id, int16_t x, int16_t y) {}
    virtual void onTouchEnd(int id, int16_t x, int16_t y) {}
    virtual void onTap(int16_t x, int16_t y) {}
    virtual void onLongPress(int16_t x, int16_t y) {}
    virtual void onSwipe(SwipeDirection direction, float velocity) {}
    virtual void onPinch(float scale) {}
    virtual void onRotate(float angle) {}
};
```

## Usage Example

Here's how to implement the gesture system in your application:

```cpp
class MyGestureHandler : public TouchGestureSystem {
public:
    MyGestureHandler(LGFX* disp) : TouchGestureSystem(disp) {
        // Customize gesture thresholds if needed
        config.tap_duration = 150;        // Faster tap detection
        config.swipe_threshold = 40;      // More sensitive swipes
        config.long_press_duration = 750; // Longer press required
    }
    
protected:
    void onTap(int16_t x, int16_t y) override {
        // Handle tap event
        Serial.printf("Tap at %d,%d\n", x, y);
    }
    
    void onLongPress(int16_t x, int16_t y) override {
        // Handle long press
        Serial.printf("Long press at %d,%d\n", x, y);
    }
    
    void onSwipe(SwipeDirection direction, float velocity) override {
        // Handle swipe with direction and speed
        const char* dir = "";
        switch (direction) {
            case SwipeDirection::Left:  dir = "Left"; break;
            case SwipeDirection::Right: dir = "Right"; break;
            case SwipeDirection::Up:    dir = "Up"; break;
            case SwipeDirection::Down:  dir = "Down"; break;
        }
        Serial.printf("Swipe %s (%.1f px/s)\n", dir, velocity);
    }
    
    void onPinch(float scale) override {
        // Handle pinch zoom
        Serial.printf("Pinch scale: %.2f\n", scale);
    }
    
    void onRotate(float angle) override {
        // Handle rotation
        Serial.printf("Rotation: %.1f degrees\n", angle);
    }
};
```

## Best Practices

1. **Gesture Detection**
   - Use appropriate thresholds for your use case
   - Consider screen size when setting distances
   - Implement proper debouncing
   - Handle edge cases properly

2. **Performance**
   - Optimize calculations
   - Use fast math where possible
   - Minimize memory allocations
   - Consider using lookup tables

3. **User Experience**
   - Provide visual feedback
   - Use appropriate timing
   - Consider accessibility
   - Maintain consistency

4. **Error Handling**
   - Handle touch glitches
   - Validate coordinates
   - Reset state appropriately
   - Implement timeout handling 

## Advanced Gesture Examples

### Momentum Scrolling Implementation

```cpp
class MomentumScroller : public TouchGestureSystem {
    float scroll_velocity;    // Current scroll velocity
    float friction;          // Deceleration rate
    float scroll_position;   // Current scroll position
    uint32_t last_update;    // Last update timestamp
    
public:
    MomentumScroller(LGFX* disp) 
        : TouchGestureSystem(disp)
        , scroll_velocity(0)
        , friction(0.95f)    // 5% velocity reduction per frame
        , scroll_position(0)
        , last_update(0)
    {}
    
protected:
    void onSwipe(SwipeDirection direction, float velocity) override {
        // Convert swipe velocity to scroll velocity
        // For vertical scrolling, only use vertical component
        if (direction == SwipeDirection::Up || direction == SwipeDirection::Down) {
            scroll_velocity = velocity * (direction == SwipeDirection::Up ? -1 : 1);
            last_update = millis();
        }
    }
    
    void update() {
        TouchGestureSystem::update();  // Handle base gesture detection
        
        // Apply momentum scrolling
        if (abs(scroll_velocity) > 0.1f) {  // Continue until nearly stopped
            uint32_t current_time = millis();
            float dt = (current_time - last_update) / 1000.0f;  // Time in seconds
            
            // Update position based on velocity
            scroll_position += scroll_velocity * dt;
            
            // Apply friction: velocity *= friction^(time_elapsed)
            scroll_velocity *= pow(friction, dt * 60);  // Scale friction to 60fps
            
            last_update = current_time;
            // Trigger redraw with new position
            updateScrollPosition(scroll_position);
        }
    }
};
```

### Multi-Touch Rotation and Scale

```cpp
class ImageManipulator : public TouchGestureSystem {
    float image_scale;     // Current image scale
    float image_rotation;  // Current rotation in degrees
    float image_x, image_y;  // Image position
    
public:
    ImageManipulator(LGFX* disp)
        : TouchGestureSystem(disp)
        , image_scale(1.0f)
        , image_rotation(0.0f)
        , image_x(disp->width() / 2)
        , image_y(disp->height() / 2)
    {
        // Increase sensitivity for rotation
        config.rotation_threshold = 2.0f;  // Detect smaller rotations
    }
    
protected:
    void onPinch(float scale) override {
        // Accumulate scale changes
        // Limit scale to reasonable range (0.1x to 10x)
        image_scale = constrain(
            image_scale * scale,  // Apply new scale factor
            0.1f,                 // Minimum scale
            10.0f                 // Maximum scale
        );
        redrawImage();
    }
    
    void onRotate(float angle) override {
        // Accumulate rotation changes
        // Keep angle in range [0, 360)
        image_rotation = fmod(
            image_rotation + angle + 360.0f,  // Add angle and normalize
            360.0f                           // Full circle
        );
        redrawImage();
    }
    
private:
    void redrawImage() {
        display->startWrite();  // Begin SPI transaction
        
        // Save current transform
        display->pushMatrix();
        
        // Apply transformations in correct order:
        // 1. Translate to pivot point
        // 2. Apply rotation
        // 3. Apply scale
        // 4. Translate back to draw position
        display->translate(image_x, image_y);
        display->rotate(image_rotation);
        display->scale(image_scale);
        display->translate(-image_x, -image_y);
        
        // Draw image here...
        
        // Restore original transform
        display->popMatrix();
        display->endWrite();
    }
};
```

### Gesture State Machine

```cpp
class GestureStateMachine : public TouchGestureSystem {
    enum class GestureState {
        None,
        Tapping,    // Potential tap in progress
        Dragging,   // Drag operation active
        Pinching,   // Pinch zoom active
        Rotating    // Rotation active
    };
    
    GestureState current_state;
    uint32_t state_start_time;
    
protected:
    void onTouchStart(int id, int16_t x, int16_t y) override {
        if (id == 0) {  // First touch
            setState(GestureState::Tapping);
        } else if (id == 1 && current_state == GestureState::Tapping) {
            // Second touch - transition to multi-touch state
            float initial_distance = getDistance(
                points[0].x, points[0].y,
                points[1].x, points[1].y
            );
            if (initial_distance > 100) {
                setState(GestureState::Pinching);
            } else {
                setState(GestureState::Rotating);
            }
        }
    }
    
private:
    void setState(GestureState new_state) {
        if (current_state != new_state) {
            onStateExit(current_state);  // Cleanup old state
            current_state = new_state;
            state_start_time = millis();
            onStateEnter(new_state);     // Initialize new state
        }
    }
};
``` 