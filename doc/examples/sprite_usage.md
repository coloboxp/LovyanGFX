# Sprite Usage Guide

This guide covers sprite operations and animation techniques in LovyanGFX, demonstrating how to create, manipulate, and render sprites efficiently.

## Sprite Basics

### Creating Sprites

```cpp
#include <LovyanGFX.hpp>
#include <LGFX_AUTODETECT.hpp>

static LGFX lcd;                 // Main display instance
static LGFX_Sprite sprite(&lcd); // Create sprite with parent display

void setup() {
    lcd.init();
    
    // Create sprite with specific dimensions and color depth
    sprite.setColorDepth(16);    // Set color depth (1, 8, 16, or 24)
    sprite.createSprite(width, height);  // Allocate sprite memory
}
```

### Basic Sprite Operations

```cpp
// Clear sprite
sprite.clear();                  // Clear with transparent/background color
sprite.clear(TFT_BLACK);         // Clear with specific color

// Drawing on sprite
sprite.fillScreen(TFT_BLACK);    // Fill entire sprite
sprite.drawPixel(x, y, color);   // Draw single pixel
sprite.drawLine(x0, y0, x1, y1, color);  // Draw line
sprite.drawRect(x, y, w, h, color);      // Draw rectangle
sprite.fillRect(x, y, w, h, color);      // Fill rectangle
sprite.drawCircle(x, y, r, color);       // Draw circle
sprite.fillCircle(x, y, r, color);       // Fill circle

// Text on sprite
sprite.setTextColor(TFT_WHITE);
sprite.setCursor(x, y);
sprite.print("Hello Sprite!");

// Display sprite on screen
sprite.pushSprite(&lcd, x, y);   // Copy to display at position
```

## Advanced Sprite Features

### Sprite with Transparency

```cpp
// Create sprite with transparency support
sprite.setColorDepth(16);        // 16-bit color with transparency
sprite.createSprite(width, height);

// Set transparent color
sprite.fillSprite(TFT_BLACK);    // Fill with background
sprite.setTransparentColor(TFT_BLACK);  // Make black transparent

// Draw with transparency
sprite.pushSprite(&lcd, x, y, TFT_BLACK);  // Black pixels are transparent
```

### Sprite Rotation and Scaling

```cpp
// Rotate sprite
sprite.pushRotateZoom(&lcd,      // Destination
    center_x,                    // Rotation center X
    center_y,                    // Rotation center Y
    angle,                       // Rotation angle (0-360)
    zoom_x,                      // X scale factor (1.0 = no zoom)
    zoom_y,                      // Y scale factor
    transparent_color);          // Transparent color

// Simple rotation (90-degree steps)
sprite.pushRotated(&lcd, angle, transparent_color);
```

### Double Buffering

```cpp
static LGFX_Sprite sprites[2];   // Two sprite buffers

void setup() {
    // Initialize both sprites
    for (int i = 0; i < 2; i++) {
        sprites[i].setColorDepth(16);
        sprites[i].createSprite(width, height);
    }
}

void loop() {
    static uint8_t flip = 0;     // Buffer index
    
    // Draw to back buffer
    sprites[flip].clear();
    // ... drawing operations ...
    
    // Display back buffer
    sprites[flip].pushSprite(&lcd, 0, 0);
    
    // Swap buffers
    flip = !flip;
}
```

## Animation Techniques

### Moving Objects

```cpp
struct Object {
    int32_t x, y;       // Position
    int32_t dx, dy;     // Velocity
    uint32_t color;     // Color
    
    void move(int32_t width, int32_t height) {
        // Update position
        x += dx;
        y += dy;
        
        // Bounce off edges
        if (x < 0 || x >= width) dx = -dx;
        if (y < 0 || y >= height) dy = -dy;
    }
};

void animateObjects() {
    static Object objects[100];
    
    sprite.clear();
    
    // Update and draw each object
    for (auto &obj : objects) {
        obj.move(lcd.width(), lcd.height());
        sprite.fillCircle(obj.x, obj.y, 5, obj.color);
    }
    
    sprite.pushSprite(&lcd, 0, 0);
}
```

### Sprite Animation Sequence

```cpp
class AnimatedSprite {
    LGFX_Sprite frames[10];  // Animation frames
    uint8_t current_frame;
    uint32_t last_update;
    uint32_t frame_delay;
    
public:
    void init(int width, int height) {
        for (auto &frame : frames) {
            frame.setColorDepth(16);
            frame.createSprite(width, height);
        }
    }
    
    void update() {
        if (millis() - last_update > frame_delay) {
            current_frame = (current_frame + 1) % 10;
            last_update = millis();
        }
    }
    
    void draw(int x, int y) {
        frames[current_frame].pushSprite(&lcd, x, y);
    }
};
```

## Performance Optimization

### Memory Management

```cpp
// Efficient sprite creation
void createOptimizedSprite() {
    // Try creating sprite with available memory
    for (uint8_t depth : {16, 8, 4, 1}) {
        sprite.setColorDepth(depth);
        if (sprite.createSprite(width, height)) {
            return;  // Success
        }
    }
    // Handle failure
}

// Release sprite memory
void cleanup() {
    sprite.deleteSprite();
}
```

### Drawing Optimization

```cpp
void optimizedDrawing() {
    // Start transaction for multiple operations
    lcd.startWrite();
    
    // Draw multiple sprites
    for (auto &sprite : sprites) {
        sprite.pushSprite(&lcd, sprite.x, sprite.y);
    }
    
    // End transaction
    lcd.endWrite();
}
```

### Partial Updates

```cpp
void updateRegion() {
    // Only update changed region
    sprite.pushSprite(&lcd,
        dest_x, dest_y,          // Destination position
        src_x, src_y,           // Source position in sprite
        width, height);         // Region size
}
```

## Best Practices

1. **Memory Management**
   - Release unused sprites
   - Use appropriate color depth
   - Consider available RAM
   - Implement sprite pooling for reuse

2. **Performance**
   - Use double buffering for smooth animation
   - Batch sprite operations
   - Minimize sprite size
   - Use partial updates when possible

3. **Animation**
   - Implement frame timing control
   - Use sprite sheets for multiple frames
   - Consider using sprite pooling
   - Implement efficient collision detection

4. **Error Handling**
   - Check sprite creation success
   - Handle memory allocation failures
   - Validate sprite operations
   - Implement cleanup on errors

## Common Issues and Solutions

1. **Memory Allocation Failures**
   ```cpp
   // Handle sprite creation failure
   if (!sprite.createSprite(width, height)) {
       // Try smaller size or lower color depth
       sprite.setColorDepth(8);
       if (!sprite.createSprite(width, height)) {
           // Handle failure
       }
   }
   ```

2. **Performance Issues**
   ```cpp
   // Optimize multiple sprite updates
   lcd.startWrite();
   for (auto &sprite : sprites) {
       // Skip invisible sprites
       if (!sprite.isVisible()) continue;
       sprite.pushSprite(&lcd, sprite.x, sprite.y);
   }
   lcd.endWrite();
   ```

3. **Animation Glitches**
   ```cpp
   // Implement proper frame timing
   static uint32_t last_frame = 0;
   const uint32_t frame_delay = 1000 / 60;  // 60 FPS
   
   if (millis() - last_frame >= frame_delay) {
       // Update animation
       last_frame = millis();
   }
   ``` 