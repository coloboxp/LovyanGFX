# Complex UI Implementation Guide

This guide explains how to create sophisticated user interfaces in LovyanGFX using screen management, layouts, and widgets.

## Screen Management System

The screen management system handles multiple screens and transitions between them:

```cpp
class Screen {
protected:
    LGFX* display;              // Display reference
    WidgetContainer widgets;    // Container for screen widgets
    bool needs_redraw;          // Screen redraw flag
    
public:
    Screen(LGFX* disp) : display(disp), needs_redraw(true) {}
    virtual ~Screen() {}
    
    // Called when screen becomes active
    virtual void enter() { 
        needs_redraw = true;  // Force full redraw on entry
    }
    
    // Called when screen becomes inactive
    virtual void exit() {}
    
    // Update screen logic and widgets
    virtual void update() {
        widgets.update();  // Update all child widgets
    }
    
    // Handle screen drawing
    virtual void draw() {
        if (needs_redraw) {
            // Full screen redraw
            display->fillScreen(TFT_BLACK);
            needs_redraw = false;
        }
        widgets.draw();  // Draw all child widgets
    }
    
    // Handle touch input
    virtual void handleTouch(int16_t x, int16_t y, bool pressed) {
        widgets.handleTouch(x, y, pressed);  // Delegate to widgets
    }
};
```

### Screen Manager

Manages screen stack and transitions:

```cpp
class ScreenManager {
    std::vector<Screen*> screen_stack;  // Stack of active screens
    
public:
    // Push new screen onto stack
    void pushScreen(Screen* screen) {
        if (!screen_stack.empty()) {
            screen_stack.back()->exit();  // Exit current screen
        }
        screen_stack.push_back(screen);
        screen->enter();  // Initialize new screen
    }
    
    // Pop current screen from stack
    void popScreen() {
        if (!screen_stack.empty()) {
            screen_stack.back()->exit();
            screen_stack.pop_back();
            if (!screen_stack.empty()) {
                screen_stack.back()->enter();  // Reactivate previous screen
            }
        }
    }
    
    // Update current screen
    void update() {
        if (!screen_stack.empty()) {
            screen_stack.back()->update();
        }
    }
    
    // Draw current screen
    void draw() {
        if (!screen_stack.empty()) {
            screen_stack.back()->draw();
        }
    }
    
    // Handle touch input
    void handleTouch(int16_t x, int16_t y, bool pressed) {
        if (!screen_stack.empty()) {
            screen_stack.back()->handleTouch(x, y, pressed);
        }
    }
};
```

## Layout Management

Layouts handle widget positioning and organization:

```cpp
class Layout {
protected:
    int16_t x, y;              // Layout position
    uint16_t w, h;             // Layout dimensions
    std::vector<Widget*> widgets;  // Child widgets
    
public:
    Layout(int16_t x, int16_t y, uint16_t w, uint16_t h)
        : x(x), y(y), w(w), h(h) {}
    
    // Add widget to layout
    virtual void addWidget(Widget* widget) {
        widgets.push_back(widget);
        updateLayout();  // Recalculate positions
    }
    
    // Update widget positions - implemented by derived classes
    virtual void updateLayout() = 0;
};

// Vertical layout - arranges widgets top to bottom
class VerticalLayout : public Layout {
    uint16_t spacing;  // Space between widgets
    
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

// Grid layout - arranges widgets in rows and columns
class GridLayout : public Layout {
    uint16_t cols;           // Number of columns
    uint16_t row_height;     // Height of each row
    uint16_t spacing;        // Space between cells
    
public:
    GridLayout(int16_t x, int16_t y, uint16_t w, uint16_t h,
               uint16_t cols, uint16_t row_height, uint16_t spacing = 5)
        : Layout(x, y, w, h)
        , cols(cols)
        , row_height(row_height)
        , spacing(spacing) {}
    
    void updateLayout() override {
        uint16_t col_width = (w - (spacing * (cols - 1))) / cols;
        
        for (size_t i = 0; i < widgets.size(); i++) {
            uint16_t col = i % cols;
            uint16_t row = i / cols;
            
            int16_t widget_x = x + col * (col_width + spacing);
            int16_t widget_y = y + row * (row_height + spacing);
            
            widgets[i]->setPosition(widget_x, widget_y);
            widgets[i]->setSize(col_width, row_height);
        }
    }
};
```

## Usage Example

Here's how to create a complex UI with screens and layouts:

```cpp
// Create main menu screen
class MainMenuScreen : public Screen {
    VerticalLayout* layout;
    
public:
    MainMenuScreen(LGFX* display) : Screen(display) {
        // Create centered vertical layout
        layout = new VerticalLayout(
            50, 50,                    // Position
            display->width() - 100,    // Width
            display->height() - 100,   // Height
            10                         // Spacing
        );
        
        // Add menu buttons
        auto startBtn = new Button("Start");
        auto settingsBtn = new Button("Settings");
        auto helpBtn = new Button("Help");
        
        layout->addWidget(startBtn);
        layout->addWidget(settingsBtn);
        layout->addWidget(helpBtn);
        
        // Set button callbacks
        startBtn->setCallback([this]() {
            // Push game screen
            screenManager->pushScreen(new GameScreen(display));
        });
    }
    
    void draw() override {
        if (needs_redraw) {
            display->fillScreen(TFT_NAVY);
            display->setTextColor(TFT_WHITE);
            display->drawString("Main Menu", 10, 10);
            needs_redraw = false;
        }
        layout->draw();
    }
};
```

## Best Practices

1. **Screen Management**
   - Keep screen stack memory usage in mind
   - Clean up resources in screen exit()
   - Handle transitions smoothly
   - Maintain screen state properly

2. **Layout Design**
   - Use appropriate layouts for content
   - Consider screen orientation changes
   - Handle dynamic content updates
   - Implement proper spacing

3. **Widget Organization**
   - Group related widgets logically
   - Use consistent styling
   - Handle focus management
   - Implement proper event bubbling

4. **Performance**
   - Minimize full screen redraws
   - Use partial updates when possible
   - Batch drawing operations
   - Optimize touch handling 