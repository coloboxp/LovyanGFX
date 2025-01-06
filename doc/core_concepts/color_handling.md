# Color Handling and Formats Guide

This guide explains color handling, formats, and color space configurations in LovyanGFX.

## Color Types

LovyanGFX supports multiple color formats through type-safe color classes:

```cpp
// Basic color types
using rgb332_t   = uint8_t;    // 8-bit RGB (3:3:2)
using rgb565_t   = uint16_t;   // 16-bit RGB (5:6:5)
using rgb888_t   = uint32_t;   // 24-bit RGB (8:8:8)
using argb8888_t = uint32_t;   // 32-bit ARGB (8:8:8:8)
using bgr888_t   = uint32_t;   // 24-bit BGR (8:8:8)
using grayscale_t = uint8_t;   // 8-bit grayscale
```

## Color Depth Configuration

### Available Color Depths

```cpp
enum color_depth_t {
    palette_1bit = 1,     // 2 colors (monochrome)
    palette_2bit = 2,     // 4 colors
    palette_4bit = 4,     // 16 colors
    palette_8bit = 8,     // 256 colors
    rgb565_2Byte = 16,   // 65,536 colors (RGB565)
    rgb888_3Byte = 24,   // 16.7M colors (RGB888)
    argb8888_4Byte = 32  // 16.7M colors with alpha
};
```

### Setting Color Depth

```cpp
class LGFX : public lgfx::LGFX_Device {
    void init() {
        // Set color depth for display
        setColorDepth(color_depth_t::rgb565_2Byte);
        
        // For sprites
        LGFX_Sprite sprite(&display);
        sprite.setColorDepth(16);  // RGB565
        // or
        sprite.setColorDepth(24);  // RGB888
        // or
        sprite.setColorDepth(32);  // ARGB8888
    }
};
```

## Color Format Handling

### RGB565 (16-bit)

```cpp
// RGB565 format: RRRRRGGGGGGBBBBB
// 5 bits red, 6 bits green, 5 bits blue

// Creating colors
uint16_t color = display.color565(255, 128, 0);  // Orange

// Color components
uint8_t r = (color >> 11) & 0x1F;        // Extract red (5 bits)
uint8_t g = (color >> 5) & 0x3F;         // Extract green (6 bits)
uint8_t b = color & 0x1F;                // Extract blue (5 bits)

// Convert to 8-bit per channel
uint8_t r8 = (r << 3) | (r >> 2);        // 5 bits to 8 bits
uint8_t g8 = (g << 2) | (g >> 4);        // 6 bits to 8 bits
uint8_t b8 = (b << 3) | (b >> 2);        // 5 bits to 8 bits
```

### RGB888 (24-bit)

```cpp
// RGB888 format: RRRRRRRR GGGGGGGG BBBBBBBB
// 8 bits each for red, green, blue

// Creating colors
uint32_t color = display.color888(255, 128, 0);  // Orange

// Color components
uint8_t r = (color >> 16) & 0xFF;  // Extract red
uint8_t g = (color >> 8) & 0xFF;   // Extract green
uint8_t b = color & 0xFF;          // Extract blue

// Convert RGB888 to RGB565
uint16_t color565 = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
```

### ARGB8888 (32-bit)

```cpp
// ARGB8888 format: AAAAAAAA RRRRRRRR GGGGGGGG BBBBBBBB
// 8 bits each for alpha, red, green, blue

// Creating colors with alpha
uint32_t color = display.colorARGB(128, 255, 128, 0);  // Semi-transparent orange

// Color components
uint8_t a = (color >> 24) & 0xFF;  // Extract alpha
uint8_t r = (color >> 16) & 0xFF;  // Extract red
uint8_t g = (color >> 8) & 0xFF;   // Extract green
uint8_t b = color & 0xFF;          // Extract blue
```

## Color Space Configuration

### RGB/BGR Order

Some displays may require BGR color order instead of RGB:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;

    void init_panel() {
        auto cfg = _panel_instance.config();
        
        cfg.rgb_order = false;  // false = RGB order
        // or
        cfg.rgb_order = true;   // true = BGR order
        
        _panel_instance.config(cfg);
    }
};
```

### Color Conversion

```cpp
// Convert between color formats
uint16_t rgb565 = color565(r8, g8, b8);
uint32_t rgb888 = color888(r8, g8, b8);
uint32_t argb8888 = colorARGB(a8, r8, g8, b8);

// Color space conversion utilities
void convertRGB888toRGB565(uint8_t* rgb888, uint16_t* rgb565, size_t len) {
    for (size_t i = 0; i < len; i++) {
        uint8_t r = rgb888[i*3];
        uint8_t g = rgb888[i*3 + 1];
        uint8_t b = rgb888[i*3 + 2];
        rgb565[i] = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
    }
}
```

## Alpha Blending

```cpp
// Alpha blending function
uint32_t blendColors(uint32_t fg, uint32_t bg) {
    uint8_t alpha = (fg >> 24) & 0xFF;
    uint8_t inv_alpha = 255 - alpha;
    
    uint8_t r = ((fg >> 16 & 0xFF) * alpha + (bg >> 16 & 0xFF) * inv_alpha) >> 8;
    uint8_t g = ((fg >> 8 & 0xFF) * alpha + (bg >> 8 & 0xFF) * inv_alpha) >> 8;
    uint8_t b = ((fg & 0xFF) * alpha + (bg & 0xFF) * inv_alpha) >> 8;
    
    return (alpha << 24) | (r << 16) | (g << 8) | b;
}
```

## Best Practices

1. **Color Depth Selection**:
```cpp
// Choose appropriate color depth based on requirements
if (needsTransparency) {
    setColorDepth(32);  // ARGB8888 for alpha support
} else if (memoryConstrained) {
    setColorDepth(16);  // RGB565 for memory efficiency
} else {
    setColorDepth(24);  // RGB888 for best color quality
}
```

2. **Memory Optimization**:
```cpp
// Use RGB565 for memory-constrained devices
sprite.setColorDepth(16);  // Uses 2 bytes per pixel
sprite.createSprite(width, height);

// Calculate memory usage
size_t bytesPerPixel = (sprite.getColorDepth() + 7) >> 3;
size_t totalBytes = width * height * bytesPerPixel;
```

3. **Color Order Handling**:
```cpp
// Check panel documentation for correct color order
auto cfg = _panel_instance.config();
if (isPanelBGR) {
    cfg.rgb_order = true;  // Set BGR order
    _panel_instance.config(cfg);
}
```

4. **Performance Optimization**:
```cpp
// Pre-calculate colors for frequently used values
static constexpr uint16_t CUSTOM_COLOR = color565(255, 128, 0);

// Use direct color values when possible
display.fillRect(x, y, w, h, CUSTOM_COLOR);
```

These implementations provide a foundation for handling colors in LovyanGFX. Adapt the code according to your specific display requirements and memory constraints. 