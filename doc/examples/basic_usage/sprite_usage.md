# Sprite Usage Guide

The LovyanGFX sprite system provides powerful off-screen graphics buffer management with hardware acceleration support. Sprites serve as secondary frame buffers in RAM, enabling efficient graphics operations without directly affecting the display.

## Memory and Performance Considerations

### Memory Requirements
The memory footprint of a sprite depends on three factors:
1. Dimensions (width × height)
2. Color depth (bits per pixel)
3. Alignment requirements for hardware acceleration

Memory calculation formula:
```cpp
// Basic memory requirement
size_t basic_memory = width * height * (bits_per_pixel / 8);

// With alignment (typically 32-bit)
size_t aligned_memory = ((basic_memory + 3) & ~3);  // Round up to 32-bit boundary
```

### Color Depth Impact
Color depth selection affects both memory usage and performance:
- 1-bit: Monochrome, 8 pixels per byte
- 4-bit: 16 colors, 2 pixels per byte
- 8-bit: 256 colors, 1 pixel per byte
- 16-bit: 65,536 colors (RGB565), 2 bytes per pixel
- 24-bit: 16.7M colors (RGB888), 3 bytes per pixel
- 32-bit: 16.7M colors with alpha (ARGB8888), 4 bytes per pixel

## Hardware Acceleration

### DMA Operations
When hardware supports it, sprite operations use DMA for efficient memory transfers:
```cpp
// Enable DMA transfers (if supported)
sprite.setDMAChannel(1);  // Assign DMA channel
sprite.enableDMA(true);   // Enable DMA operations
```

### Memory Alignment
For optimal DMA performance:
- Sprite dimensions should be multiples of 32 bits
- Memory addresses should be 32-bit aligned
- Buffer sizes should be multiples of 32 bits

## Advanced Operations

### Bit Manipulation
Color conversion and manipulation often use bit operations:
```cpp
// RGB888 to RGB565 conversion
uint16_t rgb888_to_rgb565(uint32_t rgb888) {
    // Extract RGB components
    uint8_t r = (rgb888 >> 16) & 0xFF;  // Red component (bits 23-16)
    uint8_t g = (rgb888 >> 8) & 0xFF;   // Green component (bits 15-8)
    uint8_t b = rgb888 & 0xFF;          // Blue component (bits 7-0)
    
    // Convert to RGB565 format
    // Red: 5 bits (0-31), shift right by 3
    // Green: 6 bits (0-63), shift right by 2
    // Blue: 5 bits (0-31), shift right by 3
    return ((r >> 3) << 11) | ((g >> 2) << 5) | (b >> 3);
}
```

### Rotation and Scaling
Transformation operations use fixed-point arithmetic for precision:
```cpp
// Fixed-point arithmetic for rotation
// angle is in degrees * 256 for precision
void rotateSprite(int32_t angle) {
    // Convert angle to radians * 256
    int32_t radian = (angle * 31415926L) / 180 / 256;
    
    // Calculate sine and cosine * 256
    int32_t sin_value = sin(radian) * 256;
    int32_t cos_value = cos(radian) * 256;
    
    // Transform coordinates
    int32_t new_x = (x * cos_value - y * sin_value) >> 8;
    int32_t new_y = (x * sin_value + y * cos_value) >> 8;
}
```

### Memory Management Strategies

#### Double Buffering
Efficient animation using two sprites:
```cpp
LGFX_Sprite front_buffer(&lcd);  // Display buffer
LGFX_Sprite back_buffer(&lcd);   // Drawing buffer

void setup() {
    // Create both buffers with same properties
    front_buffer.createSprite(width, height);
    back_buffer.createSprite(width, height);
}

void loop() {
    // Draw to back buffer
    back_buffer.drawSomething();
    
    // Swap buffers (pointer swap, no memory copy)
    std::swap(front_buffer, back_buffer);
    
    // Display front buffer
    front_buffer.pushSprite(0, 0);
}
```

## Basic Concepts

Sprites in LovyanGFX are off-screen buffers that can be used for:
- Double buffering
- Animation
- Complex graphical operations
- Memory-efficient rendering

## Basic Usage

### Sprite Creation
```cpp
#include <LovyanGFX.hpp>

LGFX lcd;                     // Display instance
LGFX_Sprite sprite(&lcd);     // Create sprite with parent display

void setup() {
    lcd.init();
    
    // Create sprite with dimensions and color depth
    sprite.createSprite(width, height);
    sprite.setColorDepth(color_depth);  // 1,2,4,8,16,24 bits
}
```

### Memory Management

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

### Basic Drawing
```cpp
// Clear sprite
sprite.fillScreen(TFT_BLACK);
sprite.clear();  // Alternative method

// Drawing primitives
sprite.drawPixel(x, y, color);
sprite.drawLine(x0, y0, x1, y1, color);
sprite.drawRect(x, y, w, h, color);
sprite.fillRect(x, y, w, h, color);
sprite.drawCircle(x, y, r, color);
sprite.fillCircle(x, y, r, color);
sprite.drawTriangle(x0, y0, x1, y1, x2, y2, color);
sprite.fillTriangle(x0, y0, x1, y1, x2, y2, color);
```

### Displaying Sprites

```cpp
// Basic display
sprite.pushSprite(x, y);                    // Display at coordinates

// Display with transparency
sprite.pushSprite(x, y, transparent_color);  // Use color as transparent

// Display to another sprite
sprite.pushSprite(&target_sprite, x, y);    // Push to another sprite
```

## Advanced Features

### Rotation and Transformation
```cpp
// Set rotation (0-7)
sprite.setRotation(rotation);

// Set pivot point for transformations
sprite.setPivot(x, y);

// Push with rotation and zoom
sprite.pushRotateZoom(dst_x, dst_y,        // Destination center
                     angle,                 // Rotation angle
                     zoom_x, zoom_y,        // Zoom factors
                     transparent_color);     // Transparent color
```

### Transparency and Blending
```cpp
// Set transparency color
sprite.setTransparentColor(color);

// Enable alpha blending
sprite.setColorDepth(lgfx::color_depth_t::argb8888);
sprite.fillSprite(color_with_alpha);
```

## Animation System

### Frame-Based Animation
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
            frames[i].createSprite(width, height);
            frames[i].setColorDepth(16);
            // Draw frame content...
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

## Best Practices

1. Memory Management
   - Delete sprites when no longer needed
   - Use appropriate color depth for your needs
   - Check available memory before creation
   - Consider using PSRAM for large sprites

2. Performance Optimization
   - Use hardware acceleration when available
   - Batch drawing operations
   - Reuse sprites instead of create/delete
   - Choose appropriate color depth

3. Resource Cleanup
   - Always delete sprites before recreation
   - Clean up animation resources
   - Handle memory allocation failures

## Common Issues

1. Memory Allocation
   - Check available memory
   - Use appropriate color depth
   - Handle allocation failures
   - Consider PSRAM usage

2. Performance
   - Monitor frame rates
   - Optimize drawing operations
   - Use appropriate color depth
   - Implement double buffering

3. Display Issues
   - Check transparency settings
   - Verify coordinates
   - Handle rotation correctly
   - Validate sprite dimensions

## Additional Resources

- [Sprite System Architecture](../core_concepts/sprite_system.md)
- [Memory Management Guide](../core_concepts/memory_management.md)
- [Performance Optimization](../core_concepts/performance.md)
- [Animation Guide](../advanced_examples/animation_effects.md) 