# LovyanGFX Core Concepts

This guide explains the fundamental concepts and techniques used in LovyanGFX, with practical examples and detailed explanations.

## Fixed-Point Arithmetic

LovyanGFX uses fixed-point arithmetic for smooth animations and precise positioning. Fixed-point numbers are integers that represent fractional values by dedicating some bits to the fractional part. This approach is faster than floating-point math on microcontrollers.

```cpp
// In fixed-point arithmetic, we multiply regular numbers by 256 (shift left by 8)
// to store fractions as integers. For example:
// - 160.0 becomes 160 * 256 = 40960
// - 160.5 becomes 160.5 * 256 = 41088
int32_t position_x = 160 * 256;  // Represents 160.0 pixels on X axis
int32_t position_y = 120 * 256;  // Represents 120.0 pixels on Y axis

// Movement speed (velocity) is also in fixed-point format
// A value of 256 means 1 pixel per frame
int32_t delta_x = 0;  // No horizontal movement initially
int32_t delta_y = 0;  // No vertical movement initially

void updatePosition() {
    // Add velocity to position each frame
    // Since both numbers are in fixed-point format (multiplied by 256),
    // the result stays in fixed-point format
    position_x += delta_x;  // Update X position
    position_y += delta_y;  // Update Y position
    
    // To convert back to regular pixel coordinates for drawing,
    // we divide by 256 (shift right by 8 bits)
    // Example: 40960 >> 8 = 160 pixels
    int16_t screen_x = position_x >> 8;
    int16_t screen_y = position_y >> 8;
    
    // Draw at the calculated position
    lcd.drawCircle(screen_x, screen_y, 5, TFT_WHITE);
}

// Example: Smooth acceleration towards target
void moveTowardsTarget(int32_t target_x, int32_t target_y) {
    // Convert regular pixel coordinates to fixed-point
    // by multiplying by 256 (same format as position_x/y)
    target_x *= 256;
    target_y *= 256;
    
    // Gradually change velocity based on position difference
    // If target is to the right (target_x > position_x), increase velocity
    // If target is to the left (target_x < position_x), decrease velocity
    // This creates smooth acceleration/deceleration
    delta_x += (target_x > position_x) ? 1 : -1;
    delta_y += (target_y > position_y) ? 1 : -1;
}
```

## Double Buffering

Double buffering prevents screen tearing (visual artifacts where part of the old frame is visible with the new frame) by drawing to a hidden buffer while displaying the current frame:

```cpp
// Create two sprite buffers:
// - front_buffer: currently shown on screen
// - back_buffer: where we draw the next frame
LGFX_Sprite front_buffer(&lcd);  // Buffer being displayed
LGFX_Sprite back_buffer(&lcd);   // Buffer being drawn to

void initBuffers() {
    // Set color depth to 16 bits (RGB565 format)
    // This means:
    // - 5 bits for red   (32 levels)
    // - 6 bits for green (64 levels)
    // - 5 bits for blue  (32 levels)
    front_buffer.setColorDepth(16);
    back_buffer.setColorDepth(16);
    
    // Create sprites matching display dimensions
    // This allocates memory for each buffer
    front_buffer.createSprite(lcd.width(), lcd.height());
    back_buffer.createSprite(lcd.width(), lcd.height());
}

void render() {
    // Clear back buffer to black
    // Always start with a clean buffer to prevent artifacts
    back_buffer.fillScreen(TFT_BLACK);
    
    // Draw your frame content to back buffer
    // This ensures the user never sees incomplete frames
    drawScene(&back_buffer);
    
    // Start a single transaction for faster update
    // This keeps the display bus active during the entire operation
    lcd.startWrite();
    
    // Copy back buffer to display memory
    // This is when the frame becomes visible
    back_buffer.pushSprite(0, 0);
    
    // End transaction
    lcd.endWrite();
    
    // Optionally swap buffer pointers if you need
    // to preserve the front buffer content
    std::swap(front_buffer, back_buffer);
}
```

## Sprite Management

Efficient sprite handling for animations and UI elements. This system manages sprite memory and reuses sprites to prevent memory fragmentation:

```cpp
class SpriteManager {
private:
    // Structure to track sprite status
    struct ManagedSprite {
        LGFX_Sprite* sprite;     // Pointer to sprite object
        bool in_use;             // Whether sprite is currently being used
        uint32_t last_used;      // Timestamp of last usage (milliseconds)
    };
    
    // Pool of reusable sprites to prevent memory fragmentation
    std::vector<ManagedSprite> _sprite_pool;
    
    // Maximum memory allowed for all sprites combined
    const size_t _memory_limit;
    
public:
    SpriteManager(size_t memory_limit) 
    : _memory_limit(memory_limit) {
        // Create initial pool of sprites
        createSpritesPool();
    }
    
    LGFX_Sprite* acquireSprite(uint16_t width, uint16_t height) {
        // Calculate memory needed for this sprite
        // Each pixel uses 2 bytes in 16-bit color mode (RGB565)
        size_t required = width * height * 2;
        
        // Check if we have enough memory available
        size_t current_usage = calculateCurrentUsage();
        if (current_usage + required > _memory_limit) {
            // Try to free memory by releasing unused sprites
            cleanupUnusedSprites();
        }
        
        // Look for an available sprite in the pool
        for (auto& managed : _sprite_pool) {
            if (!managed.in_use) {
                // Found an unused sprite, configure it
                managed.sprite->createSprite(width, height);
                managed.in_use = true;
                managed.last_used = millis();  // Update timestamp
                return managed.sprite;
            }
        }
        
        // No sprites available
        return nullptr;
    }
    
    void releaseSprite(LGFX_Sprite* sprite) {
        // Find the sprite in our pool and mark it as available
        for (auto& managed : _sprite_pool) {
            if (managed.sprite == sprite) {
                managed.in_use = false;  // Mark as available
                return;
            }
        }
    }
    
private:
    void cleanupUnusedSprites() {
        uint32_t now = millis();
        for (auto& managed : _sprite_pool) {
            // Free sprites that haven't been used for 5 seconds
            // This prevents memory fragmentation by reusing sprites
            if (managed.in_use && 
                now - managed.last_used > 5000) {
                managed.sprite->deleteSprite();  // Free memory
                managed.in_use = false;          // Mark as available
            }
        }
    }
};
```

## Hardware Acceleration

Utilizing hardware features for better performance:

```cpp
void optimizeRendering() {
    // Enable DMA if available
    auto cfg = lcd.config();
    cfg.dma_channel = 1;
    cfg.use_psram = true;  // Use PSRAM for large buffers
    
    // Batch operations in single transaction
    lcd.startWrite();
    {
        // Multiple drawing operations
        // SPI bus stays active between calls
        drawBackground();
        drawSprites();
        drawUI();
    }
    lcd.endWrite();
}

// Optimize sprite rotations
void drawRotatedSprite(LGFX_Sprite* sprite, float angle) {
    // Use temporary buffer for rotation
    // This prevents artifacts and improves speed
    static LGFX_Sprite temp_buffer(&lcd);
    
    // Create rotation buffer if needed
    if (!temp_buffer.width()) {
        temp_buffer.setColorDepth(16);
        temp_buffer.createSprite(
            sprite->width(),  // Same size as source
            sprite->height()
        );
    }
    
    // Clear buffer (prevent artifacts)
    temp_buffer.fillScreen(TFT_BLACK);
    
    // Perform hardware-accelerated rotation
    sprite->pushRotateZoom(
        &temp_buffer,      // Target buffer
        sprite->width()/2, // Rotation center X
        sprite->height()/2,// Rotation center Y
        angle,            // Rotation angle
        1.0, 1.0,        // No scaling
        TFT_BLACK        // Transparent color
    );
    
    // Display rotated result
    temp_buffer.pushSprite(x, y);
}
```

## Memory Management

Efficient memory usage for resource-constrained devices:

```cpp
class MemoryManager {
public:
    // Calculate sprite memory requirements
    static size_t calculateSpriteMemory(
        uint16_t width,
        uint16_t height,
        uint8_t color_depth
    ) {
        // Memory per pixel:
        // - 8-bit: 1 byte
        // - 16-bit: 2 bytes
        // - 24-bit: 3 bytes
        uint8_t bytes_per_pixel = (color_depth + 7) / 8;
        return width * height * bytes_per_pixel;
    }
    
    // Check if sprite creation is safe
    static bool canCreateSprite(
        uint16_t width,
        uint16_t height,
        uint8_t color_depth
    ) {
        size_t required = calculateSpriteMemory(
            width, height, color_depth
        );
        
        // Check available memory
        // Leave 32KB buffer for stack/heap
        return heap_caps_get_free_size(MALLOC_CAP_DMA) >
               required + 32768;
    }
    
    // Create sprite with fallback options
    static LGFX_Sprite* createSafeSprite(
        LGFX* display,
        uint16_t width,
        uint16_t height,
        uint8_t color_depth = 16
    ) {
        LGFX_Sprite* sprite = new LGFX_Sprite(display);
        
        // Try creating with requested settings
        if (canCreateSprite(width, height, color_depth)) {
            sprite->setColorDepth(color_depth);
            sprite->createSprite(width, height);
            return sprite;
        }
        
        // Try reducing color depth
        if (color_depth > 8 &&
            canCreateSprite(width, height, 8)) {
            sprite->setColorDepth(8);
            sprite->createSprite(width, height);
            return sprite;
        }
        
        // Try reducing size
        uint16_t new_width = width * 0.75;
        uint16_t new_height = height * 0.75;
        if (canCreateSprite(new_width, new_height, 8)) {
            sprite->setColorDepth(8);
            sprite->createSprite(new_width, new_height);
            return sprite;
        }
        
        // Failed to create sprite
        delete sprite;
        return nullptr;
    }
};
```

## Error Handling

Robust error handling for reliable operation:

```cpp
class GraphicsError : public std::runtime_error {
public:
    GraphicsError(const char* msg) : std::runtime_error(msg) {}
};

class SafeGraphics {
public:
    // Safe sprite creation
    LGFX_Sprite* createSprite(uint16_t w, uint16_t h) {
        try {
            auto sprite = new LGFX_Sprite(&lcd);
            if (!sprite->createSprite(w, h)) {
                delete sprite;
                throw GraphicsError("Failed to create sprite");
            }
            return sprite;
        } catch (std::bad_alloc&) {
            throw GraphicsError("Out of memory");
        }
    }
    
    // Safe drawing operations
    void drawSafely(int16_t x, int16_t y, 
                    LGFX_Sprite* sprite) {
        if (!sprite) {
            throw GraphicsError("Null sprite");
        }
        
        // Validate coordinates
        if (x < 0 || y < 0 ||
            x + sprite->width() > lcd.width() ||
            y + sprite->height() > lcd.height()) {
            throw GraphicsError("Invalid coordinates");
        }
        
        // Perform drawing
        sprite->pushSprite(x, y);
    }
};
```

These examples demonstrate key concepts in LovyanGFX with practical implementations and detailed explanations. Understanding these concepts will help you create efficient and reliable graphics applications. 