# Sprite System Architecture

The LovyanGFX sprite system provides a powerful off-screen graphics buffer management system with hardware acceleration support. This guide covers the architecture, implementation, and best practices for using sprites.

## Architecture Overview

The sprite system consists of several key components:

```cpp
namespace lgfx {
    class LGFX_Sprite;         // Main sprite class
    class Panel_Sprite;        // Sprite-specific panel implementation
    class SpriteBuffer;        // Buffer management
    struct pixelcopy_t;        // Pixel transfer operations
}
```

### Core Components

1. LGFX_Sprite:
   - Inherits from LovyanGFX
   - Manages sprite creation, deletion, and rendering
   - Handles color depth and palette management
   - Provides drawing operations

2. Panel_Sprite:
   - Implements IPanel interface for sprite-specific operations
   - Manages memory buffer and dimensions
   - Handles pixel access and drawing operations

3. SpriteBuffer:
   - Manages memory allocation and deallocation
   - Supports different allocation sources (Normal, DMA, PSRAM)
   - Handles buffer operations and transfers

## Memory Management

### Buffer Allocation

```cpp
class LGFX_Sprite : public LovyanGFX {
    void* createSprite(int32_t w, int32_t h) {
        // Create sprite buffer
        _img = _panel_sprite.createSprite(w, h, &_write_conv, _psram);
        
        // Initialize dimensions
        _sw = width();
        _sh = height();
        _clip_r = _sw - 1;
        _clip_b = _sh - 1;
        
        return _img;
    }
    
    void deleteSprite(void) {
        _clip_l = 0;
        _clip_t = 0;
        _clip_r = -1;
        _clip_b = -1;
        
        _panel_sprite.deleteSprite();
        _img = nullptr;
    }
};
```

### Memory Sources

```cpp
enum AllocationSource {
    Normal,    // Standard RAM
    Dma,       // DMA-capable memory
    Psram,     // External PSRAM
    Preallocated  // User-provided buffer
};

// Configure memory source
sprite.setPsram(true);  // Use PSRAM if available
```

## Color Management

### Color Depth Configuration

```cpp
// Set color depth during creation
sprite.setColorDepth(color_depth_t depth);  // 1,2,4,8,16,24 bits

// Color depth options
enum color_depth_t {
    rgb332_1Byte,    // 8-bit RGB
    rgb565_2Byte,    // 16-bit RGB
    rgb888_3Byte,    // 24-bit RGB
    argb8888_4Byte,  // 32-bit ARGB
};
```

### Palette Support

```cpp
// Create and configure palette
bool createPalette(void) {
    // Initialize palette
    if (!create_palette()) return false;
    
    // Set default grayscale palette
    setPaletteGrayscale();
    return true;
}

// Custom palette configuration
bool createPalette(const uint16_t* colors, uint32_t count);
```

## Drawing Operations

### Basic Drawing

```cpp
// Direct pixel operations
void drawPixel(int32_t x, int32_t y, uint32_t color);
void fillRect(int32_t x, int32_t y, int32_t w, int32_t h, uint32_t color);
void drawFastVLine(int32_t x, int32_t y, int32_t h, uint32_t color);
void drawFastHLine(int32_t x, int32_t y, int32_t w, uint32_t color);

// Complex shapes
void drawCircle(int32_t x, int32_t y, int32_t r, uint32_t color);
void fillCircle(int32_t x, int32_t y, int32_t r, uint32_t color);
void drawTriangle(int32_t x0, int32_t y0, int32_t x1, int32_t y1, int32_t x2, int32_t y2, uint32_t color);
```

### Sprite Operations

```cpp
// Push sprite to display or another sprite
void pushSprite(int32_t x, int32_t y);
void pushSprite(LovyanGFX* dst, int32_t x, int32_t y);

// Push sprite with transparency
void pushSprite(int32_t x, int32_t y, uint32_t transp);
```

## Advanced Features

### Rotation and Transformation

```cpp
// Set rotation (0-7)
void setRotation(uint8_t r);

// Set pivot point for transformations
void setPivot(float x, float y);

// Rotate around pivot
void rotateSprite(float angle);
```

### Transparency and Blending

```cpp
// Set transparency color
sprite.setTransparentColor(uint32_t color);

// Enable alpha blending
sprite.setColorDepth(lgfx::color_depth_t::argb8888_4Byte);
sprite.fillSprite(color_with_alpha);
```

### Animation Support

```cpp
// Create animation frames
LGFX_Sprite frames[4](&lcd);
for (int i = 0; i < 4; i++) {
    frames[i].createSprite(width, height);
    frames[i].setColorDepth(16);
    // Draw frame content
}

// Display animation
void showAnimation() {
    static int frame = 0;
    frames[frame].pushSprite(x, y);
    frame = (frame + 1) % 4;
}
```

## Best Practices

1. Memory Management:
   - Use appropriate color depth for your needs
   - Consider PSRAM for large sprites
   - Delete unused sprites to free memory
   - Reuse sprites instead of create/delete cycles

2. Performance Optimization:
   - Use hardware-accelerated operations when available
   - Minimize sprite creation/deletion
   - Use appropriate color depth
   - Implement double buffering for smooth animations

3. Resource Usage:
   - Monitor memory usage
   - Use sprite pooling for animations
   - Implement proper cleanup
   - Handle memory allocation failures

4. Error Handling:
   - Check sprite creation success
   - Validate memory availability
   - Handle allocation failures gracefully
   - Implement proper cleanup

## Example: Efficient Animation System

```cpp
class SpriteAnimation {
private:
    LGFX_Sprite* frames;
    size_t frame_count;
    uint32_t frame_duration;
    
public:
    SpriteAnimation(LGFX* parent, size_t count, uint32_t duration) {
        frames = new LGFX_Sprite[count](parent);
        frame_count = count;
        frame_duration = duration;
        
        // Initialize frames
        for (size_t i = 0; i < count; i++) {
            if (!frames[i].createSprite(width, height)) {
                // Handle error
            }
            frames[i].setColorDepth(16);
        }
    }
    
    void update(uint32_t current_time) {
        size_t current_frame = (current_time / frame_duration) % frame_count;
        frames[current_frame].pushSprite(x, y);
    }
    
    ~SpriteAnimation() {
        for (size_t i = 0; i < frame_count; i++) {
            frames[i].deleteSprite();
        }
        delete[] frames;
    }
};
```

The sprite system provides a robust foundation for implementing complex graphics operations, animations, and off-screen rendering while maintaining efficient memory usage and performance optimization options. 