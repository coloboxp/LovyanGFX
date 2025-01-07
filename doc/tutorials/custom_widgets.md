# Custom Widget Development Guide

This guide explains how to create custom widgets in LovyanGFX, including base widget functionality, event handling, and state management.

## Widget Base Class

The foundation for all custom widgets:

```cpp
class Widget {
protected:
    LGFX* display;           // Display reference
    int16_t x, y;           // Widget position
    uint16_t w, h;          // Widget dimensions
    uint16_t fg_color;      // Foreground color (text/borders)
    uint16_t bg_color;      // Background color
    bool is_visible;        // Visibility state
    bool needs_redraw;      // Redraw flag
    bool is_enabled;        // Enabled/disabled state
    bool is_focused;        // Focus state
    
public:
    // Constructor initializes widget with position and size
    Widget(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : display(disp)
        , x(x), y(y)
        , w(w), h(h)
        , fg_color(TFT_WHITE)
        , bg_color(TFT_BLACK)
        , is_visible(true)
        , needs_redraw(true)
        , is_enabled(true)
        , is_focused(false)
    {}
    
    virtual ~Widget() {}
    
    // Core widget methods that must be implemented
    virtual void draw() = 0;                    // Draw widget
    virtual void update() {}                    // Update logic
    virtual bool handleTouch(int16_t tx, int16_t ty, bool pressed) { return false; }
    
    // Position and size management
    void setPosition(int16_t nx, int16_t ny) {
        if (x != nx || y != ny) {
            x = nx;
            y = ny;
            needs_redraw = true;
        }
    }
    
    void setSize(uint16_t nw, uint16_t nh) {
        if (w != nw || h != nh) {
            w = nw;
            h = nh;
            needs_redraw = true;
            onResize();  // Notify derived classes
        }
    }
    
    // State management
    void setVisible(bool visible) {
        if (is_visible != visible) {
            is_visible = visible;
            needs_redraw = true;
        }
    }
    
    void setEnabled(bool enabled) {
        if (is_enabled != enabled) {
            is_enabled = enabled;
            needs_redraw = true;
            onEnableChanged();  // Notify derived classes
        }
    }
    
    // Focus handling
    virtual void setFocus(bool focused) {
        if (is_focused != focused) {
            is_focused = focused;
            needs_redraw = true;
            onFocusChanged();  // Notify derived classes
        }
    }
    
protected:
    // Virtual handlers for state changes
    virtual void onResize() {}
    virtual void onEnableChanged() {}
    virtual void onFocusChanged() {}
    
    // Hit testing for touch input
    bool hitTest(int16_t tx, int16_t ty) const {
        return tx >= x && tx < x + w && ty >= y && ty < y + h;
    }
};
```

## Example Custom Widgets

### Button Widget

A basic button implementation with states and callbacks:

```cpp
class Button : public Widget {
    std::string label;              // Button text
    bool is_pressed;               // Press state
    std::function<void()> callback; // Click callback
    uint16_t corner_radius;        // Corner rounding
    
public:
    Button(LGFX* disp, const char* text, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : Widget(disp, x, y, w, h)
        , label(text)
        , is_pressed(false)
        , corner_radius(5)
    {}
    
    void setCallback(std::function<void()> cb) {
        callback = cb;
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();  // Begin SPI transaction for multiple operations
        
        // Calculate background color based on state:
        // - If disabled: dim the background using bitwise AND with 0x7BEF 
        //   (reduces all color components while preserving color balance)
        // - If pressed: use foreground color to create inverse effect
        // - Otherwise: use normal background color
        uint16_t bg = is_enabled ? 
            (is_pressed ? fg_color : bg_color) :
            (bg_color & 0x7BEF);  // 0x7BEF = 0111 1011 1110 1111 in binary
        
        // Draw rounded rectangle with specified corner radius
        // Corner radius determines smoothness of corners
        display->fillRoundRect(x, y, w, h, corner_radius, bg);
        
        // When focused, draw double border for visual feedback
        // Outer border is 1 pixel larger on all sides (-1, -1, +2, +2)
        if (is_focused) {
            display->drawRoundRect(x-1, y-1, w+2, h+2, corner_radius, fg_color);
        }
        display->drawRoundRect(x, y, w, h, corner_radius, fg_color);
        
        // Set text color to create contrast with background
        // When pressed, invert colors for visual feedback
        display->setTextColor(is_pressed ? bg_color : fg_color);
        
        // Center text both horizontally and vertically
        display->setTextDatum(middle_center);
        
        // Draw text at center point (x + w/2, y + h/2)
        display->drawString(label.c_str(), x + w/2, y + h/2);
        
        display->endWrite();
        needs_redraw = false;
    }
    
    bool handleTouch(int16_t tx, int16_t ty, bool pressed) override {
        if (!is_visible || !is_enabled) return false;
        
        if (hitTest(tx, ty)) {
            if (is_pressed != pressed) {
                is_pressed = pressed;
                needs_redraw = true;
                
                if (!pressed && callback) {
                    callback();  // Trigger callback on release
                }
            }
            return true;  // Touch handled
        }
        
        if (is_pressed) {
            is_pressed = false;
            needs_redraw = true;
        }
        return false;
    }
};
```

### Progress Bar Widget

A progress indicator with customizable appearance:

```cpp
class ProgressBar : public Widget {
    float progress;          // Current progress (0.0-1.0)
    bool show_text;          // Show percentage text
    uint16_t border_color;   // Border color
    
public:
    ProgressBar(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : Widget(disp, x, y, w, h)
        , progress(0.0f)
        , show_text(true)
        , border_color(TFT_WHITE)
    {}
    
    void setProgress(float value) {
        // Clamp progress to 0.0-1.0
        float new_progress = std::max(0.0f, std::min(1.0f, value));
        if (progress != new_progress) {
            progress = new_progress;
            needs_redraw = true;
        }
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Draw border
        display->drawRect(x, y, w, h, border_color);
        
        // Draw progress fill
        uint16_t fill_width = (w - 2) * progress;
        if (fill_width > 0) {
            display->fillRect(x + 1, y + 1, fill_width, h - 2, fg_color);
        }
        
        // Draw percentage text
        if (show_text) {
            char buf[8];
            snprintf(buf, sizeof(buf), "%d%%", (int)(progress * 100));
            
            display->setTextColor(
                progress > 0.5f ? bg_color : fg_color
            );
            display->setTextDatum(middle_center);
            display->drawString(buf, x + w/2, y + h/2);
        }
        
        display->endWrite();
        needs_redraw = false;
    }
};
```

### Slider Widget

A customizable slider control for value selection:

```cpp
class Slider : public Widget {
    float value;            // Current value (0.0-1.0)
    float min_value;        // Minimum value
    float max_value;        // Maximum value
    bool is_vertical;       // Orientation flag
    bool is_dragging;       // Drag state
    uint16_t handle_size;   // Size of slider handle
    std::function<void(float)> on_change; // Value change callback
    
public:
    Slider(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h, bool vertical = false)
        : Widget(disp, x, y, w, h)
        , value(0.0f)
        , min_value(0.0f)
        , max_value(1.0f)
        , is_vertical(vertical)
        , is_dragging(false)
        , handle_size(20)
    {}
    
    void setValue(float new_value) {
        // Convert input value to normalized range (0.0-1.0)
        // Formula: normalized = (value - min) / (max - min)
        // This maps any input range to 0.0-1.0 scale
        float normalized = (new_value - min_value) / (max_value - min_value);
        
        // Clamp value to valid range using std::max/min
        // This ensures value stays between 0.0 and 1.0
        normalized = std::max(0.0f, std::min(1.0f, normalized));
        
        if (value != normalized) {
            value = normalized;
            needs_redraw = true;
            if (on_change) {
                // Convert back to user range when triggering callback
                on_change(getValue());
            }
        }
    }
    
    float getValue() const {
        return min_value + value * (max_value - min_value);
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        if (is_vertical) {
            // For vertical slider:
            // Center track horizontally using (w - track_width) / 2
            int16_t track_x = x + (w - 4) / 2;
            display->fillRect(track_x, y, 4, h, fg_color);
            
            // Calculate handle position:
            // - Subtract handle_size from height for valid range
            // - Invert value (1.0f - value) because screen coordinates 
            //   increase downward
            int16_t handle_y = y + (h - handle_size) * (1.0f - value);
            display->fillRoundRect(
                x, handle_y,
                w, handle_size,
                handle_size/2,  // Radius = half of handle size for circle
                is_dragging ? TFT_RED : TFT_WHITE
            );
        } else {
            // For horizontal slider:
            // Center track vertically using (h - track_height) / 2
            int16_t track_y = y + (h - 4) / 2;
            display->fillRect(x, track_y, w, 4, fg_color);
            
            // Calculate handle position:
            // - Subtract handle_size from width for valid range
            // - Multiply by value (0.0-1.0) for position
            int16_t handle_x = x + (w - handle_size) * value;
            display->fillRoundRect(
                handle_x, y,
                handle_size, h,
                handle_size/2,
                is_dragging ? TFT_RED : TFT_WHITE
            );
        }
        
        display->endWrite();
        needs_redraw = false;
    }
    
    bool handleTouch(int16_t tx, int16_t ty, bool pressed) override {
        if (!is_visible || !is_enabled) return false;
        
        if (pressed && hitTest(tx, ty)) {
            is_dragging = true;
            updateValueFromTouch(tx, ty);
            return true;
        }
        
        if (is_dragging) {
            if (pressed) {
                updateValueFromTouch(tx, ty);
            } else {
                is_dragging = false;
                needs_redraw = true;
            }
            return true;
        }
        
        return false;
    }
    
private:
    void updateValueFromTouch(int16_t tx, int16_t ty) {
        float new_value;
        if (is_vertical) {
            // For vertical slider:
            // 1. Calculate relative Y position (ty - y)
            // 2. Divide by available range (h - handle_size)
            // 3. Invert (1.0f - ...) because screen coordinates increase downward
            new_value = 1.0f - (float)(ty - y) / (h - handle_size);
        } else {
            // For horizontal slider:
            // 1. Calculate relative X position (tx - x)
            // 2. Divide by available range (w - handle_size)
            new_value = (float)(tx - x) / (w - handle_size);
        }
        setValue(new_value);
    }
};
```

### Toggle Switch Widget

A modern toggle switch implementation:

```cpp
class ToggleSwitch : public Widget {
    bool is_on;                    // Switch state
    std::function<void(bool)> on_change;  // State change callback
    static constexpr float ANIMATION_SPEED = 0.2f;  // Animation duration in seconds
    float animation_progress;      // Animation progress (0.0-1.0)
    uint32_t animation_start;      // Animation start timestamp
    
    void update() override {
        if (animation_progress < 1.0f) {
            // Convert milliseconds to seconds and calculate progress
            // elapsed = (current_time - start_time) / 1000.0f
            float elapsed = (millis() - animation_start) / 1000.0f;
            
            // Calculate progress as fraction of ANIMATION_SPEED
            // Clamp to 1.0 to prevent overshooting
            animation_progress = std::min(1.0f, elapsed / ANIMATION_SPEED);
            needs_redraw = true;
        }
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Calculate animated position:
        // - For ON state: progress goes from 0.0 to 1.0
        // - For OFF state: progress goes from 1.0 to 0.0
        // This creates smooth transition in both directions
        float current = is_on ? 
            animation_progress :    // ON: 0.0 -> 1.0
            1.0f - animation_progress;  // OFF: 1.0 -> 0.0
            
        // Select background color based on state:
        // - Enabled + ON: Green
        // - Enabled + OFF: Dark grey
        // - Disabled: Always dark grey
        uint16_t bg = is_enabled ?
            (is_on ? TFT_GREEN : TFT_DARKGREY) :
            TFT_DARKGREY;
            
        // Draw rounded rectangle background
        // Use h/2 for radius to create pill shape
        display->fillRoundRect(x, y, w, h, h/2, bg);
        
        // Calculate handle position:
        // - Start at x + 2 (2px padding)
        // - Available movement range is (w - h + 2)
        // - Multiply by current progress (0.0-1.0)
        int16_t handle_x = x + 2 + (w - h + 2) * current;
        
        // Draw circular handle:
        // - Center horizontally using (h-4)/2
        // - Center vertically using h/2
        // - Radius is (h-4)/2 for padding
        // - Color is white when enabled, light grey when disabled
        display->fillCircle(
            handle_x + (h-4)/2,  // Center X
            y + h/2,             // Center Y
            (h-4)/2,            // Radius
            is_enabled ? TFT_WHITE : TFT_LIGHTGREY
        );
        
        display->endWrite();
        needs_redraw = false;
    }
    
private:
    void startAnimation() {
        // Reset animation state
        animation_progress = 0.0f;  // Start from beginning
        animation_start = millis(); // Record start time
        needs_redraw = true;        // Request redraw
    }
};
```

### Scroll Bar Widget

A customizable scroll bar implementation:

```cpp
class ScrollBar : public Widget {
    float position;         // Current scroll position (0.0-1.0)
    float visible_ratio;    // Visible content ratio (0.0-1.0)
    bool is_vertical;       // Orientation flag
    bool is_dragging;      // Drag state
    int16_t drag_offset;   // Drag start offset
    
public:
    ScrollBar(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h, 
             bool vertical = true)
        : Widget(disp, x, y, w, h)
        , position(0.0f)
        , visible_ratio(1.0f)
        , is_vertical(vertical)
        , is_dragging(false)
        , drag_offset(0)
    {}
    
    void setPosition(float pos) {
        // Clamp position based on visible ratio:
        // Maximum position = 1.0 - visible_ratio
        // This prevents scrolling past the end
        float max_pos = std::max(0.0f, 1.0f - visible_ratio);
        position = std::max(0.0f, std::min(max_pos, pos));
        needs_redraw = true;
    }
    
    void setVisibleRatio(float ratio) {
        // Clamp ratio between 0.0 and 1.0
        // ratio = visible_size / total_size
        visible_ratio = std::max(0.0f, std::min(1.0f, ratio));
        
        // Adjust position if needed to prevent invalid scroll
        setPosition(position);
        needs_redraw = true;
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Draw track background
        display->fillRect(x, y, w, h, TFT_DARKGREY);
        
        if (visible_ratio < 1.0f) {  // Only draw handle if content is scrollable
            // Calculate handle size and position:
            // - Size is proportional to visible_ratio
            // - Position is proportional to current scroll position
            if (is_vertical) {
                // Calculate handle dimensions
                uint16_t handle_height = std::max(20u, (uint16_t)(h * visible_ratio));
                uint16_t travel = h - handle_height;
                int16_t handle_y = y + travel * position;
                
                // Draw handle
                display->fillRect(x, handle_y, w, handle_height, 
                    is_dragging ? TFT_WHITE : TFT_LIGHTGREY);
            } else {
                // Similar calculations for horizontal orientation
                uint16_t handle_width = std::max(20u, (uint16_t)(w * visible_ratio));
                uint16_t travel = w - handle_width;
                int16_t handle_x = x + travel * position;
                
                display->fillRect(handle_x, y, handle_width, h,
                    is_dragging ? TFT_WHITE : TFT_LIGHTGREY);
            }
        }
        
        display->endWrite();
        needs_redraw = false;
    }
    
    bool handleTouch(int16_t tx, int16_t ty, bool pressed) override {
        if (!is_visible || !is_enabled || visible_ratio >= 1.0f) return false;
        
        if (pressed) {
            if (!is_dragging && hitTest(tx, ty)) {
                // Start dragging - calculate offset from handle position
                is_dragging = true;
                if (is_vertical) {
                    uint16_t handle_height = std::max(20u, (uint16_t)(h * visible_ratio));
                    uint16_t travel = h - handle_height;
                    int16_t handle_y = y + travel * position;
                    drag_offset = ty - handle_y;
                } else {
                    uint16_t handle_width = std::max(20u, (uint16_t)(w * visible_ratio));
                    uint16_t travel = w - handle_width;
                    int16_t handle_x = x + travel * position;
                    drag_offset = tx - handle_x;
                }
                return true;
            }
        } else if (is_dragging) {
            is_dragging = false;
            needs_redraw = true;
            return true;
        }
        
        if (is_dragging) {
            // Update position based on touch point and initial offset
            if (is_vertical) {
                uint16_t handle_height = std::max(20u, (uint16_t)(h * visible_ratio));
                uint16_t travel = h - handle_height;
                float new_pos = (float)(ty - y - drag_offset) / travel;
                setPosition(new_pos);
            } else {
                uint16_t handle_width = std::max(20u, (uint16_t)(w * visible_ratio));
                uint16_t travel = w - handle_width;
                float new_pos = (float)(tx - x - drag_offset) / travel;
                setPosition(new_pos);
            }
            return true;
        }
        
        return false;
    }
};
```

## Best Practices

1. **State Management**
   - Track widget state changes
   - Update visual appearance accordingly
   - Handle enabled/disabled states
   - Manage focus properly

2. **Drawing Optimization**
   - Use startWrite/endWrite for multiple operations
   - Only redraw when needed
   - Implement partial updates when possible
   - Cache complex calculations

3. **Event Handling**
   - Validate touch input
   - Handle edge cases
   - Provide appropriate feedback
   - Use proper event bubbling

4. **Memory Management**
   - Clean up resources in destructor
   - Minimize dynamic allocations
   - Handle string storage efficiently
   - Consider using static buffers

## Common Issues and Solutions

1. **Touch Responsiveness**
```cpp
// Implement touch debouncing
class Button : public Widget {
    uint32_t last_touch_time = 0;
    static constexpr uint32_t DEBOUNCE_MS = 50;
    
    bool handleTouch(int16_t tx, int16_t ty, bool pressed) override {
        uint32_t current_time = millis();
        if (current_time - last_touch_time < DEBOUNCE_MS) {
            return false;
        }
        last_touch_time = current_time;
        // Normal touch handling...
    }
};
```

2. **Visual Feedback**
```cpp
// Implement press feedback
void drawPressEffect() {
    // Draw darker background when pressed
    uint16_t pressed_color = (bg_color & 0xF7DE) >> 1;
    display->fillRect(x + 2, y + 2, w - 4, h - 4, pressed_color);
}
```

3. **Focus Management**
```cpp
// Implement focus navigation
bool handleKeyInput(uint8_t key) {
    if (!is_enabled) return false;
    
    switch (key) {
        case KEY_ENTER:
            if (is_focused && callback) {
                callback();
                return true;
            }
            break;
    }
    return false;
}
``` 