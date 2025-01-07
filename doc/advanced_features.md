# Advanced Features Guide

## Introduction

This guide explains advanced features in LovyanGFX that enable you to create sophisticated graphics effects and animations. Each section includes detailed code examples with line-by-line explanations.

### What You'll Learn

1. **Custom Shaders**
   - How to modify pixels during rendering
   - Creating gradient, ripple, and plasma effects
   - Understanding color component manipulation

2. **Hardware Acceleration**
   - Using DMA for faster sprite transfers
   - Configuring parallel bus for optimal performance
   - Managing memory efficiently

3. **Animation Effects**
   - Path-based movement with smooth interpolation
   - Complex sprite effects (blur, glow, wave distortion)
   - Efficient sprite management techniques

4. **Text Effects**
   - Adding outlines to text
   - Creating shadow effects with blur
   - Applying color gradients to text

### Prerequisites

Before using these features, ensure you understand:
- Basic LovyanGFX setup and usage
- Color formats (RGB565, RGB888)
- Basic sprite operations
- Memory management concepts

### Memory Considerations

Many advanced features require additional memory for:
- Sprite buffers (temporary storage)
- Effect calculations
- DMA operations

Always check available memory and use appropriate strategies:
- Reuse sprite buffers when possible
- Clean up temporary resources
- Monitor memory usage during effects

### Performance Tips

For optimal performance:
1. Use hardware acceleration when available
2. Batch similar operations together
3. Minimize memory allocations/deallocations
4. Avoid unnecessary pixel operations
5. Use appropriate color depths

### Common Pitfalls and Troubleshooting

#### Memory Issues

1. **Sprite Creation Fails**
   ```cpp
   // WRONG: Creating sprite without checking memory
   sprite->createSprite(width, height);  // Might fail silently
   
   // CORRECT: Check memory and handle failure
   size_t required = width * height * 2;  // Calculate required memory
   if (heap_caps_get_free_size(MALLOC_CAP_DMA) > required) {
       sprite->createSprite(width, height);
   } else {
       // Handle insufficient memory
       // Options:
       // 1. Reduce sprite size
       // 2. Free unused resources
       // 3. Use lower color depth
   }
   ```

2. **Memory Leaks**
   ```cpp
   // WRONG: Not cleaning up resources
   void applyEffect() {
       auto temp = new LGFX_Sprite(display);
       temp->createSprite(width, height);
       // ... use temp ...
       return;  // Memory leak!
   }
   
   // CORRECT: Proper cleanup
   void applyEffect() {
       auto temp = new LGFX_Sprite(display);
       temp->createSprite(width, height);
       // ... use temp ...
       temp->deleteSprite();  // Free sprite memory
       delete temp;           // Free object memory
   }
   ```

#### Display Issues

1. **Screen Tearing**
   ```cpp
   // WRONG: Direct drawing without transaction
   sprite1->pushSprite(x1, y1);
   sprite2->pushSprite(x2, y2);
   
   // CORRECT: Use transaction for atomic updates
   display.startWrite();
   {
       sprite1->pushSprite(x1, y1);
       sprite2->pushSprite(x2, y2);
   }
   display.endWrite();
   ```

2. **Transparent Color Issues**
   ```cpp
   // WRONG: Inconsistent transparent color
   sprite->fillSprite(0);  // Black background
   sprite->pushSprite(x, y, TFT_BLACK);  // Won't be fully transparent
   
   // CORRECT: Use consistent transparent color
   const uint32_t TRANSPARENT = TFT_BLACK;
   sprite->fillSprite(TRANSPARENT);
   sprite->pushSprite(x, y, TRANSPARENT);
   ```

#### Performance Issues

1. **Slow Rendering**
   ```cpp
   // WRONG: Individual pixel operations
   for (int y = 0; y < height; y++) {
       for (int x = 0; x < width; x++) {
           display.drawPixel(x, y, color);  // Very slow
       }
   }
   
   // CORRECT: Use hardware-accelerated functions
   display.fillRect(0, 0, width, height, color);
   ```

2. **DMA Conflicts**
   ```cpp
   // WRONG: Not checking DMA status
   sprite->pushSpriteDMA(x, y);
   sprite->pushSpriteDMA(x2, y2);  // Might corrupt display
   
   // CORRECT: Wait for DMA completion
   sprite->pushSpriteDMA(x, y);
   while (display.dmaBusy()) {  // Wait for completion
       delay(1);
   }
   sprite->pushSpriteDMA(x2, y2);
   ```

#### Effect Quality Issues

1. **Blurry Text**
   ```cpp
   // WRONG: Scaling text directly
   display.setTextSize(2);
   display.drawString(text, x, y);
   
   // CORRECT: Use sprite for better quality
   sprite->setTextSize(1);
   sprite->createSprite(width * 2, height * 2);
   sprite->drawString(text, 0, 0);
   sprite->pushRotateZoom(&display, x, y, 0, 2.0, 2.0);
   ```

2. **Poor Animation Quality**
   ```cpp
   // WRONG: Large position changes
   position += 10;  // Jerky movement
   
   // CORRECT: Smooth movement
   position += 1;  // Update more frequently with smaller steps
   // Or use fixed-point for sub-pixel precision
   position_fp += 256;  // 1.0 in fixed-point
   ```

### Debugging Tips

1. **Visual Debugging**
   ```cpp
   // Add debug visualization
   #ifdef DEBUG_GRAPHICS
       // Draw bounding boxes
       display.drawRect(x, y, width, height, TFT_RED);
       // Show update regions
       display.drawRect(dirty_x, dirty_y, 
                       dirty_w, dirty_h, TFT_GREEN);
   #endif
   ```

2. **Memory Monitoring**
   ```cpp
   // Print memory statistics
   void debugMemory() {
       Serial.printf("Free DMA: %d bytes\n",
           heap_caps_get_free_size(MALLOC_CAP_DMA));
       Serial.printf("Largest Free Block: %d bytes\n",
           heap_caps_get_largest_free_block(MALLOC_CAP_DMA));
       Serial.printf("Free Internal: %d bytes\n",
           ESP.getFreeHeap());
   }
   ```

This guide provides in-depth explanations of advanced features in LovyanGFX, with detailed comments explaining every line of code.

## Custom Shaders

Shaders allow you to modify how pixels are rendered, creating special effects.

### Pixel Shader Implementation

```cpp
// Custom pixel shader function that processes each pixel before it's drawn
// Parameters:
// - x, y: Current pixel coordinates being processed
// - color: Reference to the 32-bit RGBA color value (format: 0xRRGGBB)
void customShader(int32_t x, int32_t y, uint32_t &color) {
    // Extract RGB components from 32-bit color
    // Colors are stored in 8-bit components (0-255 range for each channel)
    // 0xFF is used as a mask to get only the last 8 bits (binary: 11111111)
    
    // Right shift by 16 to get red component (bits 23-16)
    // Example: 0xRRGGBB -> 0x0000RR, then mask with 0xFF
    uint8_t r = (color >> 16) & 0xFF;  // Red component (0-255)
    
    // Right shift by 8 to get green component (bits 15-8)
    // Example: 0xRRGGBB -> 0x00RRGG, then mask with 0xFF to get 0x000000GG
    uint8_t g = (color >> 8) & 0xFF;   // Green component (0-255)
    
    // No shift needed for blue (bits 7-0)
    // Just mask with 0xFF to get the lowest 8 bits
    uint8_t b = color & 0xFF;          // Blue component (0-255)
    
    // Calculate gradient based on x position
    // Divide x by screen width to get a value from 0.0 (left) to 1.0 (right)
    // This creates a smooth horizontal gradient
    float factor = (float)x / display.width();
    
    // Apply gradient to each color component
    // Multiply each component by the gradient factor
    // This makes colors fade from black (factor=0) to full color (factor=1)
    // Cast back to uint8_t to ensure 0-255 range
    r = (uint8_t)(r * factor);  // Gradient-adjusted red
    g = (uint8_t)(g * factor);  // Gradient-adjusted green
    b = (uint8_t)(b * factor);  // Gradient-adjusted blue
    
    // Reconstruct 32-bit color from modified components
    // Shift each component back to its original position
    // Red: Shift left by 16 (bits 23-16)
    // Green: Shift left by 8 (bits 15-8)
    // Blue: No shift needed (bits 7-0)
    color = (r << 16) |  // Position red in bits 23-16
            (g << 8)  |  // Position green in bits 15-8
            b;           // Position blue in bits 7-0
}

// Apply shader to display
void applyShader() {
    // Register our shader function
    // This will be called for each pixel drawn until removed
    display.setShader(customShader);
    
    // Draw a white rectangle covering entire screen
    // TFT_WHITE = 0xFFFFFF = maximum value for all color channels
    display.fillRect(
        0, 0,                    // Start at top-left corner
        display.width(),         // Full screen width
        display.height(),        // Full screen height
        TFT_WHITE               // White color (will be modified by shader)
    );
    
    // Unregister shader to prevent affecting future drawing operations
    display.removeShader();
}
```

### Advanced Shader Effects

```cpp
// Ripple effect shader - creates circular waves emanating from center
void rippleShader(int32_t x, int32_t y, uint32_t &color) {
    // Calculate distance from screen center
    // Subtract half width/height to make (0,0) the center point
    float dx = x - display.width() / 2;   // X distance from center
    float dy = y - display.height() / 2;  // Y distance from center
    
    // Calculate distance from center using Pythagorean theorem
    // sqrt(dx² + dy²) gives straight-line distance to center
    float distance = sqrt(dx * dx + dy * dy);
    
    // Calculate angle from center point (in radians)
    // atan2 gives angle in range -π to π
    // This is used to create radial effects
    float angle = atan2(dy, dx);
    
    // Create wave effect using sine function
    // distance * 0.1: Controls wave frequency (smaller = wider waves)
    // millis() * 0.001: Current time in seconds for animation
    // sin() returns values between -1 and 1
    float wave = sin(
        distance * 0.1 +         // Space component (radial waves)
        millis() * 0.001        // Time component (animation)
    );
    
    // Convert wave value (-1 to 1) to intensity (0 to 1)
    // Add 1 to get range 0 to 2, then multiply by 0.5 for 0 to 1
    float intensity = (wave + 1.0f) * 0.5f;
    
    // Extract and modify color components based on wave intensity
    // Same bit manipulation as basic shader
    uint8_t r = ((color >> 16) & 0xFF) * intensity;  // Red
    uint8_t g = ((color >> 8) & 0xFF) * intensity;   // Green
    uint8_t b = (color & 0xFF) * intensity;          // Blue
    
    // Reconstruct modified color
    color = (r << 16) | (g << 8) | b;
}

// Plasma effect shader - creates colorful moving patterns
void plasmaShader(int32_t x, int32_t y, uint32_t &color) {
    // Convert milliseconds to seconds for smoother animation
    float time = millis() * 0.001;  // Current time in seconds
    
    // Combine multiple sine waves for plasma effect
    // Each sin() creates a different wave pattern:
    float value = 
        // Horizontal waves (vary with x)
        sin(x * 0.1 + time) +  // 0.1 = wave frequency
        // Vertical waves (vary with y)
        sin(y * 0.1 + time) +
        // Diagonal waves (vary with x+y)
        sin((x + y) * 0.1 + time) +
        // Circular waves (vary with distance from origin)
        sin(sqrt(x * x + y * y) * 0.1);
    
    // Normalize combined waves to 0-255 range for color
    // value is in range -4 to 4 (sum of 4 sine waves)
    // Add 4 to make it 0 to 8, then multiply by 32 (255/8)
    value = (value + 4) * 32;  // Scale to 0-255 range
    
    // Extract color components
    uint8_t r = (color >> 16) & 0xFF;  // Red component
    uint8_t g = (color >> 8) & 0xFF;   // Green component
    uint8_t b = color & 0xFF;          // Blue component
    
    // Modify each component based on plasma value
    // Divide by 255 to normalize to 0-1 range
    r = value * r / 255;  // Modified red
    g = value * g / 255;  // Modified green
    b = value * b / 255;  // Modified blue
    
    // Reconstruct modified color
    color = (r << 16) | (g << 8) | b;
}
```

These shaders demonstrate different ways to manipulate pixel colors for various effects:
1. The basic shader creates a horizontal gradient by varying color intensity based on x position
2. The ripple shader creates circular waves using distance from center and sine waves
3. The plasma shader combines multiple sine waves to create complex moving patterns

Each shader follows the same basic structure:
1. Extract color components (R,G,B)
2. Calculate effect intensity based on position and/or time
3. Modify color components
4. Reconstruct final color

## Custom Shader Effects

### Basic Color Manipulation

```cpp
// Function to create a gradient effect by modifying pixel colors
// Parameters:
// - x, y: Pixel coordinates in the sprite/display
// - color: Original 16-bit RGB565 color value
uint16_t customShader(uint16_t x, uint16_t y, uint16_t color) {
    // Extract RGB components from 16-bit color
    // RGB565 format: RRRRRGGGGGGBBBBB
    uint8_t r = (color >> 11) & 0x1F;        // Extract red (5 bits)
    uint8_t g = (color >> 5) & 0x3F;         // Extract green (6 bits)
    uint8_t b = color & 0x1F;                // Extract blue (5 bits)
    
    // Calculate gradient factor based on position
    // Factor ranges from 0.0 to 1.0 based on x position
    float factor = (float)x / WIDTH;          // Normalize x coordinate
    
    // Modify color components based on position
    // Increase red towards right side of screen
    r = min(31, (uint8_t)(r * (1.0 + factor)));  // Max red value is 31 (5 bits)
    // Decrease blue towards right side
    b = max(0, (uint8_t)(b * (1.0 - factor)));   // Min blue value is 0
    
    // Reconstruct 16-bit color value
    // Shift components back to their positions
    return (r << 11) | (g << 5) | b;         // RGB565 format
}

// Function to create a ripple effect
// Uses sine waves based on distance from center
uint16_t rippleShader(uint16_t x, uint16_t y, uint16_t color, float time) {
    // Calculate distance from center of screen
    float dx = x - (WIDTH / 2);              // X distance from center
    float dy = y - (HEIGHT / 2);             // Y distance from center
    float dist = sqrt(dx*dx + dy*dy);        // Euclidean distance
    
    // Create ripple effect using sine wave
    // Wave properties:
    const float WAVE_LENGTH = 20.0f;         // Distance between wave peaks
    const float WAVE_SPEED = 2.0f;           // Wave movement speed
    const float WAVE_HEIGHT = 0.3f;          // Wave amplitude (color intensity)
    
    // Calculate wave intensity at this point
    float wave = sin(
        (dist / WAVE_LENGTH) +               // Distance-based phase
        (time * WAVE_SPEED)                  // Time-based phase
    ) * WAVE_HEIGHT;                         // Scale by height
    
    // Extract color components
    uint8_t r = (color >> 11) & 0x1F;
    uint8_t g = (color >> 5) & 0x3F;
    uint8_t b = color & 0x1F;
    
    // Apply wave effect to each component
    // Clamp values between 0 and max for each component
    r = min(31, max(0, (int)(r * (1.0 + wave))));  // 5-bit red
    g = min(63, max(0, (int)(g * (1.0 + wave))));  // 6-bit green
    b = min(31, max(0, (int)(b * (1.0 + wave))));  // 5-bit blue
    
    // Reconstruct modified color
    return (r << 11) | (g << 5) | b;
}

// Function to create a plasma effect
// Combines multiple sine waves for complex patterns
uint16_t plasmaShader(uint16_t x, uint16_t y, float time) {
    // Scale coordinates for smoother patterns
    float px = x * 0.05f;                    // X coordinate scaling
    float py = y * 0.05f;                    // Y coordinate scaling
    
    // Create plasma pattern using multiple sine waves
    // Each wave adds a different frequency component
    float v1 = sin(px + time);                          // Horizontal wave
    float v2 = sin(py + time);                          // Vertical wave
    float v3 = sin(px + py + time * 2.0f);             // Diagonal wave
    float v4 = sin(sqrt((px*px + py*py) * 0.1f));      // Circular wave
    
    // Combine waves and normalize to 0.0 - 1.0 range
    float plasma = (v1 + v2 + v3 + v4) * 0.25f + 0.5f;
    
    // Convert plasma value to RGB components
    // Use different phases for each color
    uint8_t r = (uint8_t)(31 * sin(plasma * M_PI));            // Red phase
    uint8_t g = (uint8_t)(63 * sin(plasma * M_PI + M_PI/2));   // Green phase
    uint8_t b = (uint8_t)(31 * sin(plasma * M_PI + M_PI));     // Blue phase
    
    // Ensure values are within valid ranges
    r = min(31, max(0, r));  // 5-bit red (0-31)
    g = min(63, max(0, g));  // 6-bit green (0-63)
    b = min(31, max(0, b));  // 5-bit blue (0-31)
    
    // Combine into RGB565 format
    return (r << 11) | (g << 5) | b;
}
```

### Advanced Color Effects

```cpp
// Class to manage color palette transitions
// Useful for smooth color cycling effects
class PaletteManager {
private:
    // Color palette storage
    // Each color is RGB565 format
    uint16_t palette[256];
    
    // Current palette position for cycling
    float currentIndex;
    
    // Palette transition speed
    float cycleSpeed;
    
public:
    // Initialize palette with rainbow colors
    void initRainbow() {
        for (int i = 0; i < 256; i++) {
            // Convert HSV to RGB
            // Hue: 0-360 degrees (map from 0-255)
            float hue = (i * 360.0f) / 255.0f;
            // Saturation: 100%
            float sat = 1.0f;
            // Value: 100%
            float val = 1.0f;
            
            // Convert HSV to RGB (0-1 range)
            float c = val * sat;
            float x = c * (1 - abs(fmod(hue / 60.0f, 2) - 1));
            float m = val - c;
            
            float r, g, b;
            if (hue < 60) {
                r = c; g = x; b = 0;
            } else if (hue < 120) {
                r = x; g = c; b = 0;
            } else if (hue < 180) {
                r = 0; g = c; b = x;
            } else if (hue < 240) {
                r = 0; g = x; b = c;
            } else if (hue < 300) {
                r = x; g = 0; b = c;
            } else {
                r = c; g = 0; b = x;
            }
            
            // Convert to RGB565
            uint8_t r8 = (uint8_t)((r + m) * 31);  // 5 bits
            uint8_t g8 = (uint8_t)((g + m) * 63);  // 6 bits
            uint8_t b8 = (uint8_t)((b + m) * 31);  // 5 bits
            
            // Store in palette
            palette[i] = (r8 << 11) | (g8 << 5) | b8;
        }
    }
    
    // Get color from palette with smooth interpolation
    uint16_t getColor(float index) {
        // Ensure index is in valid range
        index = fmod(index, 256.0f);
        if (index < 0) index += 256.0f;
        
        // Get integer indices for interpolation
        int i1 = (int)index;
        int i2 = (i1 + 1) % 256;
        
        // Calculate interpolation factor
        float t = index - i1;
        
        // Get colors to interpolate between
        uint16_t c1 = palette[i1];
        uint16_t c2 = palette[i2];
        
        // Extract components
        uint8_t r1 = (c1 >> 11) & 0x1F;
        uint8_t g1 = (c1 >> 5) & 0x3F;
        uint8_t b1 = c1 & 0x1F;
        
        uint8_t r2 = (c2 >> 11) & 0x1F;
        uint8_t g2 = (c2 >> 5) & 0x3F;
        uint8_t b2 = c2 & 0x1F;
        
        // Interpolate components
        uint8_t r = r1 + (uint8_t)(t * (r2 - r1));
        uint8_t g = g1 + (uint8_t)(t * (g2 - g1));
        uint8_t b = b1 + (uint8_t)(t * (b2 - b1));
        
        // Return interpolated color
        return (r << 11) | (g << 5) | b;
    }
    
    // Update palette cycling
    void update(float deltaTime) {
        currentIndex += cycleSpeed * deltaTime;
        if (currentIndex >= 256.0f) {
            currentIndex -= 256.0f;
        }
    }
};
```

These shader effects demonstrate several key concepts:

1. Color Space Manipulation
   - RGB565 color format (16-bit: 5 red, 6 green, 5 blue bits)
   - Component extraction and reconstruction
   - Color space conversion (HSV to RGB)

2. Mathematical Effects
   - Sine waves for periodic effects
   - Distance calculations for radial patterns
   - Linear interpolation for smooth transitions

3. Performance Considerations
   - Pre-calculated values where possible
   - Efficient bit manipulation
   - Bounded value ranges to prevent overflow

## Hardware Acceleration

### DMA (Direct Memory Access) Operations

```cpp
// Class to manage sprite transfers using DMA
// DMA allows data transfer without CPU intervention
class DMASpriteManager {
    // Array to store sprite pointers
    // MAX_SPRITES is typically defined based on available memory
    // and maximum number of sprites needed simultaneously
    LGFX_Sprite* sprites[MAX_SPRITES];
    
    // Flag to track DMA transfer status
    // True while transfer is in progress
    // False when transfer is complete or not started
    bool dma_busy;
    
public:
    // Update sprite position using DMA transfer
    // Parameters:
    // - index: Sprite array index (0 to MAX_SPRITES-1)
    // - x, y: Target position on screen
    void updateSprite(uint32_t index, uint16_t x, uint16_t y) {
        // Safety checks before starting transfer
        if (dma_busy ||                    // Don't start if DMA is busy
            index >= MAX_SPRITES) {        // Prevent array overflow
            return;
        }
        
        // Set busy flag before starting transfer
        // This prevents starting another transfer while one is in progress
        dma_busy = true;
        
        // Start DMA transfer
        // Lambda function is called when transfer completes
        sprites[index]->pushSpriteDMA(
            x, y,                          // Destination coordinates
            [this]() {                     // Completion callback
                dma_busy = false;          // Clear busy flag
            }
        );
    }
    
    // Wait for current DMA transfer to complete
    // This prevents starting new operations while DMA is busy
    void waitDMA() {
        // Loop until transfer completes
        while (dma_busy) {
            delay(1);  // Short delay to prevent tight loop
                      // 1ms is typically sufficient for task switching
        }
    }
};
```

### Parallel Bus Configuration

```cpp
// Configure 8-bit parallel bus for maximum performance
// Parallel bus allows faster data transfer than SPI
void setupParallelBus() {
    // Create new 8-bit parallel bus instance
    auto bus = new lgfx::Bus_Parallel8();
    // Get configuration structure for bus setup
    auto cfg = bus->config();
    
    // Configure control pins
    // These pins control data flow on the bus
    cfg.pin_wr = 4;   // Write strobe pin (active low)
    cfg.pin_rd = 5;   // Read strobe pin (active low)
    cfg.pin_rs = 6;   // Register select/Data-Command pin
    
    // Configure 8-bit data bus pins
    // These pins transfer one byte of data in parallel
    // Pins should be consecutive for best performance
    cfg.pin_d0 = 16;  // Data bit 0 (LSB)
    cfg.pin_d1 = 17;  // Data bit 1
    cfg.pin_d2 = 18;  // Data bit 2
    cfg.pin_d3 = 19;  // Data bit 3
    cfg.pin_d4 = 20;  // Data bit 4
    cfg.pin_d5 = 21;  // Data bit 5
    cfg.pin_d6 = 22;  // Data bit 6
    cfg.pin_d7 = 23;  // Data bit 7 (MSB)
    
    // Set bus frequencies
    // Higher frequencies = faster updates but may need level shifters
    cfg.freq_write = 40000000;  // Write at 40MHz
                               // Maximum stable frequency depends on:
                               // - MCU capabilities
                               // - PCB layout
                               // - Cable length
    
    cfg.freq_read = 20000000;   // Read at 20MHz
                               // Read is typically slower than write
                               // due to display controller limitations
    
    // Apply configuration to bus
    bus->config(cfg);
    // Attach configured bus to display
    // This replaces any previous bus configuration
    display.setBus(bus);
}
```

### Memory-Mapped Operations

```cpp
// Class to manage direct memory access to display buffer
// This provides fastest possible update speed
class MemoryMappedDisplay {
private:
    // Direct pointer to display memory
    // Allows writing pixels without function calls
    uint16_t* frameBuffer;
    
    // Display dimensions
    const uint16_t WIDTH;   // Display width in pixels
    const uint16_t HEIGHT;  // Display height in pixels
    
    // Calculate memory requirements
    // Each pixel uses 2 bytes (16-bit color)
    const uint32_t BUFFER_SIZE;  // Total bytes needed
    
public:
    // Initialize display with memory mapping
    MemoryMappedDisplay(uint16_t w, uint16_t h)
        : WIDTH(w)
        , HEIGHT(h)
        , BUFFER_SIZE(w * h * 2)  // 2 bytes per pixel (RGB565)
    {
        // Allocate memory for frame buffer
        // Use DMA-capable memory for hardware acceleration
        frameBuffer = (uint16_t*)heap_caps_malloc(
            BUFFER_SIZE,           // Number of bytes needed
            MALLOC_CAP_DMA        // Memory must be DMA-capable
        );
    }
    
    // Fast pixel write using direct memory access
    void writePixel(uint16_t x, uint16_t y, uint16_t color) {
        // Bounds checking to prevent buffer overflow
        if (x >= WIDTH || y >= HEIGHT) return;
        
        // Calculate pixel offset in buffer
        // offset = y * width + x
        // This converts 2D coordinates to 1D array index
        uint32_t offset = y * WIDTH + x;
        
        // Write color directly to memory
        // No function call overhead
        frameBuffer[offset] = color;
    }
    
    // Fast horizontal line drawing
    void writeHLine(uint16_t x, uint16_t y, uint16_t w, uint16_t color) {
        // Bounds checking
        if (y >= HEIGHT) return;
        if (x >= WIDTH) return;
        
        // Clip width to screen bounds
        if (x + w > WIDTH) {
            w = WIDTH - x;
        }
        
        // Calculate start offset
        uint32_t offset = y * WIDTH + x;
        
        // Use hardware acceleration if available
        if (w > 32) {  // Threshold for DMA usage
                      // Below 32 pixels, CPU copy is faster
            // DMA copy for long lines
            dmaFill(
                &frameBuffer[offset],  // Destination
                color,                 // Fill value
                w * 2                  // Bytes to fill (2 per pixel)
            );
        } else {
            // CPU copy for short lines
            // Manual loop is faster for small counts
            for (uint16_t i = 0; i < w; i++) {
                frameBuffer[offset + i] = color;
            }
        }
    }
    
    // Update display from buffer
    void refresh() {
        // Start DMA transfer of entire buffer
        display.startWrite();
        display.writeBytes(
            (uint8_t*)frameBuffer,  // Source data
            BUFFER_SIZE,            // Number of bytes
            true                    // Use DMA if available
        );
        display.endWrite();
    }
};
```

These hardware acceleration techniques provide significant performance improvements:
1. DMA transfers free the CPU during data movement
2. Parallel bus increases data throughput
3. Memory mapping eliminates function call overhead
4. Hardware-assisted operations reduce CPU load

Key considerations:
1. DMA requires proper memory alignment
2. Bus timing depends on hardware capabilities
3. Memory mapping needs careful bounds checking
4. Hardware acceleration may need minimum operation sizes

## Best Practices

### Memory Management

```cpp
// Example of efficient sprite memory management
class SpriteManager {
    // Track allocated sprites and their states
    struct SpriteInfo {
        LGFX_Sprite* sprite;
        bool in_use;
        uint32_t last_used;
    };
    
    // Vector to store sprite information
    std::vector<SpriteInfo> sprites;
    
    // Maximum memory allowed for sprites (in bytes)
    const size_t MAX_MEMORY = 1024 * 1024;  // 1MB limit
    
    // Current memory usage
    size_t current_memory = 0;
    
public:
    // Request sprite with specific dimensions
    LGFX_Sprite* requestSprite(uint16_t width, uint16_t height) {
        // Calculate required memory
        size_t required = width * height * 2;  // 2 bytes per pixel (16-bit color)
        
        // Check if we have enough memory
        if (current_memory + required > MAX_MEMORY) {
            // Try to free unused sprites
            cleanupUnused();
            
            // Check again after cleanup
            if (current_memory + required > MAX_MEMORY) {
                return nullptr;  // Still not enough memory
            }
        }
        
        // Look for existing unused sprite of right size
        for (auto& info : sprites) {
            if (!info.in_use &&
                info.sprite->width() == width &&
                info.sprite->height() == height) {
                // Found suitable sprite
                info.in_use = true;
                info.last_used = millis();
                return info.sprite;
            }
        }
        
        // Create new sprite if none found
        auto sprite = new LGFX_Sprite(&display);
        sprite->createSprite(width, height);
        
        // Add to tracking
        sprites.push_back({
            sprite,         // Sprite pointer
            true,          // Mark as in use
            millis()       // Set last used time
        });
        
        // Update memory tracking
        current_memory += required;
        
        return sprite;
    }
    
private:
    // Clean up sprites not used recently
    void cleanupUnused() {
        uint32_t now = millis();
        
        for (auto& info : sprites) {
            // Free sprites unused for more than 5 seconds
            if (info.in_use && (now - info.last_used > 5000)) {
                // Calculate memory to free
                size_t freed = info.sprite->width() *
                             info.sprite->height() * 2;
                
                // Delete sprite
                info.sprite->deleteSprite();
                info.in_use = false;
                
                // Update memory tracking
                current_memory -= freed;
            }
        }
    }
};
```

These examples demonstrate advanced features with detailed explanations of how each line works. Understanding these concepts will help you create more efficient and sophisticated graphics applications. 

## Advanced Animation Effects

### Path-Based Animation

```cpp
// Class for animating objects along complex paths
class PathAnimator {
    // Structure to hold point data with position and rotation
    struct Point {
        float x, y;       // Position coordinates
        float angle;      // Rotation angle at this point
    };
    
    // Vector to store path points
    std::vector<Point> points;
    // Current progress along path (0.0 to 1.0)
    float progress;
    // Movement speed (units per frame)
    float speed;
    
public:
    // Add a new point to the path
    void addPoint(float x, float y) {
        // Create new point structure
        Point p = {
            x,           // X coordinate
            y,           // Y coordinate
            0           // Initial angle (will be calculated)
        };
        
        // If this isn't the first point, calculate angle
        if (!points.empty()) {
            // Get reference to previous point for angle calculation
            Point& prev = points.back();
            
            // Calculate angle between previous point and new point
            // atan2 gives angle in radians (-π to π)
            p.angle = atan2(y - prev.y,    // Y difference
                          x - prev.x);     // X difference
        }
        
        // Add point to path
        points.push_back(p);
    }
    
    // Get interpolated position along path
    // Parameter t ranges from 0.0 (start) to 1.0 (end)
    Point getPosition(float t) {
        // Check if we have enough points for interpolation
        if (points.size() < 2) {
            return {0, 0, 0};  // Return origin if not enough points
        }
        
        // Calculate total path length
        float totalLength = 0;
        // Store segment lengths for later use
        std::vector<float> lengths;
        
        // Calculate length of each path segment
        for (size_t i = 1; i < points.size(); i++) {
            // Calculate distance between points using Pythagorean theorem
            float dx = points[i].x - points[i-1].x;    // X difference
            float dy = points[i].y - points[i-1].y;    // Y difference
            float len = sqrt(dx * dx + dy * dy);       // Segment length
            
            totalLength += len;        // Add to total path length
            lengths.push_back(len);    // Store segment length
        }
        
        // Convert progress (t) to actual distance along path
        float targetDist = t * totalLength;
        float currentDist = 0;
        
        // Find which segment contains our target position
        for (size_t i = 0; i < lengths.size(); i++) {
            // Check if target is in this segment
            if (currentDist + lengths[i] >= targetDist) {
                // Calculate how far along this segment we are (0.0 to 1.0)
                float segmentT = (targetDist - currentDist) / lengths[i];
                
                // Get points for this segment
                Point& p1 = points[i];        // Start point
                Point& p2 = points[i+1];      // End point
                
                // Interpolate between points
                return {
                    // Linear interpolation of X coordinate
                    p1.x + (p2.x - p1.x) * segmentT,
                    // Linear interpolation of Y coordinate
                    p1.y + (p2.y - p1.y) * segmentT,
                    // Linear interpolation of angle
                    p1.angle + (p2.angle - p1.angle) * segmentT
                };
            }
            
            // Add segment length to current distance
            currentDist += lengths[i];
        }
        
        // Return last point if we've gone past the end
        return points.back();
    }
};
```

### Advanced Sprite Effects

```cpp
// Class for creating complex sprite effects
class SpriteEffects {
private:
    // Source sprite to apply effects to
    LGFX_Sprite* source;
    // Buffer for effect processing
    LGFX_Sprite* buffer;
    
public:
    // Constructor takes source sprite
    SpriteEffects(LGFX_Sprite* src) : source(src) {
        // Create processing buffer same size as source
        buffer = new LGFX_Sprite(source->getParent());
        buffer->createSprite(
            source->width(),     // Same width as source
            source->height()     // Same height as source
        );
    }
    
    // Apply blur effect to sprite
    // radius: Blur radius in pixels
    void applyBlur(uint8_t radius) {
        // Temporary buffers for horizontal and vertical passes
        LGFX_Sprite* temp = new LGFX_Sprite(source->getParent());
        temp->createSprite(source->width(), source->height());
        
        // Copy source to temp buffer
        source->pushSprite(temp, 0, 0);
        
        // Horizontal blur pass
        for (int16_t y = 0; y < source->height(); y++) {
            for (int16_t x = 0; x < source->width(); x++) {
                // Accumulate color values
                uint32_t r = 0, g = 0, b = 0;
                uint32_t count = 0;
                
                // Sample neighboring pixels
                for (int16_t i = -radius; i <= radius; i++) {
                    // Check if sample position is within bounds
                    if (x + i >= 0 && x + i < source->width()) {
                        // Get pixel color
                        uint32_t color = temp->readPixel(x + i, y);
                        // Accumulate color components
                        r += (color >> 16) & 0xFF;  // Red
                        g += (color >> 8) & 0xFF;   // Green
                        b += color & 0xFF;          // Blue
                        count++;
                    }
                }
                
                // Calculate average color
                r /= count;  // Average red
                g /= count;  // Average green
                b /= count;  // Average blue
                
                // Write blurred pixel to buffer
                buffer->writePixel(x, y, (r << 16) | (g << 8) | b);
            }
        }
        
        // Copy buffer back to temp for vertical pass
        buffer->pushSprite(temp, 0, 0);
        
        // Vertical blur pass
        for (int16_t x = 0; x < source->width(); x++) {
            for (int16_t y = 0; y < source->height(); y++) {
                // Accumulate color values
                uint32_t r = 0, g = 0, b = 0;
                uint32_t count = 0;
                
                // Sample neighboring pixels
                for (int16_t i = -radius; i <= radius; i++) {
                    // Check if sample position is within bounds
                    if (y + i >= 0 && y + i < source->height()) {
                        // Get pixel color
                        uint32_t color = temp->readPixel(x, y + i);
                        // Accumulate color components
                        r += (color >> 16) & 0xFF;  // Red
                        g += (color >> 8) & 0xFF;   // Green
                        b += color & 0xFF;          // Blue
                        count++;
                    }
                }
                
                // Calculate average color
                r /= count;  // Average red
                g /= count;  // Average green
                b /= count;  // Average blue
                
                // Write blurred pixel to source
                source->writePixel(x, y, (r << 16) | (g << 8) | b);
            }
        }
        
        // Clean up temporary buffer
        temp->deleteSprite();
        delete temp;
    }
    
    // Apply glow effect to sprite
    // color: Glow color
    // intensity: Glow strength (0.0 to 1.0)
    void applyGlow(uint32_t color, float intensity) {
        // Extract glow color components
        uint8_t glow_r = (color >> 16) & 0xFF;  // Red component
        uint8_t glow_g = (color >> 8) & 0xFF;   // Green component
        uint8_t glow_b = color & 0xFF;          // Blue component
        
        // First apply blur for glow base
        applyBlur(4);  // Radius of 4 pixels
        
        // Apply glow color and intensity
        for (int16_t y = 0; y < source->height(); y++) {
            for (int16_t x = 0; x < source->width(); x++) {
                // Get current pixel color
                uint32_t pixel = source->readPixel(x, y);
                uint8_t r = (pixel >> 16) & 0xFF;  // Red
                uint8_t g = (pixel >> 8) & 0xFF;   // Green
                uint8_t b = pixel & 0xFF;          // Blue
                
                // Mix with glow color based on intensity
                r = r + (glow_r - r) * intensity;
                g = g + (glow_g - g) * intensity;
                b = b + (glow_b - b) * intensity;
                
                // Write modified pixel back
                source->writePixel(x, y, (r << 16) | (g << 8) | b);
            }
        }
    }
    
    // Apply wave distortion effect
    // amplitude: Wave height
    // frequency: Wave frequency
    // phase: Wave phase (animation progress)
    void applyWave(float amplitude, float frequency, float phase) {
        // Clear buffer
        buffer->fillScreen(TFT_BLACK);
        
        // Process each pixel
        for (int16_t y = 0; y < source->height(); y++) {
            for (int16_t x = 0; x < source->width(); x++) {
                // Calculate wave offset
                float offset = amplitude * 
                    sin(frequency * x + phase);
                
                // Calculate source position with wave distortion
                int16_t src_y = y + (int16_t)offset;
                
                // Check if source position is valid
                if (src_y >= 0 && src_y < source->height()) {
                    // Copy pixel with wave distortion
                    uint32_t color = source->readPixel(x, src_y);
                    buffer->writePixel(x, y, color);
                }
            }
        }
        
        // Copy buffer back to source
        buffer->pushSprite(source, 0, 0);
    }
};
```

These examples demonstrate advanced animation and effect techniques with detailed explanations of each operation. The code includes comprehensive comments explaining the mathematics and logic behind each effect. 

### Advanced Text Effects

```cpp
// Class to create advanced text effects
class TextEffects {
private:
    FontManager& font;  // Reference to font manager
    
public:
    TextEffects(FontManager& f) : font(f) {}
    
    // Draw text with outline effect
    void drawOutlinedText(int16_t x, int16_t y,
                         const char* text,
                         uint16_t fillColor,    // Inner color
                         uint16_t outlineColor, // Outline color
                         uint8_t thickness = 1) // Outline thickness
    {
        // Draw outline by offsetting text in 8 directions
        static const int8_t offsets[8][2] = {
            {-1, -1}, {0, -1}, {1, -1},  // Top row
            {-1,  0},          {1,  0},  // Middle row
            {-1,  1}, {0,  1}, {1,  1}   // Bottom row
        };
        
        // Draw outline layers
        for (uint8_t t = 0; t < thickness; t++) {
            for (uint8_t i = 0; i < 8; i++) {
                int16_t dx = offsets[i][0] * (t + 1);
                int16_t dy = offsets[i][1] * (t + 1);
                
                font.setTextColor(outlineColor);
                font.drawText(x + dx, y + dy, text);
            }
        }
        
        // Draw main text
        font.setTextColor(fillColor);
        font.drawText(x, y, text);
    }
    
    // Draw text with drop shadow
    void drawShadowedText(int16_t x, int16_t y,
                         const char* text,
                         uint16_t textColor,    // Text color
                         uint16_t shadowColor,  // Shadow color
                         uint8_t offsetX = 2,   // Shadow X offset
                         uint8_t offsetY = 2,   // Shadow Y offset
                         uint8_t blur = 1)      // Shadow blur radius
    {
        // Draw blurred shadow layers
        for (uint8_t by = 0; by <= blur; by++) {
            for (uint8_t bx = 0; bx <= blur; bx++) {
                // Calculate alpha for this blur pixel
                // Further pixels get lower alpha
                uint8_t alpha = 255 / ((bx + by + 1) * 2);
                
                // Blend shadow color with background
                uint16_t blurColor = alphaBlend(
                    shadowColor,    // Shadow color
                    0x0000,        // Background (black)
                    alpha          // Calculated alpha
                );
                
                // Draw shadow layer
                font.setTextColor(blurColor);
                font.drawText(
                    x + offsetX + bx,  // Offset + blur X
                    y + offsetY + by,  // Offset + blur Y
                    text
                );
            }
        }
        
        // Draw main text
        font.setTextColor(textColor);
        font.drawText(x, y, text);
    }
    
    // Draw text with color gradient
    void drawGradientText(int16_t x, int16_t y,
                         const char* text,
                         uint16_t startColor,  // Gradient start color
                         uint16_t endColor)    // Gradient end color
    {
        // Calculate total text width for gradient
        uint16_t width = font.getTextWidth(text);
        if (width == 0) return;
        
        // Draw each character with interpolated color
        float progress = 0.0f;
        const char* p = text;
        int16_t curX = x;
        
        while (*p) {
            // Calculate color for this character
            uint16_t color = interpolateColor(
                startColor,  // Start of gradient
                endColor,    // End of gradient
                progress    // Position in gradient (0-1)
            );
            
            // Draw single character
            char tmp[2] = {*p, 0};
            font.setTextColor(color);
            font.drawText(curX, y, tmp);
            
            // Advance position and progress
            curX += font.getCharWidth(*p);
            progress = (float)(curX - x) / width;
            p++;
        }
    }
    
private:
    // Blend colors using alpha value (0-255)
    uint16_t alphaBlend(uint16_t fg, uint16_t bg,
                       uint8_t alpha)
    {
        // Handle special cases
        if (alpha == 0) return bg;    // Fully transparent
        if (alpha == 255) return fg;  // Fully opaque
        
        // Extract RGB components (RGB565 format)
        uint8_t fgR = (fg >> 11) & 0x1F;  // 5 bits red
        uint8_t fgG = (fg >> 5) & 0x3F;   // 6 bits green
        uint8_t fgB = fg & 0x1F;          // 5 bits blue
        
        uint8_t bgR = (bg >> 11) & 0x1F;
        uint8_t bgG = (bg >> 5) & 0x3F;
        uint8_t bgB = bg & 0x1F;
        
        // Blend components
        float a = alpha / 255.0f;
        uint8_t r = fgR * a + bgR * (1 - a);
        uint8_t g = fgG * a + bgG * (1 - a);
        uint8_t b = fgB * a + bgB * (1 - a);
        
        // Reconstruct RGB565 color
        return (r << 11) | (g << 5) | b;
    }
    
    // Interpolate between two colors
    uint16_t interpolateColor(uint16_t c1, uint16_t c2,
                            float t)
    {
        // Handle special cases
        if (t <= 0.0f) return c1;  // Start of gradient
        if (t >= 1.0f) return c2;  // End of gradient
        
        // Extract RGB components
        uint8_t r1 = (c1 >> 11) & 0x1F;
        uint8_t g1 = (c1 >> 5) & 0x3F;
        uint8_t b1 = c1 & 0x1F;
        
        uint8_t r2 = (c2 >> 11) & 0x1F;
        uint8_t g2 = (c2 >> 5) & 0x3F;
        uint8_t b2 = c2 & 0x1F;
        
        // Interpolate components
        uint8_t r = r1 + (r2 - r1) * t;
        uint8_t g = g1 + (g2 - g1) * t;
        uint8_t b = b1 + (b2 - b1) * t;
        
        // Reconstruct RGB565 color
        return (r << 11) | (g << 5) | b;
    }
};
```

The text effects system demonstrates several advanced techniques:

1. Outline Effects
   - Multi-directional offset rendering
   - Configurable outline thickness
   - Efficient outline generation
   - Clean edge rendering

2. Shadow Effects
   - Configurable shadow offset
   - Gaussian-like blur simulation
   - Alpha blending for smooth shadows
   - Optimized blur calculation

3. Gradient Effects
   - Smooth color interpolation
   - Character-by-character rendering
   - Progress-based color calculation
   - Efficient color component handling

4. Color Operations
   - RGB565 color manipulation
   - Alpha blending calculations
   - Color space interpolation
   - Optimized component extraction

## Touch Input Handling

### Basic Touch Processing

```cpp
// Structure to hold touch event data
// Contains all information about a single touch point
struct TouchEvent {
    // Touch coordinates in screen space
    int16_t x;          // X coordinate (0 to screen width-1)
    int16_t y;          // Y coordinate (0 to screen height-1)
    
    // Raw ADC values from touch controller
    uint16_t pressure;  // Touch pressure (0-1023, hardware dependent)
    
    // Calculated touch properties
    uint32_t duration;  // How long touch has been active (milliseconds)
    bool isNew;        // True if this is first frame of touch
    
    // Previous touch position for gesture detection
    int16_t lastX;     // Previous X coordinate
    int16_t lastY;     // Previous Y coordinate
    
    // Velocity calculation
    float velocityX;   // X velocity in pixels per second
    float velocityY;   // Y velocity in pixels per second
    uint32_t lastTime; // Timestamp of last update
};

// Class to manage touch input and gesture detection
class TouchManager {
private:
    // Constants for gesture detection
    static const uint16_t SWIPE_THRESHOLD = 50;      // Minimum pixels for swipe
    static const uint16_t TAP_THRESHOLD = 10;        // Maximum movement for tap
    static const uint32_t TAP_DURATION = 200;        // Maximum ms for tap
    static const uint32_t HOLD_DURATION = 500;       // Minimum ms for hold
    
    // Current touch state
    TouchEvent currentTouch;
    bool touchActive;
    
    // Gesture state tracking
    bool gestureInProgress;
    GestureType currentGesture;
    
public:
    // Update touch state with new readings
    // Should be called every frame
    void update() {
        // Read hardware touch controller
        uint16_t x, y, pressure;
        bool touched = readTouchInput(&x, &y, &pressure);
        
        if (touched) {
            uint32_t currentTime = millis();
            
            if (!touchActive) {
                // New touch detected
                touchActive = true;
                currentTouch.isNew = true;
                currentTouch.duration = 0;
                currentTouch.lastTime = currentTime;
                currentTouch.x = x;
                currentTouch.y = y;
                currentTouch.lastX = x;
                currentTouch.lastY = y;
                currentTouch.velocityX = 0;
                currentTouch.velocityY = 0;
                currentTouch.pressure = pressure;
            } else {
                // Continue existing touch
                currentTouch.isNew = false;
                currentTouch.duration += currentTime - currentTouch.lastTime;
                
                // Calculate velocity
                float deltaTime = (currentTime - currentTouch.lastTime) / 1000.0f;
                if (deltaTime > 0) {
                    currentTouch.velocityX = (x - currentTouch.lastX) / deltaTime;
                    currentTouch.velocityY = (y - currentTouch.lastY) / deltaTime;
                }
                
                // Update position
                currentTouch.lastX = currentTouch.x;
                currentTouch.lastY = currentTouch.y;
                currentTouch.x = x;
                currentTouch.y = y;
                currentTouch.lastTime = currentTime;
                currentTouch.pressure = pressure;
            }
            
            // Update gesture detection
            updateGestures();
        } else if (touchActive) {
            // Touch released
            touchActive = false;
            finalizeGesture();
        }
    }
    
private:
    // Detect and track gestures
    void updateGestures() {
        if (!gestureInProgress) {
            // Check for start of new gesture
            if (currentTouch.isNew) {
                gestureInProgress = true;
                currentGesture = GESTURE_NONE;
            }
        } else {
            // Update existing gesture
            if (currentTouch.duration >= HOLD_DURATION) {
                // Hold gesture detected
                if (currentGesture == GESTURE_NONE) {
                    currentGesture = GESTURE_HOLD;
                    onGestureDetected(GESTURE_HOLD);
                }
            } else {
                // Check for swipe
                int16_t deltaX = currentTouch.x - currentTouch.lastX;
                int16_t deltaY = currentTouch.y - currentTouch.lastY;
                float distance = sqrt(deltaX*deltaX + deltaY*deltaY);
                
                if (distance >= SWIPE_THRESHOLD) {
                    // Determine swipe direction
                    float angle = atan2(deltaY, deltaX) * 180.0f / M_PI;
                    
                    // Convert angle to swipe direction
                    // Use 45-degree sectors for direction detection
                    if (angle >= -22.5f && angle < 22.5f) {
                        currentGesture = GESTURE_SWIPE_RIGHT;
                    } else if (angle >= 22.5f && angle < 67.5f) {
                        currentGesture = GESTURE_SWIPE_DOWN_RIGHT;
                    } else if (angle >= 67.5f && angle < 112.5f) {
                        currentGesture = GESTURE_SWIPE_DOWN;
                    } else if (angle >= 112.5f && angle < 157.5f) {
                        currentGesture = GESTURE_SWIPE_DOWN_LEFT;
                    } else if (angle >= 157.5f || angle < -157.5f) {
                        currentGesture = GESTURE_SWIPE_LEFT;
                    } else if (angle >= -157.5f && angle < -112.5f) {
                        currentGesture = GESTURE_SWIPE_UP_LEFT;
                    } else if (angle >= -112.5f && angle < -67.5f) {
                        currentGesture = GESTURE_SWIPE_UP;
                    } else {
                        currentGesture = GESTURE_SWIPE_UP_RIGHT;
                    }
                    
                    onGestureDetected(currentGesture);
                }
            }
        }
    }
    
    // Finalize gesture when touch is released
    void finalizeGesture() {
        if (gestureInProgress) {
            if (currentGesture == GESTURE_NONE) {
                // Check for tap
                float distance = sqrt(
                    pow(currentTouch.x - currentTouch.lastX, 2) +
                    pow(currentTouch.y - currentTouch.lastY, 2)
                );
                
                if (distance <= TAP_THRESHOLD &&
                    currentTouch.duration <= TAP_DURATION) {
                    onGestureDetected(GESTURE_TAP);
                }
            }
            
            gestureInProgress = false;
            currentGesture = GESTURE_NONE;
        }
    }
};
```

### Touch Calibration

```cpp
// Class to handle touch screen calibration
class TouchCalibration {
private:
    // Calibration points in screen coordinates
    static const uint8_t NUM_POINTS = 3;
    Point screenPoints[NUM_POINTS] = {
        {50, 50},                    // Top-left
        {WIDTH - 50, HEIGHT / 2},    // Middle-right
        {WIDTH / 2, HEIGHT - 50}     // Bottom-center
    };
    
    // Measured raw values from touch controller
    Point rawPoints[NUM_POINTS];
    
    // Calibration matrix
    // Transforms raw touch coordinates to screen coordinates
    float matrix[6];
    
public:
    // Perform touch calibration process
    bool calibrate() {
        // Draw calibration points and collect measurements
        for (int i = 0; i < NUM_POINTS; i++) {
            // Draw target point
            display.fillCircle(
                screenPoints[i].x,
                screenPoints[i].y,
                5,                     // Outer circle radius
                COLOR_RED
            );
            display.drawCircle(
                screenPoints[i].x,
                screenPoints[i].y,
                10,                    // Inner circle radius
                COLOR_WHITE
            );
            
            // Wait for touch
            while (!readRawTouch(&rawPoints[i].x, &rawPoints[i].y)) {
                delay(10);
            }
            
            // Wait for release
            while (readRawTouch(nullptr, nullptr)) {
                delay(10);
            }
            
            delay(500);  // Debounce delay
        }
        
        // Calculate calibration matrix
        return calculateMatrix();
    }
    
private:
    // Calculate calibration matrix using 3-point calibration
    bool calculateMatrix() {
        // Calculate matrix coefficients
        // Using least squares method to solve:
        // screen_x = a*raw_x + b*raw_y + c
        // screen_y = d*raw_x + e*raw_y + f
        
        float dx1 = rawPoints[1].x - rawPoints[0].x;
        float dx2 = rawPoints[2].x - rawPoints[0].x;
        float dy1 = rawPoints[1].y - rawPoints[0].y;
        float dy2 = rawPoints[2].y - rawPoints[0].y;
        float sx1 = screenPoints[1].x - screenPoints[0].x;
        float sx2 = screenPoints[2].x - screenPoints[0].x;
        float sy1 = screenPoints[1].y - screenPoints[0].y;
        float sy2 = screenPoints[2].y - screenPoints[0].y;
        
        // Calculate determinant
        float det = dx1 * dy2 - dx2 * dy1;
        if (abs(det) < 0.1f) {
            // Invalid calibration points
            return false;
        }
        
        // Calculate matrix elements
        matrix[0] = (sx1 * dy2 - sx2 * dy1) / det;
        matrix[1] = (dx1 * sx2 - dx2 * sx1) / det;
        matrix[2] = screenPoints[0].x -
                   matrix[0] * rawPoints[0].x -
                   matrix[1] * rawPoints[0].y;
        
        matrix[3] = (sy1 * dy2 - sy2 * dy1) / det;
        matrix[4] = (dx1 * sy2 - dx2 * sy1) / det;
        matrix[5] = screenPoints[0].y -
                   matrix[3] * rawPoints[0].x -
                   matrix[4] * rawPoints[0].y;
        
        return true;
    }
    
    // Transform raw coordinates using calibration matrix
    void transformPoint(uint16_t* x, uint16_t* y) {
        float raw_x = *x;
        float raw_y = *y;
        
        *x = matrix[0] * raw_x + matrix[1] * raw_y + matrix[2];
        *y = matrix[3] * raw_x + matrix[4] * raw_y + matrix[5];
    }
};
```

The touch input system demonstrates several important concepts:

1. Event Processing
   - Raw touch data acquisition
   - Touch state tracking
   - Event timing and duration calculation
   - Velocity and movement tracking

2. Gesture Recognition
   - Tap detection with movement and duration thresholds
   - Swipe detection with direction calculation
   - Hold gesture timing
   - Multi-directional swipe classification

3. Touch Calibration
   - 3-point calibration process
   - Matrix transformation calculation
   - Linear coordinate mapping
   - Error detection and validation

4. Performance Optimization
   - Efficient gesture detection algorithms
   - Minimal floating-point calculations
   - State-based processing to reduce calculations
   - Early exit conditions for invalid states

## Sprite Animation and Transformation

### Sprite Animation System

```cpp
// Structure to hold sprite frame data
struct SpriteFrame {
    // Frame dimensions
    uint16_t width;      // Frame width in pixels
    uint16_t height;     // Frame height in pixels
    
    // Frame timing
    uint16_t duration;   // Display time in milliseconds
    
    // Memory management
    uint16_t* data;      // Pointer to pixel data
    uint32_t dataSize;   // Size in bytes (width * height * 2)
    
    // Frame properties
    uint8_t alpha;       // Global alpha for frame (0-255)
    bool useChroma;      // Whether to use chroma keying
    uint16_t chromaKey;  // Color to treat as transparent
};

// Class to manage animated sprites with transformations
class AnimatedSprite {
private:
    // Animation properties
    SpriteFrame* frames;         // Array of animation frames
    uint8_t frameCount;          // Total number of frames
    uint8_t currentFrame;        // Current frame index
    uint32_t frameTimer;         // Time in current frame
    bool looping;               // Whether animation loops
    
    // Transform properties
    float scale;                // Current scale factor
    float rotation;             // Current rotation in degrees
    float pivotX, pivotY;       // Rotation pivot point
    
    // Memory optimization
    uint16_t* renderBuffer;     // Temporary buffer for transforms
    bool isDirty;              // Whether transform needs update
    
public:
    // Initialize sprite with frame data
    AnimatedSprite(uint8_t numFrames) {
        frameCount = numFrames;
        frames = new SpriteFrame[numFrames];
        currentFrame = 0;
        frameTimer = 0;
        looping = true;
        
        // Initialize transform properties
        scale = 1.0f;
        rotation = 0.0f;
        pivotX = pivotY = 0.5f;  // Center pivot by default
        
        // Allocate render buffer
        // Size based on maximum frame dimensions
        uint16_t maxWidth = 0;
        uint16_t maxHeight = 0;
        for (int i = 0; i < numFrames; i++) {
            maxWidth = max(maxWidth, frames[i].width);
            maxHeight = max(maxHeight, frames[i].height);
        }
        
        // Add 1 pixel border for rotation overflow
        uint32_t bufferSize = (maxWidth + 2) * (maxHeight + 2);
        renderBuffer = new uint16_t[bufferSize];
        isDirty = true;
    }
    
    // Update animation state
    void update(uint32_t deltaTime) {
        if (!looping && currentFrame >= frameCount - 1) {
            return;  // Animation complete
        }
        
        frameTimer += deltaTime;
        
        // Check for frame advance
        if (frameTimer >= frames[currentFrame].duration) {
            frameTimer -= frames[currentFrame].duration;
            currentFrame = (currentFrame + 1) % frameCount;
            isDirty = true;  // Need to update transform
        }
    }
    
    // Set sprite transformation
    void setTransform(float newScale, float newRotation,
                     float newPivotX, float newPivotY) {
        // Only mark dirty if values actually change
        if (newScale != scale || newRotation != rotation ||
            newPivotX != pivotX || newPivotY != pivotY) {
            scale = newScale;
            rotation = newRotation;
            pivotX = newPivotX;
            pivotY = newPivotY;
            isDirty = true;
        }
    }
    
    // Render sprite with current transform
    void render(int16_t x, int16_t y) {
        SpriteFrame& frame = frames[currentFrame];
        
        if (isDirty) {
            // Update transform
            updateTransform(frame);
            isDirty = false;
        }
        
        // Calculate display coordinates
        int16_t displayX = x - (frame.width * scale * pivotX);
        int16_t displayY = y - (frame.height * scale * pivotY);
        
        // Draw transformed sprite
        display.pushImage(
            displayX, displayY,
            frame.width, frame.height,
            renderBuffer
        );
    }
    
private:
    // Apply current transformation to frame
    void updateTransform(const SpriteFrame& frame) {
        // Clear render buffer
        memset(renderBuffer, 0, frame.dataSize);
        
        // Calculate transform matrix
        float cosA = cos(rotation * M_PI / 180.0f);
        float sinA = sin(rotation * M_PI / 180.0f);
        
        // Calculate pivot point in pixels
        float px = frame.width * pivotX;
        float py = frame.height * pivotY;
        
        // Transform each pixel
        for (int y = 0; y < frame.height; y++) {
            for (int x = 0; x < frame.width; x++) {
                // Center coordinates on pivot
                float dx = (x - px) * scale;
                float dy = (y - py) * scale;
                
                // Apply rotation
                float rx = dx * cosA - dy * sinA + px;
                float ry = dx * sinA + dy * cosA + py;
                
                // Check if transformed point is in bounds
                if (rx >= 0 && rx < frame.width &&
                    ry >= 0 && ry < frame.height) {
                    // Sample source pixel
                    uint16_t color = frame.data[
                        ((int)ry * frame.width) + (int)rx
                    ];
                    
                    // Apply chroma key if enabled
                    if (!frame.useChroma ||
                        color != frame.chromaKey) {
                        // Apply alpha blending if needed
                        if (frame.alpha < 255) {
                            uint16_t bg = renderBuffer[
                                y * frame.width + x
                            ];
                            color = blendColors(
                                color, bg, frame.alpha
                            );
                        }
                        
                        // Write to render buffer
                        renderBuffer[y * frame.width + x] = color;
                    }
                }
            }
        }
    }
    
    // Blend two RGB565 colors using alpha
    uint16_t blendColors(uint16_t fg, uint16_t bg, uint8_t alpha) {
        // Extract components
        uint8_t fgR = (fg >> 11) & 0x1F;
        uint8_t fgG = (fg >> 5) & 0x3F;
        uint8_t fgB = fg & 0x1F;
        
        uint8_t bgR = (bg >> 11) & 0x1F;
        uint8_t bgG = (bg >> 5) & 0x3F;
        uint8_t bgB = bg & 0x1F;
        
        // Blend components
        float a = alpha / 255.0f;
        uint8_t r = fgR * a + bgR * (1 - a);
        uint8_t g = fgG * a + bgG * (1 - a);
        uint8_t b = fgB * a + bgB * (1 - a);
        
        // Reconstruct RGB565
        return (r << 11) | (g << 5) | b;
    }
};
```

### Sprite Optimization Techniques

```cpp
// Class to manage sprite memory and reuse
class SpritePool {
private:
    // Pool configuration
    static const uint16_t MAX_SPRITES = 32;    // Maximum sprites in pool
    static const uint32_t MAX_MEMORY = 1024*1024;  // 1MB total memory
    
    // Sprite tracking
    struct PoolEntry {
        uint16_t* data;          // Sprite pixel data
        uint32_t size;           // Size in bytes
        uint32_t lastUsed;       // Last access timestamp
        bool inUse;             // Whether sprite is active
    };
    
    PoolEntry pool[MAX_SPRITES];
    uint32_t totalMemory;       // Current memory usage
    
public:
    // Initialize empty pool
    SpritePool() : totalMemory(0) {
        memset(pool, 0, sizeof(pool));
    }
    
    // Allocate sprite memory
    uint16_t* allocate(uint32_t size) {
        uint32_t oldestTime = UINT32_MAX;
        int oldestIndex = -1;
        
        // Try to find unused entry
        for (int i = 0; i < MAX_SPRITES; i++) {
            if (!pool[i].inUse) {
                // Found unused entry
                if (pool[i].data && pool[i].size >= size) {
                    // Can reuse existing allocation
                    pool[i].inUse = true;
                    pool[i].lastUsed = millis();
                    return pool[i].data;
                }
                oldestIndex = i;
                break;
            }
            // Track oldest used entry
            if (pool[i].lastUsed < oldestTime) {
                oldestTime = pool[i].lastUsed;
                oldestIndex = i;
            }
        }
        
        if (oldestIndex < 0) {
            return nullptr;  // Pool full
        }
        
        // Free old entry if needed
        if (pool[oldestIndex].data) {
            totalMemory -= pool[oldestIndex].size;
            delete[] pool[oldestIndex].data;
        }
        
        // Check memory limit
        if (totalMemory + size > MAX_MEMORY) {
            // Need to free some memory
            // Sort by last used time
            struct SortEntry {
                int index;
                uint32_t time;
            };
            SortEntry sorted[MAX_SPRITES];
            int sortCount = 0;
            
            for (int i = 0; i < MAX_SPRITES; i++) {
                if (pool[i].inUse) {
                    sorted[sortCount].index = i;
                    sorted[sortCount].time = pool[i].lastUsed;
                    sortCount++;
                }
            }
            
            // Sort oldest first
            for (int i = 0; i < sortCount-1; i++) {
                for (int j = i+1; j < sortCount; j++) {
                    if (sorted[j].time < sorted[i].time) {
                        SortEntry temp = sorted[i];
                        sorted[i] = sorted[j];
                        sorted[j] = temp;
                    }
                }
            }
            
            // Free sprites until we have enough memory
            for (int i = 0; i < sortCount &&
                 totalMemory + size > MAX_MEMORY; i++) {
                int index = sorted[i].index;
                totalMemory -= pool[index].size;
                delete[] pool[index].data;
                pool[index].data = nullptr;
                pool[index].size = 0;
                pool[index].inUse = false;
            }
        }
        
        // Allocate new memory
        pool[oldestIndex].data = new uint16_t[size/2];
        pool[oldestIndex].size = size;
        pool[oldestIndex].lastUsed = millis();
        pool[oldestIndex].inUse = true;
        totalMemory += size;
        
        return pool[oldestIndex].data;
    }
    
    // Release sprite memory
    void release(uint16_t* data) {
        for (int i = 0; i < MAX_SPRITES; i++) {
            if (pool[i].data == data) {
                pool[i].inUse = false;
                return;
            }
        }
    }
};
```

The sprite animation system demonstrates several key concepts:

1. Animation Management
   - Frame-based animation system
   - Timing control and frame duration
   - Looping and non-looping animations
   - Memory-efficient frame storage

2. Sprite Transformation
   - Scale and rotation transforms
   - Pivot point handling
   - Pixel-perfect transformation
   - Efficient dirty state tracking

3. Memory Optimization
   - Sprite pooling for memory reuse
   - Least-recently-used eviction
   - Memory limit enforcement
   - Efficient memory allocation

4. Performance Techniques
   - Pre-allocated render buffers
   - Minimal recalculation of transforms
   - Efficient pixel blending
   - Smart memory management

## Text Rendering System

### Font Management

```cpp
// Structure to hold glyph metrics and data
struct GlyphInfo {
    uint8_t width;       // Glyph width in pixels
    uint8_t height;      // Glyph height in pixels
    int8_t xAdvance;    // Horizontal advance in pixels
    int8_t xOffset;     // X offset from cursor position
    int8_t yOffset;     // Y offset from baseline
    uint32_t dataOffset; // Offset into font bitmap data
};

// Class to manage font rendering
class FontManager {
private:
    // Font data storage
    const uint8_t* fontData;     // Raw font bitmap data
    GlyphInfo* glyphs;          // Array of glyph information
    uint8_t firstChar;          // First character code
    uint8_t lastChar;           // Last character code
    uint8_t height;            // Font height in pixels
    int8_t baseline;           // Baseline position
    
    // Rendering properties
    uint16_t textColor;        // Current text color (RGB565)
    uint16_t bgColor;          // Background color (RGB565)
    bool transparent;          // Whether to draw background
    float scale;              // Text scaling factor
    
public:
    // Initialize font with data and metrics
    FontManager(const uint8_t* data, uint8_t first, uint8_t last,
                uint8_t fontHeight, int8_t baselineOffset)
        : fontData(data)
        , firstChar(first)
        , lastChar(last)
        , height(fontHeight)
        , baseline(baselineOffset)
        , textColor(0xFFFF)     // White default color
        , bgColor(0x0000)       // Black background
        , transparent(true)     // Transparent by default
        , scale(1.0f)          // No scaling
    {
        // Allocate and initialize glyph array
        uint8_t numChars = lastChar - firstChar + 1;
        glyphs = new GlyphInfo[numChars];
        
        // Parse font data header and populate glyph info
        uint32_t offset = 0;
        for (uint8_t i = 0; i < numChars; i++) {
            // Read metrics from font data
            glyphs[i].width = fontData[offset++];
            glyphs[i].height = fontData[offset++];
            glyphs[i].xAdvance = (int8_t)fontData[offset++];
            glyphs[i].xOffset = (int8_t)fontData[offset++];
            glyphs[i].yOffset = (int8_t)fontData[offset++];
            glyphs[i].dataOffset = offset;
            
            // Skip bitmap data to next glyph
            // Each row is packed into bytes
            uint16_t bytesPerRow = (glyphs[i].width + 7) / 8;
            offset += bytesPerRow * glyphs[i].height;
        }
    }
    
    // Draw text string at specified position
    void drawText(int16_t x, int16_t y, const char* text) {
        y += baseline * scale;  // Adjust for baseline
        
        while (*text) {
            char c = *text++;
            if (c < firstChar || c > lastChar) {
                continue;  // Skip invalid chars
            }
            
            // Get glyph info and calculate position
            const GlyphInfo& glyph = glyphs[c - firstChar];
            int16_t glyphX = x + glyph.xOffset * scale;
            int16_t glyphY = y + glyph.yOffset * scale;
            
            // Draw scaled glyph
            drawGlyph(glyphX, glyphY, glyph);
            
            // Advance cursor
            x += glyph.xAdvance * scale;
        }
    }
    
private:
    // Draw single glyph with current settings
    void drawGlyph(int16_t x, int16_t y, const GlyphInfo& glyph) {
        const uint8_t* glyphData = &fontData[glyph.dataOffset];
        
        // Calculate scaled dimensions
        uint16_t scaledWidth = glyph.width * scale;
        uint16_t scaledHeight = glyph.height * scale;
        
        // Draw background if not transparent
        if (!transparent) {
            display.fillRect(
                x, y,
                scaledWidth, scaledHeight,
                bgColor
            );
        }
        
        // Draw glyph bitmap
        uint8_t mask = 0x80;  // Start with leftmost bit
        uint16_t byteOffset = 0;
        
        for (uint16_t py = 0; py < scaledHeight; py++) {
            // Calculate source y position
            float srcY = py / scale;
            uint16_t srcYi = (uint16_t)srcY;
            
            for (uint16_t px = 0; px < scaledWidth; px++) {
                // Calculate source x position
                float srcX = px / scale;
                uint16_t srcXi = (uint16_t)srcX;
                
                // Calculate bit position in font data
                uint16_t bitOffset = srcYi * glyph.width + srcXi;
                if (bitOffset / 8 != byteOffset) {
                    byteOffset = bitOffset / 8;
                    mask = 0x80 >> (bitOffset & 7);
                }
                
                // Draw pixel if bit is set
                if (glyphData[byteOffset] & mask) {
                    display.drawPixel(x + px, y + py, textColor);
                }
                
                // Move to next bit
                mask >>= 1;
                if (!mask) {
                    mask = 0x80;  // Reset to leftmost bit
                    byteOffset++; // Move to next byte
                }
            }
        }
    }
};
```

The font rendering system demonstrates several key concepts:

1. Font Data Organization
   - Efficient bitmap font storage
   - Glyph metrics for proper positioning
   - Baseline and offset handling
   - Memory-efficient bit packing

2. Rendering Features
   - Scalable text rendering
   - Background transparency
   - Color control
   - Proper text alignment

3. Performance Optimization
   - Minimal memory allocation
   - Efficient bit operations
   - Smart byte access
   - Cached glyph metrics