# Custom Widget Development Guide

This guide explains how to create custom widgets in LovyanGFX.

## Basic Widget Framework

### Widget Base Class

```cpp
class Widget {
protected:
    LGFX* display;
    int16_t x, y;           // Position
    uint16_t w, h;          // Size
    uint16_t fg_color;      // Foreground color
    uint16_t bg_color;      // Background color
    bool is_visible;        // Visibility state
    bool needs_redraw;      // Redraw flag
    
public:
    Widget(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : display(disp)
        , x(x), y(y)
        , w(w), h(h)
        , fg_color(TFT_WHITE)
        , bg_color(TFT_BLACK)
        , is_visible(true)
        , needs_redraw(true)
    {}
    
    virtual ~Widget() {}
    
    // Virtual methods for widget behavior
    virtual void draw() = 0;
    virtual void update() {}
    virtual bool handleTouch(int16_t tx, int16_t ty, bool pressed) { return false; }
    
    // Common widget methods
    void setPosition(int16_t nx, int16_t ny) {
        x = nx;
        y = ny;
        needs_redraw = true;
    }
    
    void setSize(uint16_t nw, uint16_t nh) {
        w = nw;
        h = nh;
        needs_redraw = true;
    }
    
    void setColors(uint16_t fg, uint16_t bg) {
        fg_color = fg;
        bg_color = bg;
        needs_redraw = true;
    }
    
    void setVisible(bool visible) {
        if (is_visible != visible) {
            is_visible = visible;
            needs_redraw = true;
        }
    }
    
    bool isVisible() const { return is_visible; }
    bool needsRedraw() const { return needs_redraw; }
    void clearRedraw() { needs_redraw = false; }
    
    // Hit testing
    bool hitTest(int16_t tx, int16_t ty) const {
        return tx >= x && tx < x + w && ty >= y && ty < y + h;
    }
};
```

## Example Widgets

### Button Widget

```cpp
class Button : public Widget {
    const char* label;
    bool pressed;
    std::function<void()> callback;
    
public:
    Button(LGFX* disp, const char* text, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : Widget(disp, x, y, w, h)
        , label(text)
        , pressed(false)
    {}
    
    void setCallback(std::function<void()> cb) {
        callback = cb;
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Draw button background
        display->fillRoundRect(x, y, w, h, 5, 
            pressed ? fg_color : bg_color);
        
        // Draw button border
        display->drawRoundRect(x, y, w, h, 5, fg_color);
        
        // Draw label
        display->setTextColor(pressed ? bg_color : fg_color);
        display->setTextDatum(middle_center);
        display->drawString(label, x + w/2, y + h/2);
        
        display->endWrite();
        clearRedraw();
    }
    
    bool handleTouch(int16_t tx, int16_t ty, bool touch_pressed) override {
        if (!is_visible) return false;
        
        if (hitTest(tx, ty)) {
            if (touch_pressed != pressed) {
                pressed = touch_pressed;
                needs_redraw = true;
                
                if (!pressed && callback) {
                    callback();
                }
            }
            return true;
        }
        
        if (pressed) {
            pressed = false;
            needs_redraw = true;
        }
        return false;
    }
};
```

### Slider Widget

```cpp
class Slider : public Widget {
    float value;        // Current value (0.0-1.0)
    float min_value;    // Minimum value
    float max_value;    // Maximum value
    bool vertical;      // Orientation
    bool dragging;      // Touch state
    std::function<void(float)> callback;
    
public:
    Slider(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h,
           bool vertical = false)
        : Widget(disp, x, y, w, h)
        , value(0.0f)
        , min_value(0.0f)
        , max_value(1.0f)
        , vertical(vertical)
        , dragging(false)
    {}
    
    void setRange(float min, float max) {
        min_value = min;
        max_value = max;
        needs_redraw = true;
    }
    
    void setValue(float v) {
        value = constrain(v, min_value, max_value);
        needs_redraw = true;
        if (callback) callback(value);
    }
    
    void setCallback(std::function<void(float)> cb) {
        callback = cb;
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Draw background
        display->fillRect(x, y, w, h, bg_color);
        display->drawRect(x, y, w, h, fg_color);
        
        // Calculate thumb position
        float normalized = (value - min_value) / (max_value - min_value);
        if (vertical) {
            int16_t ty = y + h - (h * normalized);
            display->fillRect(x, ty - 4, w, 8, fg_color);
        } else {
            int16_t tx = x + (w * normalized);
            display->fillRect(tx - 4, y, 8, h, fg_color);
        }
        
        display->endWrite();
        clearRedraw();
    }
    
    bool handleTouch(int16_t tx, int16_t ty, bool touch_pressed) override {
        if (!is_visible) return false;
        
        if (touch_pressed && hitTest(tx, ty)) {
            dragging = true;
            updateValue(tx, ty);
            return true;
        }
        
        if (dragging && !touch_pressed) {
            dragging = false;
        }
        
        if (dragging) {
            updateValue(tx, ty);
            return true;
        }
        
        return false;
    }
    
private:
    void updateValue(int16_t tx, int16_t ty) {
        float normalized;
        if (vertical) {
            normalized = 1.0f - constrain((float)(ty - y) / h, 0.0f, 1.0f);
        } else {
            normalized = constrain((float)(tx - x) / w, 0.0f, 1.0f);
        }
        
        float new_value = min_value + normalized * (max_value - min_value);
        if (new_value != value) {
            setValue(new_value);
        }
    }
};
```

### Progress Bar Widget

```cpp
class ProgressBar : public Widget {
    float progress;     // Progress value (0.0-1.0)
    bool show_text;     // Show percentage text
    
public:
    ProgressBar(LGFX* disp, int16_t x, int16_t y, uint16_t w, uint16_t h)
        : Widget(disp, x, y, w, h)
        , progress(0.0f)
        , show_text(true)
    {}
    
    void setProgress(float p) {
        progress = constrain(p, 0.0f, 1.0f);
        needs_redraw = true;
    }
    
    void showText(bool show) {
        show_text = show;
        needs_redraw = true;
    }
    
    void draw() override {
        if (!is_visible) return;
        
        display->startWrite();
        
        // Draw background
        display->fillRect(x, y, w, h, bg_color);
        display->drawRect(x, y, w, h, fg_color);
        
        // Draw progress
        uint16_t pw = w * progress;
        if (pw > 0) {
            display->fillRect(x + 1, y + 1, pw - 2, h - 2, fg_color);
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
        clearRedraw();
    }
};
```

## Widget Container

```cpp
class WidgetContainer {
    std::vector<Widget*> widgets;
    Widget* focused_widget;
    
public:
    void addWidget(Widget* widget) {
        widgets.push_back(widget);
    }
    
    void removeWidget(Widget* widget) {
        auto it = std::find(widgets.begin(), widgets.end(), widget);
        if (it != widgets.end()) {
            widgets.erase(it);
            if (focused_widget == widget) {
                focused_widget = nullptr;
            }
        }
    }
    
    void update() {
        for (auto widget : widgets) {
            if (widget->isVisible()) {
                widget->update();
            }
        }
    }
    
    void draw() {
        for (auto widget : widgets) {
            if (widget->isVisible() && widget->needsRedraw()) {
                widget->draw();
            }
        }
    }
    
    void handleTouch(int16_t x, int16_t y, bool pressed) {
        if (pressed) {
            // Find touched widget
            for (auto it = widgets.rbegin(); it != widgets.rend(); ++it) {
                if ((*it)->isVisible() && (*it)->handleTouch(x, y, true)) {
                    focused_widget = *it;
                    break;
                }
            }
        } else if (focused_widget) {
            // Update focused widget
            focused_widget->handleTouch(x, y, false);
            focused_widget = nullptr;
        }
    }
};
```

## Usage Example

```cpp
// Create display instance
LGFX display;

// Create widget container
WidgetContainer container;

// Create button
Button* button = new Button(&display, "Click Me", 10, 10, 100, 40);
button->setColors(TFT_WHITE, TFT_BLUE);
button->setCallback([]() {
    Serial.println("Button clicked!");
});

// Create slider
Slider* slider = new Slider(&display, 10, 60, 200, 30);
slider->setRange(0, 100);
slider->setCallback([](float value) {
    Serial.printf("Slider value: %.1f\n", value);
});

// Create progress bar
ProgressBar* progress = new ProgressBar(&display, 10, 100, 200, 30);
progress->setColors(TFT_GREEN, TFT_BLACK);

// Add widgets to container
container.addWidget(button);
container.addWidget(slider);
container.addWidget(progress);

void loop() {
    // Update widgets
    container.update();
    
    // Handle touch input
    if (display.getTouch(&tx, &ty)) {
        container.handleTouch(tx, ty, true);
    } else {
        container.handleTouch(0, 0, false);
    }
    
    // Draw widgets
    container.draw();
}
```

These examples demonstrate how to create custom widgets in LovyanGFX. Adapt them according to your specific needs and requirements. 