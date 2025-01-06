# Basic Usage Examples

This guide covers the fundamental operations and features of LovyanGFX through practical examples.

## Display Initialization

Basic setup for display initialization:

```cpp
#include <LovyanGFX.hpp>
#include <LGFX_AUTODETECT.hpp>  // Automatic board detection

static LGFX lcd;                 // Create LGFX instance
static LGFX_Sprite sprite(&lcd); // Create sprite instance (optional)

void setup() {
    lcd.init();                  // Initialize the display
    lcd.setRotation(1);         // Set rotation (0-3)
    lcd.setBrightness(128);     // Set brightness (0-255)
    lcd.setColorDepth(24);      // Set color depth (16 or 24)
}
```

## Basic Drawing Operations

### Drawing Primitives

```cpp
// Points and Lines
lcd.drawPixel(x, y, color);              // Draw a single pixel
lcd.drawLine(x0, y0, x1, y1, color);     // Draw a line
lcd.drawFastVLine(x, y, height, color);  // Draw a vertical line
lcd.drawFastHLine(x, y, width, color);   // Draw a horizontal line

// Rectangles
lcd.drawRect(x, y, w, h, color);         // Draw rectangle outline
lcd.fillRect(x, y, w, h, color);         // Draw filled rectangle
lcd.drawRoundRect(x, y, w, h, r, color); // Draw rounded rectangle
lcd.fillRoundRect(x, y, w, h, r, color); // Draw filled rounded rectangle

// Circles and Ellipses
lcd.drawCircle(x, y, radius, color);     // Draw circle outline
lcd.fillCircle(x, y, radius, color);     // Draw filled circle
lcd.drawEllipse(x, y, rx, ry, color);    // Draw ellipse outline
lcd.fillEllipse(x, y, rx, ry, color);    // Draw filled ellipse

// Triangles
lcd.drawTriangle(x0, y0, x1, y1, x2, y2, color); // Draw triangle outline
lcd.fillTriangle(x0, y0, x1, y1, x2, y2, color); // Draw filled triangle

// Curves
lcd.drawBezier(x0, y0, x1, y1, x2, y2, color);   // Draw quadratic bezier curve
lcd.drawBezier(x0, y0, x1, y1, x2, y2, x3, y3, color); // Draw cubic bezier curve

// Arcs
lcd.drawArc(x, y, r0, r1, angle0, angle1, color); // Draw arc outline
lcd.fillArc(x, y, r0, r1, angle0, angle1, color); // Draw filled arc
```

## Color Management

### Color Formats

```cpp
// RGB888 (24-bit) color format
uint32_t color = lcd.color888(red, green, blue);  // Each component 0-255

// RGB565 (16-bit) color format
uint16_t color = lcd.color565(red, green, blue);  // Each component 0-255

// RGB332 (8-bit) color format
uint8_t color = lcd.color332(red, green, blue);   // Each component 0-255
```

### Color Constants

```cpp
// Common color definitions
#define TFT_BLACK       0x0000      // 0,   0,   0
#define TFT_NAVY        0x000F      // 0,   0, 128
#define TFT_DARKGREEN   0x03E0      // 0, 128,   0
#define TFT_DARKCYAN    0x03EF      // 0, 128, 128
#define TFT_MAROON      0x7800      // 128,   0,   0
#define TFT_PURPLE      0x780F      // 128,   0, 128
#define TFT_OLIVE       0x7BE0      // 128, 128,   0
#define TFT_LIGHTGREY   0xC618      // 192, 192, 192
#define TFT_DARKGREY    0x7BEF      // 128, 128, 128
#define TFT_BLUE        0x001F      // 0,   0, 255
#define TFT_GREEN       0x07E0      // 0, 255,   0
#define TFT_CYAN        0x07FF      // 0, 255, 255
#define TFT_RED         0xF800      // 255,   0,   0
#define TFT_MAGENTA     0xF81F      // 255,   0, 255
#define TFT_YELLOW      0xFFE0      // 255, 255,   0
#define TFT_WHITE       0xFFFF      // 255, 255, 255
```

## Performance Optimization

### Using startWrite/endWrite

```cpp
// Without optimization
lcd.drawLine(0, 0, 100, 100, TFT_RED);    // Each call acquires/releases SPI bus
lcd.drawLine(0, 100, 100, 0, TFT_BLUE);

// With optimization
lcd.startWrite();                          // Acquire SPI bus once
lcd.drawLine(0, 0, 100, 100, TFT_RED);    // Multiple drawing operations
lcd.drawLine(0, 100, 100, 0, TFT_BLUE);
lcd.endWrite();                            // Release SPI bus once
```

### Fast Pixel Operations

```cpp
lcd.startWrite();
for (uint32_t x = 0; x < width; ++x) {
    for (uint32_t y = 0; y < height; ++y) {
        lcd.writePixel(x, y, color);       // Fast pixel write without bus checks
    }
}
lcd.endWrite();
```

## Screen Management

### Screen Clearing

```cpp
// Fill screen methods
lcd.fillScreen(TFT_BLACK);     // Fill with color (uses drawing color)
lcd.clear(TFT_BLACK);          // Fill with color (uses background color)
lcd.clear();                   // Fill with current background color

// Set background color
lcd.setBaseColor(TFT_BLACK);   // Set background color for clear()
```

### Display Control

```cpp
// Display rotation
lcd.setRotation(0);           // 0° rotation
lcd.setRotation(1);           // 90° rotation
lcd.setRotation(2);           // 180° rotation
lcd.setRotation(3);           // 270° rotation

// Display inversion
lcd.invertDisplay(true);      // Invert display colors
lcd.invertDisplay(false);     // Normal display colors

// Display on/off
lcd.displayOn();             // Turn display on
lcd.displayOff();            // Turn display off
```

## Error Handling

```cpp
void setup() {
    if (!lcd.init()) {
        // Handle initialization failure
        Serial.println("Display initialization failed!");
        while (1) { delay(100); }
    }
    
    // Check color depth support
    if (!lcd.setColorDepth(24)) {
        // Fall back to 16-bit color
        lcd.setColorDepth(16);
    }
}
```

## Best Practices

1. **Initialization**
   - Always check initialization success
   - Set appropriate rotation for your application
   - Configure color depth based on needs

2. **Performance**
   - Use `startWrite`/`endWrite` for multiple drawing operations
   - Use appropriate color depth (16-bit is faster)
   - Batch similar drawing operations together

3. **Memory Usage**
   - Consider available RAM when creating sprites
   - Use appropriate color depth for your needs
   - Clear unused sprites to free memory

4. **Error Handling**
   - Check initialization results
   - Validate parameters before drawing
   - Implement appropriate error recovery

## Common Issues and Solutions

1. **Display Not Working**
   ```cpp
   // Check initialization
   if (!lcd.init()) {
       Serial.println("Init failed! Check connections.");
       return;
   }
   ```

2. **Incorrect Colors**
   ```cpp
   // Ensure correct color depth
   lcd.setColorDepth(16);  // Try 16-bit if 24-bit has issues
   ```

3. **Slow Performance**
   ```cpp
   // Optimize multiple drawing operations
   lcd.startWrite();
   // Multiple drawing operations here
   lcd.endWrite();
   ```