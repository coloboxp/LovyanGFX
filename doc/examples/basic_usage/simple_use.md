# Simple Use Example

This example demonstrates the fundamental usage of LovyanGFX, covering basic initialization, graphics operations, and color handling.

## Overview

The Simple Use example (`examples/HowToUse/1_simple_use/1_simple_use.ino`) shows how to:
- Initialize a display
- Configure basic display parameters
- Perform fundamental drawing operations
- Handle different color formats
- Use sprites for graphics operations

## Hardware Configuration

The example supports multiple display types through automatic detection:
```cpp
#define LGFX_AUTODETECT // Automatic board recognition
```

For specific hardware, you can uncomment the appropriate define:
```cpp
// #define LGFX_M5STACK       // M5Stack Basic / Gray / Go / Fire
// #define LGFX_M5STACK_CORE2 // M5Stack Core2
// #define LGFX_M5STICK_C     // M5Stack M5Stick C / CPlus
```

## Basic Setup

1. Initialize the display:
```cpp
#include <LovyanGFX.hpp>
#include <LGFX_AUTODETECT.hpp>

static LGFX lcd;                 // Create LGFX instance
static LGFX_Sprite sprite(&lcd); // Create sprite instance if needed

void setup() {
    lcd.init();                  // Initialize the display
    lcd.setRotation(1);         // Set rotation (0-3)
    lcd.setBrightness(128);     // Set brightness (0-255)
    lcd.setColorDepth(24);      // Set color depth (16 or 24)
}
```

## Drawing Operations

### Basic Shapes
```cpp
// Point
lcd.drawPixel(0, 0, 0xFFFF);  // Draw white pixel at 0,0

// Lines
lcd.drawFastVLine(2, 0, 100, lcd.color888(255, 0, 0));   // Red vertical line
lcd.drawFastHLine(0, 2, 100, 0xFF0000U);                 // Red horizontal line
lcd.drawLine(0, 1, 39, 40, red);                         // Diagonal line

// Rectangles
lcd.drawRect(10, 10, 50, 50, 0xF800);    // Rectangle outline
lcd.fillRect(20, 20, 20, 20, 0xE0);      // Filled rectangle

// Circles
lcd.drawCircle(40, 80, 20, blue);        // Circle outline
lcd.fillCircle(40, 80, 20, red);         // Filled circle

// Complex Shapes
lcd.drawTriangle(80, 80, 60, 80, 80, 60, blue);     // Triangle outline
lcd.fillTriangle(80, 80, 60, 80, 80, 60, red);      // Filled triangle
lcd.drawBezier(60, 80, 80, 80, 80, 60, green);      // Bezier curve
```

## Color Handling

### Color Formats
1. 24-bit RGB888 (uint32_t):
```cpp
uint32_t red = 0xFF0000;                     // Red color
lcd.drawPixel(x, y, lcd.color888(255,0,0));  // Using color888 function
```

2. 16-bit RGB565 (uint16_t):
```cpp
uint16_t green = 0x07E0;                     // Green color
lcd.drawPixel(x, y, lcd.color565(0,255,0));  // Using color565 function
```

3. 8-bit RGB332 (uint8_t):
```cpp
uint8_t blue = 0x03;                         // Blue color
lcd.drawPixel(x, y, lcd.color332(0,0,255));  // Using color332 function
```

### Color Management
```cpp
// Set default color
lcd.setColor(0xFF0000U);    // Set red as current color

// Set background color
lcd.setBaseColor(0x000000U); // Set black as background color
lcd.clear();                 // Clear screen with background color
```

## Performance Optimization

### SPI Bus Management
```cpp
// Automatic SPI bus handling
lcd.drawLine(0, 1, 39, 40, red);   // Each call manages SPI bus

// Manual SPI bus optimization
lcd.startWrite();                   // Lock SPI bus
lcd.drawLine(38, 0, 0, 38, yellow);
lcd.drawLine(39, 1, 1, 39, magenta);
lcd.drawLine(40, 2, 2, 40, cyan);
lcd.endWrite();                     // Release SPI bus
```

### Fast Pixel Writing
```cpp
lcd.startWrite();
for (uint32_t x = 0; x < 128; ++x) {
    for (uint32_t y = 0; y < 128; ++y) {
        lcd.writePixel(x, y, lcd.color888(x*2, x+y, y*2));
    }
}
lcd.endWrite();
```

## Best Practices

1. Color Depth Selection
   - Use 16-bit for better performance
   - Use 24-bit for better color quality
   ```cpp
   lcd.setColorDepth(16);  // RGB565 for performance
   lcd.setColorDepth(24);  // RGB888 for quality
   ```

2. SPI Bus Optimization
   - Use startWrite/endWrite for multiple operations
   - Manage transaction counts carefully
   ```cpp
   lcd.startWrite();     // Start transaction
   // Multiple drawing operations
   lcd.endWrite();      // End transaction
   ```

3. Memory Management
   - Use appropriate color depth
   - Release sprites when not needed
   - Consider available RAM when creating sprites

## Common Issues

1. Display Orientation
   - Check rotation setting if display is upside down
   - Use values 0-3 for normal orientation
   - Use values 4-7 for mirrored orientation

2. Color Issues
   - Verify color depth setting
   - Check RGB/BGR order for your display
   - Use appropriate color format functions

3. Performance
   - Use 16-bit color for faster operation
   - Batch operations between startWrite/endWrite
   - Use writePixel instead of drawPixel in loops

## Additional Resources

- [Color Handling Guide](../core_concepts/color_handling.md)
- [Display Configuration](../core_concepts/display_config.md)
- [Performance Optimization](../core_concepts/performance.md)
- [Hardware Compatibility](../hardware_compatibility.md) 