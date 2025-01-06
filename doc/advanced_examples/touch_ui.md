# Touch UI Implementation

This guide demonstrates how to create interactive touch-based user interfaces using LovyanGFX's touch capabilities.

## Touch Event System

### Basic Touch Handler

```cpp
class TouchHandler {
private:
    LGFX* _display;
    int16_t _last_x;
    int16_t _last_y;
    bool _was_touched;
    uint32_t _touch_start_time;
    bool _tracking;

public:
    TouchHandler(LGFX* display)
    : _display(display)
    , _last_x(-1)
    , _last_y(-1)
    , _was_touched(false)
    , _touch_start_time(0)
    , _tracking(false)
    {
    }

    void update() {
        if (!_display->touch()) return;

        int16_t x, y;
        bool touched = _display->getTouch(&x, &y);

        if (touched) {
            if (!_was_touched) {
                // Touch start
                _touch_start_time = millis();
                _tracking = true;
                onTouchStart(x, y);
            } else {
                // Touch move
                onTouchMove(x, y, x - _last_x, y - _last_y);
            }
        } else if (_was_touched) {
            // Touch end
            onTouchEnd(x, y, millis() - _touch_start_time);
            _tracking = false;
        }

        _last_x = x;
        _last_y = y;
        _was_touched = touched;
    }

protected:
    virtual void onTouchStart(int16_t x, int16_t y) {}
    virtual void onTouchMove(int16_t x, int16_t y, int16_t dx, int16_t dy) {}
    virtual void onTouchEnd(int16_t x, int16_t y, uint32_t duration) {}
};
```

## UI Components

### Base Widget

```cpp
class Widget {
protected:
    int16_t _x, _y;
    int16_t _width, _height;
    bool _visible;
    bool _enabled;
    LGFX_Sprite* _sprite;

public:
    Widget(int16_t x, int16_t y, int16_t w, int16_t h)
    : _x(x)
    , _y(y)
    , _width(w)
    , _height(h)
    , _visible(true)
    , _enabled(true)
    , _sprite(nullptr)
    {
    }

    virtual ~Widget() {
        if (_sprite) {
            _sprite->deleteSprite();
            delete _sprite;
        }
    }

    virtual void draw(LGFX_Sprite* target) = 0;
    
    virtual bool hitTest(int16_t x, int16_t y) {
        return _visible && _enabled &&
               x >= _x && x < _x + _width &&
               y >= _y && y < _y + _height;
    }

    virtual void handleTouch(int16_t x, int16_t y, bool pressed) {}
};
```

### Button Widget

```cpp
class Button : public Widget {
private:
    String _text;
    uint32_t _bg_color;
    uint32_t _text_color;
    uint32_t _border_color;
    uint8_t _corner_radius;
    std::function<void()> _callback;
    bool _pressed;

public:
    Button(int16_t x, int16_t y, int16_t w, int16_t h, const String& text)
    : Widget(x, y, w, h)
    , _text(text)
    , _bg_color(TFT_BLUE)
    , _text_color(TFT_WHITE)
    , _border_color(TFT_DARKGREY)
    , _corner_radius(5)
    , _pressed(false)
    {
        _sprite = new LGFX_Sprite(nullptr);
        _sprite->createSprite(w, h);
    }

    void setColors(uint32_t bg, uint32_t text, uint32_t border) {
        _bg_color = bg;
        _text_color = text;
        _border_color = border;
    }

    void setCallback(std::function<void()> callback) {
        _callback = callback;
    }

    void draw(LGFX_Sprite* target) override {
        if (!_visible) return;

        _sprite->fillSprite(_pressed ? darken(_bg_color) : _bg_color);
        _sprite->drawRoundRect(0, 0, _width, _height, _corner_radius, _border_color);
        
        _sprite->setTextColor(_text_color);
        _sprite->setTextDatum(middle_center);
        _sprite->drawString(_text, _width/2, _height/2);
        
        _sprite->pushSprite(target, _x, _y);
    }

    void handleTouch(int16_t x, int16_t y, bool pressed) override {
        if (!_enabled) return;
        
        bool was_pressed = _pressed;
        _pressed = pressed && hitTest(x, y);
        
        if (was_pressed && !_pressed && _callback) {
            _callback();
        }
    }

private:
    uint32_t darken(uint32_t color) {
        uint8_t r = (color >> 16) & 0xFF;
        uint8_t g = (color >> 8) & 0xFF;
        uint8_t b = color & 0xFF;
        
        r = r * 0.7;
        g = g * 0.7;
        b = b * 0.7;
        
        return (r << 16) | (g << 8) | b;
    }
};
```

### Slider Widget

```cpp
class Slider : public Widget {
private:
    float _value;
    float _min_value;
    float _max_value;
    uint32_t _track_color;
    uint32_t _thumb_color;
    bool _vertical;
    std::function<void(float)> _callback;

public:
    Slider(int16_t x, int16_t y, int16_t w, int16_t h, 
           float min = 0.0f, float max = 1.0f, bool vertical = false)
    : Widget(x, y, w, h)
    , _value(min)
    , _min_value(min)
    , _max_value(max)
    , _track_color(TFT_DARKGREY)
    , _thumb_color(TFT_BLUE)
    , _vertical(vertical)
    {
        _sprite = new LGFX_Sprite(nullptr);
        _sprite->createSprite(w, h);
    }

    void setValue(float value) {
        _value = constrain(value, _min_value, _max_value);
    }

    float getValue() const { return _value; }

    void setCallback(std::function<void(float)> callback) {
        _callback = callback;
    }

    void draw(LGFX_Sprite* target) override {
        if (!_visible) return;

        _sprite->fillSprite(TFT_BLACK);
        
        // Draw track
        if (_vertical) {
            int16_t track_x = _width / 2 - 2;
            _sprite->fillRect(track_x, 0, 4, _height, _track_color);
            
            // Draw thumb
            float thumb_y = (_value - _min_value) / (_max_value - _min_value) * _height;
            _sprite->fillCircle(_width/2, _height - thumb_y, 8, _thumb_color);
        } else {
            int16_t track_y = _height / 2 - 2;
            _sprite->fillRect(0, track_y, _width, 4, _track_color);
            
            // Draw thumb
            float thumb_x = (_value - _min_value) / (_max_value - _min_value) * _width;
            _sprite->fillCircle(thumb_x, _height/2, 8, _thumb_color);
        }
        
        _sprite->pushSprite(target, _x, _y);
    }

    void handleTouch(int16_t x, int16_t y, bool pressed) override {
        if (!_enabled || !pressed) return;
        
        if (hitTest(x, y)) {
            float new_value;
            if (_vertical) {
                float relative_y = y - _y;
                new_value = _max_value - (relative_y / _height) * (_max_value - _min_value);
            } else {
                float relative_x = x - _x;
                new_value = _min_value + (relative_x / _width) * (_max_value - _min_value);
            }
            
            setValue(new_value);
            if (_callback) _callback(_value);
        }
    }
};
```

## Usage Example

```cpp
class TouchUI : public TouchHandler {
private:
    LGFX* _display;
    LGFX_Sprite* _buffer;
    std::vector<Widget*> _widgets;

public:
    TouchUI(LGFX* display) : TouchHandler(display), _display(display) {
        _buffer = new LGFX_Sprite(_display);
        _buffer->createSprite(_display->width(), _display->height());
        
        // Create UI elements
        auto button = new Button(10, 10, 100, 40, "Click Me");
        button->setCallback([this]() {
            Serial.println("Button clicked!");
        });
        _widgets.push_back(button);
        
        auto slider = new Slider(10, 60, 200, 30);
        slider->setCallback([](float value) {
            Serial.printf("Slider value: %.2f\n", value);
        });
        _widgets.push_back(slider);
    }

    void update() {
        TouchHandler::update();
        
        // Draw all widgets
        _buffer->fillScreen(TFT_BLACK);
        for (auto widget : _widgets) {
            widget->draw(_buffer);
        }
        _buffer->pushSprite(0, 0);
    }

protected:
    void onTouchStart(int16_t x, int16_t y) override {
        for (auto widget : _widgets) {
            widget->handleTouch(x, y, true);
        }
    }

    void onTouchEnd(int16_t x, int16_t y, uint32_t duration) override {
        for (auto widget : _widgets) {
            widget->handleTouch(x, y, false);
        }
    }
};
```

## Best Practices

1. Touch Handling
   - Implement debouncing
   - Handle multi-touch if supported
   - Consider touch pressure
   - Validate touch coordinates

2. UI Components
   - Use consistent styling
   - Implement proper hit testing
   - Handle state changes
   - Provide visual feedback

3. Performance
   - Use sprite buffering
   - Minimize redraws
   - Optimize touch polling
   - Handle memory constraints

4. Error Handling
   - Validate touch input
   - Handle memory allocation failures
   - Implement component bounds checking 