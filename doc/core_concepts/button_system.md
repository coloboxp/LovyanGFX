# Button System in LovyanGFX

The button system in LovyanGFX provides a flexible way to create and manage interactive buttons in your graphical interface. This document explains the core concepts and usage of the `LGFX_Button` class.

## Button Creation and Initialization

The `LGFX_Button` class offers two initialization methods:

```cpp
// Center-based initialization
void initButton(LovyanGFX *gfx, int16_t x, int16_t y, uint16_t w, uint16_t h,
               color_t outline, color_t fill, color_t textcolor, 
               const char *label, float textsize_x = 1.0f, float textsize_y = 0.0f);

// Upper-left corner based initialization
void initButtonUL(LovyanGFX *gfx, int16_t x, int16_t y, uint16_t w, uint16_t h,
                 color_t outline, color_t fill, color_t textcolor,
                 const char *label, float textsize_x = 1.0f, float textsize_y = 0.0f);
```

The difference between these methods is the reference point:
- `initButton`: Uses the center of the button as the reference point (x,y)
- `initButtonUL`: Uses the upper-left corner as the reference point (x,y)

## Button Properties

Buttons have several customizable properties:

```cpp
// Colors
void setOutlineColor(color_t clr);  // Border color
void setFillColor(color_t clr);     // Background color
void setTextColor(color_t clr);     // Text color

// Label
void setLabelText(const char* label);  // Maximum 11 characters by default
void setLabelDatum(int16_t x_delta, int16_t y_delta, textdatum_t datum = middle_center);
```

## Button States and Interaction

The button system provides methods to handle button states:

```cpp
bool contains(int16_t x, int16_t y);  // Check if point is within button bounds
void press(bool p);                   // Set button press state
bool isPressed();                     // Check if button is currently pressed
bool justPressed();                   // Check if button was just pressed
bool justReleased();                  // Check if button was just released
```

## Drawing and Custom Rendering

Buttons can be drawn with default or custom rendering:

```cpp
// Default drawing
void drawButton(bool inverted = false, const char* long_name = nullptr);

// Custom drawing callback
typedef void (*drawCb)(LovyanGFX *gfx, int32_t x, int32_t y, 
                      int32_t w, int32_t h, bool invert, 
                      const char* long_name);
void setDrawCb(drawCb cb);
```

## Example Usage

```cpp
LGFX_Button myButton;
LGFX display;

void setup() {
    display.init();
    
    // Initialize button at center (160,120) with size 100x40
    myButton.initButton(&display, 160, 120, 100, 40, 
                       TFT_WHITE,  // Outline color
                       TFT_BLUE,   // Fill color
                       TFT_WHITE,  // Text color
                       "Click Me", // Label
                       2);        // Text size
}

void loop() {
    // Draw the button
    myButton.drawButton(false);
    
    // Handle touch input
    if (display.getTouch(&x, &y)) {
        if (myButton.contains(x, y)) {
            myButton.press(true);
            if (myButton.justPressed()) {
                // Handle button press
            }
        } else {
            myButton.press(false);
        }
    }
}
```

## Best Practices

1. **Memory Management**: Button labels are limited to 11 characters by default. Use `long_name` parameter in `drawButton()` for longer text.

2. **Touch Handling**: Always check if a point is within the button bounds before processing touch events.

3. **Custom Drawing**: Use `setDrawCb()` for complex button appearances or animations.

4. **State Management**: Use `justPressed()` and `justReleased()` for one-time actions rather than `isPressed()` for continuous actions.

## Performance Considerations

- Button drawing operations are optimized but should be used judiciously in performance-critical applications.
- Consider using sprite-based buttons for complex UIs with many buttons.
- Custom drawing callbacks should be optimized to maintain good performance. 