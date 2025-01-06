# Complex UI Implementation Guide

This guide explains how to create complex user interfaces in LovyanGFX.

## UI Framework

### Screen Management

```cpp
class Screen {
protected:
    LGFX* display;
    WidgetContainer widgets;
    bool needs_redraw;
    
public:
    Screen(LGFX* disp) : display(disp), needs_redraw(true) {}
    virtual ~Screen() {}
    
    virtual void enter() { needs_redraw = true; }
    virtual void exit() {}
    virtual void update() {
        widgets.update();
    }
    virtual void draw() {
        if (needs_redraw) {
            display->fillScreen(TFT_BLACK);
            needs_redraw = false;
        }
        widgets.draw();
    }
    virtual void handleTouch(int16_t x, int16_t y, bool pressed) {
        widgets.handleTouch(x, y, pressed);
    }
};

class ScreenManager {
    std::vector<Screen*> screen_stack;
    
public:
    void pushScreen(Screen* screen) {
        if (!screen_stack.empty()) {
            screen_stack.back()->exit();
        }
        screen_stack.push_back(screen);
        screen->enter();
    }
    
    void popScreen() {
        if (!screen_stack.empty()) {
            screen_stack.back()->exit();
            screen_stack.pop_back();
            if (!screen_stack.empty()) {
                screen_stack.back()->enter();
            }
        }
    }
    
    void update() {
        if (!screen_stack.empty()) {
            screen_stack.back()->update();
        }
    }
    
    void draw() {
        if (!screen_stack.empty()) {
            screen_stack.back()->draw();
        }
    }
    
    void handleTouch(int16_t x, int16_t y, bool pressed) {
        if (!screen_stack.empty()) {
            screen_stack.back()->handleTouch(x, y, pressed);
        }
    }
};
```

### Layout Management

```cpp
class Layout {
protected:
    int16_t x, y;
    uint16_t w, h;
    std::vector<Widget*> widgets;
    
public:
    Layout(int16_t x, int16_t y, uint16_t w, uint16_t h)
        : x(x), y(y), w(w), h(h) {}
    
    virtual void addWidget(Widget* widget) {
        widgets.push_back(widget);
        updateLayout();
    }
    
    virtual void updateLayout() = 0;
};

class VerticalLayout : public Layout {
    uint16_t spacing;
    
public:
    VerticalLayout(int16_t x, int16_t y, uint16_t w, uint16_t h, uint16_t spacing = 5)
        : Layout(x, y, w, h), spacing(spacing) {}
    
    void updateLayout() override {
        int16_t current_y = y;
        for (auto widget : widgets) {
            widget->setPosition(x, current_y);
            widget->setSize(w, widget->getHeight());
            current_y += widget->getHeight() + spacing;
        }
    }
};

class GridLayout : public Layout {
    uint16_t cols;
    uint16_t row_height;
    uint16_t spacing;
    
public:
    GridLayout(int16_t x, int16_t y, uint16_t w, uint16_t h,
               uint16_t cols, uint16_t row_height, uint16_t spacing = 5)
        : Layout(x, y, w, h)
        , cols(cols)
        , row_height(row_height)
        , spacing(spacing) {}
    
    void updateLayout() override {
        uint16_t col_width = (w - (cols - 1) * spacing) / cols;
        size_t index = 0;
        
        for (auto widget : widgets) {
            uint16_t row = index / cols;
            uint16_t col = index % cols;
            
            int16_t widget_x = x + col * (col_width + spacing);
            int16_t widget_y = y + row * (row_height + spacing);
            
            widget->setPosition(widget_x, widget_y);
            widget->setSize(col_width, row_height);
            
            index++;
        }
    }
};
```

## Example Screens

### Main Menu Screen

```cpp
class MainMenuScreen : public Screen {
    Button* btn_settings;
    Button* btn_tools;
    Button* btn_help;
    
public:
    MainMenuScreen(LGFX* disp) : Screen(disp) {
        // Create vertical layout
        VerticalLayout* layout = new VerticalLayout(10, 10, 220, 300);
        
        // Create menu buttons
        btn_settings = new Button(display, "Settings", 0, 0, 0, 40);
        btn_settings->setCallback([this]() {
            // Push settings screen
            screenManager->pushScreen(new SettingsScreen(display));
        });
        
        btn_tools = new Button(display, "Tools", 0, 0, 0, 40);
        btn_tools->setCallback([this]() {
            screenManager->pushScreen(new ToolsScreen(display));
        });
        
        btn_help = new Button(display, "Help", 0, 0, 0, 40);
        btn_help->setCallback([this]() {
            screenManager->pushScreen(new HelpScreen(display));
        });
        
        // Add buttons to layout
        layout->addWidget(btn_settings);
        layout->addWidget(btn_tools);
        layout->addWidget(btn_help);
        
        // Add buttons to widget container
        widgets.addWidget(btn_settings);
        widgets.addWidget(btn_tools);
        widgets.addWidget(btn_help);
    }
    
    void draw() override {
        if (needs_redraw) {
            display->fillScreen(TFT_NAVY);
            display->setTextColor(TFT_WHITE);
            display->setTextDatum(top_center);
            display->drawString("Main Menu", display->width()/2, 20);
            needs_redraw = false;
        }
        widgets.draw();
    }
};
```

### Settings Screen

```cpp
class SettingsScreen : public Screen {
    Slider* brightness_slider;
    Slider* volume_slider;
    Button* btn_back;
    
public:
    SettingsScreen(LGFX* disp) : Screen(disp) {
        // Create grid layout
        GridLayout* layout = new GridLayout(10, 60, 220, 200, 1, 40);
        
        // Create settings controls
        brightness_slider = new Slider(display, 0, 0, 0, 0);
        brightness_slider->setRange(0, 100);
        brightness_slider->setValue(50);
        brightness_slider->setCallback([](float value) {
            // Update brightness
            analogWrite(TFT_BL, value * 2.55f);
        });
        
        volume_slider = new Slider(display, 0, 0, 0, 0);
        volume_slider->setRange(0, 100);
        volume_slider->setValue(75);
        volume_slider->setCallback([](float value) {
            // Update volume
        });
        
        btn_back = new Button(display, "Back", 10, 280, 100, 30);
        btn_back->setCallback([this]() {
            screenManager->popScreen();
        });
        
        // Add controls to layout
        layout->addWidget(brightness_slider);
        layout->addWidget(volume_slider);
        
        // Add controls to widget container
        widgets.addWidget(brightness_slider);
        widgets.addWidget(volume_slider);
        widgets.addWidget(btn_back);
    }
    
    void draw() override {
        if (needs_redraw) {
            display->fillScreen(TFT_DARKGREY);
            display->setTextColor(TFT_WHITE);
            display->setTextDatum(top_center);
            display->drawString("Settings", display->width()/2, 20);
            
            // Draw labels
            display->setTextDatum(middle_left);
            display->drawString("Brightness", 10, 80);
            display->drawString("Volume", 10, 120);
            
            needs_redraw = false;
        }
        widgets.draw();
    }
};
```

### Dialog Screen

```cpp
class DialogScreen : public Screen {
    const char* message;
    Button* btn_ok;
    Button* btn_cancel;
    std::function<void(bool)> callback;
    
public:
    DialogScreen(LGFX* disp, const char* msg, std::function<void(bool)> cb)
        : Screen(disp), message(msg), callback(cb) {
        // Create buttons
        btn_ok = new Button(display, "OK", 40, 200, 80, 30);
        btn_ok->setCallback([this]() {
            if (callback) callback(true);
            screenManager->popScreen();
        });
        
        btn_cancel = new Button(display, "Cancel", 140, 200, 80, 30);
        btn_cancel->setCallback([this]() {
            if (callback) callback(false);
            screenManager->popScreen();
        });
        
        // Add buttons to widget container
        widgets.addWidget(btn_ok);
        widgets.addWidget(btn_cancel);
    }
    
    void draw() override {
        if (needs_redraw) {
            // Draw dialog background
            display->fillRoundRect(20, 60, 200, 180, 10, TFT_WHITE);
            display->drawRoundRect(20, 60, 200, 180, 10, TFT_DARKGREY);
            
            // Draw message
            display->setTextColor(TFT_BLACK);
            display->setTextDatum(middle_center);
            display->drawString(message, 120, 130);
            
            needs_redraw = false;
        }
        widgets.draw();
    }
};
```

## Advanced Features

### Animation Manager

```cpp
class Animation {
protected:
    uint32_t start_time;
    uint32_t duration;
    bool completed;
    
public:
    Animation(uint32_t duration)
        : start_time(0), duration(duration), completed(false) {}
    
    virtual void start() {
        start_time = millis();
        completed = false;
    }
    
    virtual void update() {
        if (completed) return;
        
        uint32_t elapsed = millis() - start_time;
        float progress = min(1.0f, (float)elapsed / duration);
        
        updateAnimation(progress);
        
        if (progress >= 1.0f) {
            completed = true;
        }
    }
    
    virtual void updateAnimation(float progress) = 0;
    bool isCompleted() const { return completed; }
};

class AnimationManager {
    std::vector<Animation*> animations;
    
public:
    void addAnimation(Animation* animation) {
        animation->start();
        animations.push_back(animation);
    }
    
    void update() {
        for (auto it = animations.begin(); it != animations.end();) {
            (*it)->update();
            if ((*it)->isCompleted()) {
                delete *it;
                it = animations.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

### Theme Manager

```cpp
struct Theme {
    uint16_t background_color;
    uint16_t text_color;
    uint16_t primary_color;
    uint16_t secondary_color;
    uint16_t accent_color;
    uint16_t error_color;
    
    const GFXfont* title_font;
    const GFXfont* body_font;
};

class ThemeManager {
    Theme current_theme;
    
public:
    void setTheme(const Theme& theme) {
        current_theme = theme;
        // Notify all widgets of theme change
    }
    
    const Theme& getTheme() const {
        return current_theme;
    }
    
    static ThemeManager& getInstance() {
        static ThemeManager instance;
        return instance;
    }
};
```

### Touch Gesture Manager

```cpp
class GestureManager {
    int16_t start_x, start_y;
    uint32_t start_time;
    bool tracking;
    
public:
    void update(int16_t x, int16_t y, bool pressed) {
        if (pressed && !tracking) {
            // Start gesture
            start_x = x;
            start_y = y;
            start_time = millis();
            tracking = true;
        } else if (!pressed && tracking) {
            // End gesture
            int16_t dx = x - start_x;
            int16_t dy = y - start_y;
            uint32_t duration = millis() - start_time;
            
            // Detect gesture type
            if (duration < 200) {
                // Tap
                handleTap(x, y);
            } else if (abs(dx) > abs(dy)) {
                // Horizontal swipe
                if (dx > 50) {
                    handleSwipe(SwipeDirection::Right);
                } else if (dx < -50) {
                    handleSwipe(SwipeDirection::Left);
                }
            } else {
                // Vertical swipe
                if (dy > 50) {
                    handleSwipe(SwipeDirection::Down);
                } else if (dy < -50) {
                    handleSwipe(SwipeDirection::Up);
                }
            }
            
            tracking = false;
        }
    }
    
protected:
    virtual void handleTap(int16_t x, int16_t y) {}
    virtual void handleSwipe(SwipeDirection direction) {}
};
```

These examples demonstrate how to create complex user interfaces in LovyanGFX. Adapt them according to your specific needs and requirements. 