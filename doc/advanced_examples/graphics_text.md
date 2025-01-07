# Advanced Graphics and Text Rendering

This guide demonstrates advanced graphics and text rendering techniques using LovyanGFX's sprite system and hardware acceleration features.

## Graphics Buffer Management

The `GraphicsBuffer` class provides a high-level abstraction for managing display buffers:

```cpp
class GraphicsBuffer {
private:
    // Core display reference for hardware operations
    LGFX* _display;
    
    // Main sprite buffer for rendering
    // This acts as a double-buffer to prevent tearing
    LGFX_Sprite* _buffer;
    
    // Configuration flags
    bool _use_psram;        // Whether to use PSRAM for large buffers
    uint8_t _rotation;      // Current rotation (0-7, includes mirroring)
    uint8_t _color_depth;   // Color depth in bits (8, 16, or 24)

public:
    // Constructor with memory management options
    // Parameters:
    // - display: Pointer to LGFX display instance
    // - use_psram: Whether to allocate buffer in PSRAM (default: false)
    // - color_depth: Color depth in bits (default: 16 for RGB565)
    GraphicsBuffer(LGFX* display, bool use_psram = false, uint8_t color_depth = 16)
    : _display(display)
    , _use_psram(use_psram)
    , _rotation(0)
    , _color_depth(color_depth)
    {
        // Create and configure the sprite buffer
        _buffer = new LGFX_Sprite(display);
        
        // Configure PSRAM usage if requested
        // PSRAM allows for larger buffers but is slower than internal RAM
        _buffer->setPsram(_use_psram);
        
        // Set color depth (affects memory usage):
        // - 8-bit: 256 colors, 1 byte per pixel
        // - 16-bit: 65,536 colors (RGB565), 2 bytes per pixel
        // - 24-bit: 16.7M colors (RGB888), 3 bytes per pixel
        _buffer->setColorDepth(_color_depth);
        
        // Create buffer matching display dimensions
        // Memory required = width * height * (color_depth / 8)
        _buffer->createSprite(display->width(), display->height());
    }

    // Set buffer rotation and handle mirroring
    // Parameter r: 0-7 where:
    // - 0-3: 0°, 90°, 180°, 270° rotation
    // - 4-7: Same as 0-3 but with horizontal mirroring
    void setRotation(uint8_t r) {
        _rotation = r % 8;  // Ensure valid rotation value
        _buffer->setRotation(_rotation);
    }

    // Clear buffer with specified color
    // Uses 24-bit color format (0x00RRGGBB)
    void clear(uint32_t color = TFT_BLACK) {
        _buffer->fillScreen(color);
    }

    // Push buffer contents to display
    // This completes the double-buffering process
    void render() {
        _buffer->pushSprite(0, 0);
    }

    // Get direct access to sprite buffer
    // Useful for advanced rendering operations
    LGFX_Sprite* buffer() { return _buffer; }
};
```

## Advanced Text Rendering

The `TextRenderer` class provides sophisticated text effects:

```cpp
class TextRenderer {
private:
    LGFX_Sprite* _target;           // Target sprite for rendering
    float _text_size;               // Text scaling factor
    uint32_t _text_color;          // Primary text color (24-bit RGB)
    const lgfx::GFXfont* _font;    // Current font face

public:
    TextRenderer(LGFX_Sprite* target)
    : _target(target)
    , _text_size(1.0f)
    , _text_color(TFT_WHITE)
    , _font(nullptr)
    {
    }

    // Set custom font for rendering
    // Supports standard GFX fonts and custom font files
    void setFont(const lgfx::GFXfont* font) {
        _font = font;
        if (_font) {
            _target->setFont(_font);
        }
    }

    // Set text scaling factor
    // Values > 1.0 increase size, < 1.0 decrease size
    void setTextSize(float size) {
        _text_size = size;
        _target->setTextSize(size);
    }

    // Set primary text color in 24-bit RGB format
    void setTextColor(uint32_t color) {
        _text_color = color;
        _target->setTextColor(color);
    }

    // Draw text with drop shadow effect
    // Parameters:
    // - text: String to render
    // - x, y: Text position
    // - shadow_color: Shadow color (24-bit RGB)
    // - offset_x, offset_y: Shadow offset in pixels
    void drawTextWithShadow(const char* text, int32_t x, int32_t y, 
                           uint32_t shadow_color = TFT_DARKGREY, 
                           int8_t offset_x = 2, 
                           int8_t offset_y = 2) 
    {
        // Draw shadow first (underneath main text)
        _target->setTextColor(shadow_color);
        _target->drawString(text, x + offset_x, y + offset_y);
        
        // Draw main text on top
        _target->setTextColor(_text_color);
        _target->drawString(text, x, y);
    }

    // Draw text with outline effect
    // Parameters:
    // - text: String to render
    // - x, y: Text position
    // - outline_color: Outline color (24-bit RGB)
    // - thickness: Outline thickness in pixels
    void drawTextWithOutline(const char* text, int32_t x, int32_t y,
                            uint32_t outline_color = TFT_BLACK,
                            uint8_t thickness = 1)
    {
        // Draw outline by offsetting text in all directions
        for (int8_t i = -thickness; i <= thickness; i++) {
            for (int8_t j = -thickness; j <= thickness; j++) {
                // Skip center position (main text location)
                if (i == 0 && j == 0) continue;
                
                // Draw outline segment
                _target->setTextColor(outline_color);
                _target->drawString(text, x + i, y + j);
            }
        }
        
        // Draw main text in center
        _target->setTextColor(_text_color);
        _target->drawString(text, x, y);
    }

    // Draw text with vertical color gradient
    // Parameters:
    // - text: String to render
    // - x, y: Text position
    // - color1: Start color (top, 24-bit RGB)
    // - color2: End color (bottom, 24-bit RGB)
    void drawTextWithGradient(const char* text, int32_t x, int32_t y,
                             uint32_t color1, uint32_t color2)
    {
        // Calculate text dimensions
        int16_t w = _target->textWidth(text);    // Width in pixels
        int16_t h = _target->fontHeight();       // Height in pixels
        
        // Create temporary sprite for gradient mask
        LGFX_Sprite temp(_target);
        temp.createSprite(w, h);
        temp.setFont(_font);
        temp.setTextSize(_text_size);
        
        // Create text mask (white text on black)
        temp.fillScreen(TFT_BLACK);
        temp.setTextColor(TFT_WHITE);
        temp.drawString(text, 0, 0);
        
        // Apply gradient to masked text
        for (int16_t py = 0; py < h; py++) {
            // Calculate gradient progress (0.0 - 1.0)
            float progress = (float)py / h;
            
            // Interpolate RGB components
            uint8_t r = lerp(color1 >> 16, color2 >> 16, progress);
            uint8_t g = lerp((color1 >> 8) & 0xFF, (color2 >> 8) & 0xFF, progress);
            uint8_t b = lerp(color1 & 0xFF, color2 & 0xFF, progress);
            
            // Combine into 24-bit color
            uint32_t color = (r << 16) | (g << 8) | b;
            
            // Apply color to masked pixels
            for (int16_t px = 0; px < w; px++) {
                if (temp.readPixel(px, py)) {
                    _target->drawPixel(x + px, y + py, color);
                }
            }
        }
    }

private:
    // Linear interpolation helper for gradient calculation
    // Returns value between a and b based on t (0.0 - 1.0)
    uint8_t lerp(uint8_t a, uint8_t b, float t) {
        return a + (b - a) * t;
    }
};
```

## Advanced Shape Drawing

The `ShapeRenderer` class provides sophisticated shape rendering with effects:

```cpp
class ShapeRenderer {
private:
    // Target sprite for rendering operations
    LGFX_Sprite* _target;

public:
    ShapeRenderer(LGFX_Sprite* target) : _target(target) {}

    // Draw a rounded rectangle with gradient fill
    // Parameters:
    // - x, y: Top-left corner position
    // - w, h: Width and height of rectangle
    // - radius: Corner radius in pixels
    // - color1: Start color of gradient (24-bit RGB)
    // - color2: End color of gradient (24-bit RGB)
    // - vertical: If true, gradient runs top-to-bottom, else left-to-right
    void drawRoundedRectWithGradient(int32_t x, int32_t y, int32_t w, int32_t h,
                                    int32_t radius, uint32_t color1, uint32_t color2,
                                    bool vertical = true)
    {
        // Draw gradient fill first
        for (int32_t i = 0; i < (vertical ? h : w); i++) {
            // Calculate gradient progress (0.0 - 1.0)
            float progress = (float)i / (vertical ? h : w);
            
            // Interpolate RGB components
            // Extract RGB components using bit shifting
            uint8_t r = lerp(color1 >> 16, color2 >> 16, progress);
            uint8_t g = lerp((color1 >> 8) & 0xFF, (color2 >> 8) & 0xFF, progress);
            uint8_t b = lerp(color1 & 0xFF, color2 & 0xFF, progress);
            
            // Combine components into 24-bit color
            uint32_t color = (r << 16) | (g << 8) | b;
            
            // Draw line segment based on gradient direction
            if (vertical) {
                // Horizontal line for vertical gradient
                _target->drawFastHLine(x, y + i, w, color);
            } else {
                // Vertical line for horizontal gradient
                _target->drawFastVLine(x + i, y, h, color);
            }
        }
        
        // Draw rounded corners over gradient
        drawRoundCorners(x, y, w, h, radius);
    }

    // Draw a circle with alternating pattern
    // Parameters:
    // - x0, y0: Center coordinates
    // - r: Radius in pixels
    // - fg_color: Foreground color for pattern (24-bit RGB)
    // - bg_color: Background color for pattern (24-bit RGB)
    // - pattern: 8-bit pattern (1 bits = fg_color, 0 bits = bg_color)
    void drawCircleWithPattern(int32_t x0, int32_t y0, int32_t r,
                              uint32_t fg_color, uint32_t bg_color,
                              uint8_t pattern = 0xAA)  // 0xAA = 10101010 binary
    {
        // Initialize Bresenham's circle algorithm variables
        int32_t f = 1 - r;           // Decision parameter
        int32_t ddF_x = 1;           // x-increment
        int32_t ddF_y = -2 * r;      // y-increment
        int32_t x = 0;               // Current x offset from center
        int32_t y = r;               // Current y offset from center
        
        // Track pattern position (0-7)
        uint8_t pattern_pos = 0;
        
        // Lambda function to draw pattern pixel
        auto drawPixelPattern = [&](int32_t x, int32_t y) {
            // Get color based on current pattern bit
            uint32_t color = (pattern & (1 << pattern_pos)) ? fg_color : bg_color;
            _target->drawPixel(x, y, color);
            // Move to next pattern bit (0-7)
            pattern_pos = (pattern_pos + 1) & 7;
        };
        
        // Draw initial points at 90-degree intervals
        drawPixelPattern(x0, y0 + r);      // Bottom
        drawPixelPattern(x0, y0 - r);      // Top
        drawPixelPattern(x0 + r, y0);      // Right
        drawPixelPattern(x0 - r, y0);      // Left
        
        // Bresenham's circle algorithm
        while (x < y) {
            // Update decision parameter and coordinates
            if (f >= 0) {
                y--;                // Move up
                ddF_y += 2;        // Update y-increment
                f += ddF_y;        // Update decision parameter
            }
            x++;                   // Move right
            ddF_x += 2;            // Update x-increment
            f += ddF_x;            // Update decision parameter
            
            // Draw 8 symmetric points
            // This completes the circle using 8-way symmetry
            drawPixelPattern(x0 + x, y0 + y);  // Octant 1
            drawPixelPattern(x0 - x, y0 + y);  // Octant 4
            drawPixelPattern(x0 + x, y0 - y);  // Octant 8
            drawPixelPattern(x0 - x, y0 - y);  // Octant 5
            drawPixelPattern(x0 + y, y0 + x);  // Octant 2
            drawPixelPattern(x0 - y, y0 + x);  // Octant 3
            drawPixelPattern(x0 + y, y0 - x);  // Octant 7
            drawPixelPattern(x0 - y, y0 - x);  // Octant 6
        }
    }

private:
    // Helper function to draw rounded corners
    // Uses anti-aliasing for smooth appearance
    void drawRoundCorners(int32_t x, int32_t y, int32_t w, int32_t h, int32_t r) {
        // Implementation of rounded corner drawing
        // This would include anti-aliasing calculations
        // and proper corner radius handling
    }

    // Linear interpolation for gradient calculations
    // Parameters:
    // - a: Start value
    // - b: End value
    // - t: Interpolation factor (0.0 - 1.0)
    uint8_t lerp(uint8_t a, uint8_t b, float t) {
        return a + (b - a) * t;
    }
};
```

## Usage Examples

### Advanced Text Effects

This example demonstrates various text rendering techniques:

```cpp
// Initialize core components
LGFX display;                           // Display driver
GraphicsBuffer graphics(&display);      // Double-buffered graphics
TextRenderer text(graphics.buffer());   // Text rendering system
ShapeRenderer shapes(graphics.buffer()); // Shape rendering system

void setup() {
    // Initialize display hardware
    display.init();
    
    // Set landscape orientation for wider viewing area
    display.setRotation(1);
    
    // Set mid-level brightness to save power
    // Range: 0 (off) to 255 (maximum)
    display.setBrightness(128);
    
    // Clear buffer with black background
    // This prevents artifacts from previous content
    graphics.clear(TFT_BLACK);
    
    // Draw text with drop shadow effect
    // Size 2.0 = double the base font size
    text.setTextSize(2.0f);
    text.setTextColor(TFT_WHITE);
    text.drawTextWithShadow(
        "Shadow Text",    // Text to render
        50, 50,          // X, Y position
        TFT_DARKGREY,    // Shadow color
        2, 2             // Shadow offset (2 pixels right and down)
    );
    
    // Draw text with outline effect
    // Size 3.0 = triple the base font size
    text.setTextSize(3.0f);
    text.drawTextWithOutline(
        "Outline",       // Text to render
        50, 100,         // X, Y position
        TFT_RED,         // Outline color
        2                // Outline thickness (2 pixels)
    );
    
    // Draw text with vertical gradient
    // Size 4.0 = quadruple the base font size
    text.setTextSize(4.0f);
    text.drawTextWithGradient(
        "Gradient",      // Text to render
        50, 150,         // X, Y position
        TFT_BLUE,        // Top color
        TFT_GREEN        // Bottom color
    );
    
    // Push graphics buffer to display
    graphics.render();
}
```

### Complex Graphics

This example shows advanced shape rendering with effects:

```cpp
void drawComplexGraphics() {
    // Clear buffer for new frame
    graphics.clear(TFT_BLACK);
    
    // Draw gradient-filled rounded rectangle
    // Creates a button-like effect
    shapes.drawRoundedRectWithGradient(
        20, 20,          // X, Y position
        200, 100,        // Width, Height
        10,              // Corner radius
        TFT_BLUE,        // Top color
        TFT_PURPLE,      // Bottom color
        true             // Vertical gradient
    );
    
    // Draw patterned circle
    // Creates a dashed outline effect
    shapes.drawCircleWithPattern(
        120, 160,        // Center X, Y
        50,              // Radius
        TFT_WHITE,       // Pattern foreground
        TFT_DARKGREY,    // Pattern background
        0xAA             // Pattern: 10101010 = alternating pixels
    );
    
    // Push graphics buffer to display
    graphics.render();
}
```

## Performance Optimization

1. Buffer Management
   ```cpp
   // Calculate buffer size before allocation
   size_t bufferSize = width * height * (colorDepth / 8);
   
   // Check available memory
   if (heap_caps_get_free_size(MALLOC_CAP_DMA) > bufferSize) {
       // Safe to create sprite
       sprite.createSprite(width, height);
   } else {
       // Handle insufficient memory
       // Options:
       // 1. Reduce color depth
       // 2. Reduce dimensions
       // 3. Use tiled rendering
   }
   ```

2. Drawing Optimization
   ```cpp
   // Batch similar operations in single transaction
   display.startWrite();
   {
       // Multiple drawing operations here
       // This reduces SPI overhead by keeping
       // the bus active between operations
   }
   display.endWrite();
   
   // Use hardware acceleration when available
   sprite.enableBackgroundDMA(true);  // Enable DMA
   ```

3. Memory Considerations
   ```cpp
   // Monitor sprite memory usage
   size_t spriteMemory = width * height * (colorDepth / 8);
   size_t freeMemory = heap_caps_get_free_size(MALLOC_CAP_DMA);
   
   // Clean up unused resources
   sprite.deleteSprite();  // Free sprite memory
   
   // Handle memory constraints
   if (freeMemory < threshold) {
       // Implement memory-saving strategies:
       // 1. Reduce sprite size
       // 2. Lower color depth
       // 3. Use partial updates
   }
   ```

4. Error Handling
   ```cpp
   // Validate parameters
   if (x < 0 || y < 0 || x >= width || y >= height) {
       return;  // Skip invalid coordinates
   }
   
   // Handle font loading errors
   if (!font) {
       // Fall back to built-in font
       setFont(&defaultFont);
   }
   
   // Implement graceful degradation
   if (!sprite.createSprite(width, height)) {
       // Fall back to direct drawing
       // or reduce quality settings
   }
   ```

These examples demonstrate the advanced graphics capabilities of LovyanGFX while maintaining good performance and memory management practices. 