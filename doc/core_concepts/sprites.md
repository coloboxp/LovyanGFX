# Sprite System

LovyanGFX's sprite system provides off-screen graphics buffer management with hardware acceleration support. Sprites can be used for animation, buffering, and complex graphical operations.

## Basic Sprite Creation

```cpp
#include <LovyanGFX.hpp>

LGFX lcd;
LGFX_Sprite sprite(&lcd);  // Create sprite with parent display

void setup() {
    lcd.init();
    
    // Create sprite with dimensions and color depth
    sprite.createSprite(width, height);
    sprite.setColorDepth(color_depth);  // 1,2,4,8,16,24 bits
}
```

## Memory Management

```cpp
// Create sprite
sprite.createSprite(160, 120);  // Width, Height

// Delete sprite and free memory
sprite.deleteSprite();

// Calculate memory requirements
size_t required_memory = width * height * (bits_per_pixel / 8);

// Check available memory
if (heap_caps_get_free_size(MALLOC_CAP_DMA) > required_memory) {
    sprite.createSprite(width, height);
}
```

## Drawing Operations

Sprites support all standard drawing functions:

```cpp
// Basic drawing
sprite.fillScreen(color);
sprite.drawPixel(x, y, color);
sprite.drawLine(x0, y0, x1, y1, color);
sprite.fillRect(x, y, w, h, color);
sprite.drawCircle(x, y, radius, color);

// Text rendering
sprite.setTextColor(color);
sprite.drawString("Text", x, y);

// Image operations
sprite.drawBitmap(x, y, bitmap_data, w, h, color);
sprite.pushImage(x, y, w, h, data);
```

## Sprite Transformations

Advanced sprite operations:

```cpp
// Basic display
sprite.pushSprite(x, y);  // Copy to parent

// With transparency
sprite.pushSprite(x, y, transparent_color);

// Rotation and scaling
sprite.pushRotateZoom(dst_x, dst_y,
                     angle,      // Rotation angle (0-360)
                     zoom_x,     // X scale factor (1.0 = 100%)
                     zoom_y);    // Y scale factor

// Affine transformation
float matrix[6] = { 1.0, 0.0, 0.0,    // Scale X, Shear X, Move X
                    0.0, 1.0, 0.0 };   // Shear Y, Scale Y, Move Y
sprite.pushAffine(matrix);
```

## Color and Palette Management

```cpp
// Set color depth
sprite.setColorDepth(8);  // 8-bit indexed color

// Configure palette
for (int i = 0; i < 256; i++) {
    sprite.setPaletteColor(i, color);  // Set palette entry
}

// Color operations
sprite.setColorSource(sprite2);  // Use another sprite's palette
sprite.setPivot(px, py);        // Set rotation center
```

## Hardware Acceleration

Optimize performance with hardware features:

```cpp
// Enable DMA transfers
sprite.enableDMA();

// DMA operations
sprite.pushSpriteDMA(x, y);
sprite.pushRotateZoomDMA(x, y, angle, zoom);

// Double buffering
LGFX_Sprite sprites[2](&lcd);
bool flip = false;

void setup() {
    for (int i = 0; i < 2; i++) {
        sprites[i].createSprite(lcd.width(), lcd.height());
    }
}

void loop() {
    sprites[flip].clear();
    // Draw to current buffer
    sprites[flip].pushSpriteDMA(0, 0);
    flip = !flip;  // Swap buffers
}
```

## Performance Tips

1. Memory Optimization:
```cpp
// Use appropriate color depth
sprite.setColorDepth(1);   // Monochrome for simple graphics
sprite.setColorDepth(16);  // RGB565 for complex images
```

2. Batch Operations:
```cpp
sprite.startWrite();
// Multiple drawing operations...
sprite.endWrite();
```

3. Buffer Management:
```cpp
// Reuse sprites instead of create/delete
sprite.clear();  // Clear for reuse
```

## Example: Animated Sprite

```cpp
LGFX lcd;
LGFX_Sprite sprite(&lcd);
LGFX_Sprite animation[4](&sprite);  // Animation frames

void setup() {
    lcd.init();
    sprite.createSprite(32, 32);
    
    // Create animation frames
    for (int i = 0; i < 4; i++) {
        animation[i].createSprite(32, 32);
        animation[i].setColorDepth(16);
        // Draw frame content...
    }
}

void loop() {
    static int frame = 0;
    static int x = 0;
    
    lcd.startWrite();
    animation[frame].pushSprite(&sprite, 0, 0);
    sprite.pushSprite(x, 0);
    lcd.endWrite();
    
    frame = (frame + 1) % 4;
    x = (x + 1) % lcd.width();
    delay(50);
}
``` 