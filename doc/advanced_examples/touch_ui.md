# Touch UI Implementation

This guide demonstrates how to create interactive touch-based user interfaces using LovyanGFX's touch capabilities. The implementation includes a robust event system and reusable UI components.

## Touch Event System

### Basic Touch Handler

The `TouchHandler` class provides a foundation for touch input management:

```cpp
class TouchHandler {
private:
    LGFX* _display;                // Display driver reference
    int16_t _last_x;              // Previous touch X coordinate
    int16_t _last_y;              // Previous touch Y coordinate
    bool _was_touched;            // Previous touch state
    uint32_t _touch_start_time;   // Timestamp of touch start
    bool _tracking;               // Whether currently tracking touch

public:
    TouchHandler(LGFX* display)
    : _display(display)
    , _last_x(-1)                 // Initialize to invalid coordinate
    , _last_y(-1)                 // Initialize to invalid coordinate
    , _was_touched(false)         // Start with no touch
    , _touch_start_time(0)        // No touch duration yet
    , _tracking(false)            // Not tracking initially
    {
    }

    // Update touch state and trigger appropriate events
    void update() {
        // Skip if touch hardware not available
        if (!_display->touch()) return;

        // Get current touch state
        int16_t x, y;
        bool touched = _display->getTouch(&x, &y);

        if (touched) {
            if (!_was_touched) {
                // New touch detected
                _touch_start_time = millis();  // Record start time
                _tracking = true;              // Begin tracking
                onTouchStart(x, y);           // Trigger start event
            } else {
                // Continuing touch
                // Calculate movement delta from last position
                onTouchMove(x, y, x - _last_x, y - _last_y);
            }
        } else if (_was_touched) {
            // Touch just ended
            // Calculate total touch duration
            onTouchEnd(x, y, millis() - _touch_start_time);
            _tracking = false;  // Stop tracking
        }

        // Update previous state
        _last_x = x;
        _last_y = y;
        _was_touched = touched;
    }

protected:
    // Touch event handlers to be implemented by derived classes
    virtual void onTouchStart(int16_t x, int16_t y) {}
    virtual void onTouchMove(int16_t x, int16_t y, int16_t dx, int16_t dy) {}
    virtual void onTouchEnd(int16_t x, int16_t y, uint32_t duration) {}
};
```

## UI Components

### Base Widget

The `Widget` class serves as the foundation for all UI elements:

```cpp
class Widget {
protected:
    int16_t _x, _y;              // Widget position
    int16_t _width, _height;     // Widget dimensions
    bool _visible;               // Visibility flag
    bool _enabled;               // Interaction enabled flag
    LGFX_Sprite* _sprite;        // Buffer for widget rendering

public:
    Widget(int16_t x, int16_t y, int16_t w, int16_t h)
    : _x(x)
    , _y(y)
    , _width(w)
    , _height(h)
    , _visible(true)             // Visible by default
    , _enabled(true)             // Enabled by default
    , _sprite(nullptr)           // No sprite buffer initially
    {
    }

    // Clean up sprite buffer on destruction
    virtual ~Widget() {
        if (_sprite) {
            _sprite->deleteSprite();
            delete _sprite;
        }
    }

    // Pure virtual draw function
    // Each widget must implement its own drawing logic
    virtual void draw(LGFX_Sprite* target) = 0;
    
    // Hit testing for touch input
    // Returns true if point (x,y) is within widget bounds
    virtual bool hitTest(int16_t x, int16_t y) {
        return _visible && _enabled &&
               x >= _x && x < _x + _width &&
               y >= _y && y < _y + _height;
    }

    // Handle touch events
    // Default implementation does nothing
    virtual void handleTouch(int16_t x, int16_t y, bool pressed) {}
};
```

### Button Widget

The `Button` class implements a customizable interactive button:

```cpp
class Button : public Widget {
private:
    String _text;                // Button label
    uint32_t _bg_color;         // Background color
    uint32_t _text_color;       // Text color
    uint32_t _border_color;     // Border color
    uint8_t _corner_radius;     // Corner rounding
    std::function<void()> _callback;  // Click handler
    bool _pressed;              // Current press state

public:
    Button(int16_t x, int16_t y, int16_t w, int16_t h, const String& text)
    : Widget(x, y, w, h)
    , _text(text)
    , _bg_color(TFT_BLUE)       // Default colors
    , _text_color(TFT_WHITE)
    , _border_color(TFT_DARKGREY)
    , _corner_radius(5)         // Default corner rounding
    , _pressed(false)
    {
        // Create sprite buffer for button
        _sprite = new LGFX_Sprite(nullptr);
        _sprite->createSprite(w, h);
    }

    // Configure button colors
    void setColors(uint32_t bg, uint32_t text, uint32_t border) {
        _bg_color = bg;
        _text_color = text;
        _border_color = border;
    }

    // Set click handler function
    void setCallback(std::function<void()> callback) {
        _callback = callback;
    }

    // Draw button with current state
    void draw(LGFX_Sprite* target) override {
        if (!_visible) return;

        // Fill background with state-dependent color
        _sprite->fillSprite(_pressed ? darken(_bg_color) : _bg_color);
        
        // Draw rounded border
        _sprite->drawRoundRect(0, 0, _width, _height, _corner_radius, _border_color);
        
        // Draw centered text
        _sprite->setTextColor(_text_color);
        _sprite->setTextDatum(middle_center);
        _sprite->drawString(_text, _width/2, _height/2);
        
        // Copy to target buffer
        _sprite->pushSprite(target, _x, _y);
    }

    // Handle touch events
    void handleTouch(int16_t x, int16_t y, bool pressed) override {
        if (!_enabled) return;
        
        // Update press state
        bool was_pressed = _pressed;
        _pressed = pressed && hitTest(x, y);
        
        // Trigger callback on release within bounds
        if (was_pressed && !_pressed && _callback) {
            _callback();
        }
    }

private:
    // Darken color for press effect
    // Reduces each RGB component by 30%
    uint32_t darken(uint32_t color) {
        uint8_t r = (color >> 16) & 0xFF;
        uint8_t g = (color >> 8) & 0xFF;
        uint8_t b = color & 0xFF;
        
        r = r * 0.7;  // Reduce red
        g = g * 0.7;  // Reduce green
        b = b * 0.7;  // Reduce blue
        
        return (r << 16) | (g << 8) | b;
    }
};
```

### Slider Widget

The `Slider` class implements a customizable value slider control:

```cpp
class Slider : public Widget {
private:
    float _value;                // Current slider value
    float _min_value;           // Minimum allowed value
    float _max_value;           // Maximum allowed value
    uint32_t _track_color;      // Color of slider track
    uint32_t _thumb_color;      // Color of slider thumb
    bool _vertical;             // Orientation flag
    std::function<void(float)> _callback;  // Value change handler

public:
    // Constructor with position, size, and value range
    // Parameters:
    // - x, y: Position coordinates
    // - w, h: Dimensions
    // - min, max: Value range
    // - vertical: Orientation (false = horizontal)
    Slider(int16_t x, int16_t y, int16_t w, int16_t h, 
           float min = 0.0f, float max = 1.0f, bool vertical = false)
    : Widget(x, y, w, h)
    , _value(min)               // Start at minimum value
    , _min_value(min)
    , _max_value(max)
    , _track_color(TFT_DARKGREY)  // Default colors
    , _thumb_color(TFT_BLUE)
    , _vertical(vertical)
    {
        // Create sprite buffer for slider
        _sprite = new LGFX_Sprite(nullptr);
        _sprite->createSprite(w, h);
    }

    // Set slider value with range validation
    void setValue(float value) {
        _value = constrain(value, _min_value, _max_value);
    }

    // Get current slider value
    float getValue() const { return _value; }

    // Set value change callback handler
    void setCallback(std::function<void(float)> callback) {
        _callback = callback;
    }

    // Draw slider with current state
    void draw(LGFX_Sprite* target) override {
        if (!_visible) return;

        // Clear sprite buffer
        _sprite->fillSprite(TFT_BLACK);
        
        if (_vertical) {
            // Draw vertical track
            // Center track horizontally with 4-pixel width
            int16_t track_x = _width / 2 - 2;
            _sprite->fillRect(track_x, 0, 4, _height, _track_color);
            
            // Calculate thumb position based on value
            // Map value range to pixel coordinates
            float thumb_y = (_value - _min_value) / 
                          (_max_value - _min_value) * _height;
            
            // Draw thumb (circle)
            // Position inverted for natural up=high mapping
            _sprite->fillCircle(
                _width/2,           // Center horizontally
                _height - thumb_y,  // Position based on value
                8,                  // Radius (16px diameter)
                _thumb_color
            );
        } else {
            // Draw horizontal track
            // Center track vertically with 4-pixel height
            int16_t track_y = _height / 2 - 2;
            _sprite->fillRect(0, track_y, _width, 4, _track_color);
            
            // Calculate thumb position based on value
            // Map value range to pixel coordinates
            float thumb_x = (_value - _min_value) / 
                          (_max_value - _min_value) * _width;
            
            // Draw thumb (circle)
            _sprite->fillCircle(
                thumb_x,            // Position based on value
                _height/2,          // Center vertically
                8,                  // Radius (16px diameter)
                _thumb_color
            );
        }
        
        // Copy to target buffer
        _sprite->pushSprite(target, _x, _y);
    }

    // Handle touch input
    void handleTouch(int16_t x, int16_t y, bool pressed) override {
        if (!_enabled || !pressed) return;
        
        // Check if touch is within slider bounds
        if (hitTest(x, y)) {
            float new_value;
            
            if (_vertical) {
                // Calculate value from vertical position
                // Convert touch coordinates to relative position
                float relative_y = y - _y;
                // Invert and map to value range
                new_value = _max_value - 
                           (relative_y / _height) * 
                           (_max_value - _min_value);
            } else {
                // Calculate value from horizontal position
                // Convert touch coordinates to relative position
                float relative_x = x - _x;
                // Map to value range
                // Map touch position to value range:
                // 1. relative_x / _width gives position ratio (0.0 to 1.0)
                // 2. Multiply by value range to get offset from min
                // 3. Add min value to get final value in range
                new_value = _min_value + 
                           (relative_x / _width) *  // Convert position to 0-1 ratio
                           (_max_value - _min_value); // Scale to full value range
            }
            
            // Update value and notify
            setValue(new_value);
            if (_callback) _callback(_value);
        }
    }
};
```

## Usage Example

This example demonstrates how to create a complete touch-based UI:

```cpp
// TouchUI class combines touch handling and widget management
class TouchUI : public TouchHandler {
private:
    LGFX* _display;                    // Display driver reference
    LGFX_Sprite* _buffer;             // Double-buffer for smooth rendering
    std::vector<Widget*> _widgets;    // Collection of UI widgets

public:
    TouchUI(LGFX* display) : TouchHandler(display), _display(display) {
        // Create full-screen double buffer
        _buffer = new LGFX_Sprite(_display);
        _buffer->createSprite(_display->width(), _display->height());
        
        // Create example UI elements
        
        // Add a button widget
        auto button = new Button(
            10, 10,         // X, Y position
            100, 40,        // Width, Height
            "Click Me"      // Button text
        );
        // Set button click handler
        button->setCallback([this]() {
            Serial.println("Button clicked!");
        });
        _widgets.push_back(button);
        
        // Add a horizontal slider widget
        auto slider = new Slider(
            10, 60,         // X, Y position
            200, 30,        // Width, Height
            0.0f, 1.0f,     // Value range
            false           // Horizontal orientation
        );
        // Set slider value change handler
        slider->setCallback([](float value) {
            Serial.printf("Slider value: %.2f\n", value);
        });
        _widgets.push_back(slider);
    }

    // Main update loop
    void update() {
        // Process touch input
        TouchHandler::update();
        
        // Clear buffer for new frame
        _buffer->fillScreen(TFT_BLACK);
        
        // Draw all widgets
        for (auto widget : _widgets) {
            widget->draw(_buffer);
        }
        
        // Push buffer to display
        _buffer->pushSprite(0, 0);
    }

protected:
    // Touch event handlers
    
    // Handle touch start event
    void onTouchStart(int16_t x, int16_t y) override {
        // Notify all widgets of touch state
        for (auto widget : _widgets) {
            widget->handleTouch(x, y, true);
        }
    }

    // Handle touch end event
    void onTouchEnd(int16_t x, int16_t y, uint32_t duration) override {
        // Notify all widgets of touch state
        for (auto widget : _widgets) {
            widget->handleTouch(x, y, false);
        }
    }
};
```

## Best Practices

1. Touch Handling
   ```cpp
   // Implement touch debouncing
   class DebouncedTouch {
   private:
       uint32_t _last_change;
       uint32_t _debounce_ms;
       bool _state;
       
   public:
       DebouncedTouch(uint32_t debounce_ms = 50)
       : _last_change(0)
       , _debounce_ms(debounce_ms)
       , _state(false)
       {}
       
       bool update(bool raw_state) {
           uint32_t now = millis();
           if (now - _last_change >= _debounce_ms) {
               if (raw_state != _state) {
                   _state = raw_state;
                   _last_change = now;
               }
           }
           return _state;
       }
   };
   
   // Handle multi-touch if supported
   struct TouchPoint {
       int16_t x, y;      // Coordinates
       uint16_t id;       // Touch point identifier
       uint16_t pressure; // Touch pressure (if supported)
   };
   
   // Consider touch pressure
   void handleTouch(TouchPoint& point) {
       if (point.pressure > pressure_threshold) {
           // Process touch event
       }
   }
   
   // Validate touch coordinates
   bool validateTouch(int16_t x, int16_t y) {
       return x >= 0 && x < display.width() &&
              y >= 0 && y < display.height();
   }
   ```

2. UI Components
   ```cpp
   // Use consistent styling
   struct UIStyle {
       uint32_t bg_color;      // Background color
       uint32_t text_color;    // Text color
       uint32_t accent_color;  // Accent color
       uint8_t corner_radius;  // Corner rounding
       uint8_t border_width;   // Border thickness
   };
   
   // Implement proper hit testing
   bool Widget::hitTest(int16_t x, int16_t y) {
       // First check if the touch point is within the widget's rectangular bounds
       // _x, _y: Top-left corner coordinates of the widget
       // _width, _height: Dimensions of the widget's bounding box
       
       // Check if x coordinate is outside widget bounds
       if (x < _x ||                // Touch is left of widget
           x >= _x + _width) {      // Touch is right of widget
           return false;            // Touch point is outside horizontal bounds
       }
       
       // Check if y coordinate is outside widget bounds
       if (y < _y ||               // Touch is above widget
           y >= _y + _height) {    // Touch is below widget
           return false;           // Touch point is outside vertical bounds
       }
       
       // Check shape-specific hit area
       return isPointInShape(x - _x, y - _y);
   }
   
   // Handle state changes
   void Widget::setState(WidgetState new_state) {
       if (_state != new_state) {
           _state = new_state;
           invalidate();  // Request redraw
       }
   }
   
   // Provide visual feedback
   void Button::updateVisuals() {
       switch (_state) {
           case NORMAL:
               _current_color = _normal_color;
               break;
           case HOVER:
               _current_color = lighten(_normal_color);
               break;
           case PRESSED:
               _current_color = darken(_normal_color);
               break;
       }
   }
   ```

3. Performance
   ```cpp
   // Use sprite buffering
   class BufferedWidget : public Widget {
   private:
       LGFX_Sprite _buffer;
       bool _needs_update;
       
   public:
       void draw() {
           if (_needs_update) {
               updateBuffer();
               _needs_update = false;
           }
           _buffer.pushSprite(_x, _y);
       }
   };
   
   // Minimize redraws
   void Widget::invalidate(const Rect& area) {
       if (_dirty_region.isEmpty()) {
           _dirty_region = area;
       } else {
           _dirty_region.extend(area);
       }
   }
   
   // Optimize touch polling
   void TouchUI::update() {
       static uint32_t last_poll = 0;
       uint32_t now = millis();
       
       if (now - last_poll >= TOUCH_POLL_MS) {
           checkTouch();
           last_poll = now;
       }
   }
   
   // Handle memory constraints
   void Widget::createBuffer() {
       size_t required = _width * _height * 2;  // 16-bit color
       if (heap_caps_get_free_size(MALLOC_CAP_DMA) > required) {
           _buffer.createSprite(_width, _height);
       } else {
           // Fall back to direct drawing
           _use_buffer = false;
       }
   }
   ```

4. Error Handling
   ```cpp
   // Validate touch input
   bool TouchHandler::processTouch() {
       int16_t x, y;
       if (!_display->getTouch(&x, &y)) {
           return false;  // Touch read failed
       }
       
       if (!validateCoordinates(x, y)) {
           return false;  // Invalid coordinates
       }
       
       return true;
   }
   
   // Handle memory allocation failures
   Widget* createWidget(WidgetType type) {
       Widget* widget = nullptr;
       try {
           widget = new Widget(type);
           if (!widget->initialize()) {
               delete widget;
               return nullptr;
           }
       } catch (std::bad_alloc&) {
           return nullptr;  // Memory allocation failed
       }
       return widget;
   }
   
   // Implement component bounds checking
   void Widget::setPosition(int16_t x, int16_t y) {
       // Ensure widget stays within display bounds
       _x = constrain(x, 0, _display->width() - _width);
       _y = constrain(y, 0, _display->height() - _height);
   }
   ```

These examples demonstrate how to create a robust touch-based UI system using LovyanGFX, with proper event handling, widget management, and error handling. 