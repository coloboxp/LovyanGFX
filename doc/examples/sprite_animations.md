# LovyanGFX Sprite Animation Examples

This guide provides practical examples of sprite animations based on the LovyanGFX example code.

## Basic Animation Concepts

Sprites in LovyanGFX can be used to create smooth animations by:
1. Creating a sprite buffer in memory
2. Drawing to the sprite using hardware-accelerated functions
3. Manipulating the sprite (rotation, scaling, etc.) using fixed-point math
4. Displaying the sprite using DMA transfers when available
5. Repeating the process while managing memory and performance

## Memory Management Considerations

Before implementing animations, consider these memory aspects:
```cpp
// Calculate sprite memory requirements
// Color depth affects memory usage:
// - 16-bit color (RGB565): 2 bytes per pixel
// - 24-bit color (RGB888): 3 bytes per pixel
// - 32-bit color (ARGB8888): 4 bytes per pixel
size_t calculateSpriteMemory(int width, int height, int colorDepth) {
    // Add 32-byte alignment padding for DMA
    size_t bytesPerPixel = (colorDepth + 7) >> 3;  // Convert bits to bytes
    size_t baseSize = width * height * bytesPerPixel;
    return (baseSize + 31) & ~31;  // Round up to 32-byte boundary
}
```

## Rotating Dial Example

Based on the RotateDial example, this demonstrates a smooth rotating animation:

```cpp
// Sprite instances for double-buffering technique
LGFX_Sprite dialSprite(&display);    // Main dial sprite (source buffer)
LGFX_Sprite tempSprite(&display);    // Temporary sprite for rotation (work buffer)

void setup() {
    display.init();
    display.setRotation(0);
    display.fillScreen(TFT_BLACK);
    
    // Create sprites with RGB565 color depth (16-bit)
    // This provides good balance between color quality and memory usage
    dialSprite.setColorDepth(16);    // 2 bytes per pixel
    dialSprite.createSprite(120, 120);  // 28.8KB memory usage
    tempSprite.setColorDepth(16);
    tempSprite.createSprite(120, 120);  // Another 28.8KB
    
    // Draw the initial dial design
    drawDial();
}

void drawDial() {
    // Clear sprite with black background
    dialSprite.fillSprite(TFT_BLACK);
    
    // Draw concentric circles for dial background
    // Outer circle: Dark blue background
    dialSprite.fillCircle(60, 60, 58, TFT_NAVY);
    // Inner circle: Lighter blue for contrast
    dialSprite.fillCircle(60, 60, 52, TFT_BLUE);
    
    // Draw tick marks at 30-degree intervals
    // Complete circle = 360 degrees
    // 12 tick marks = 30 degrees each
    for (int i = 0; i < 360; i += 30) {
        // Convert degrees to radians for trigonometric functions
        // DEG_TO_RAD = π/180 ≈ 0.0174532925
        float rad = i * DEG_TO_RAD;
        
        // Calculate tick mark endpoints using trigonometry
        // sin(angle) gives X offset, cos(angle) gives Y offset
        // Center point (60,60) + offset gives final coordinates
        int x1 = 60 + sin(rad) * 40;  // Inner point, 40 pixels from center
        int y1 = 60 - cos(rad) * 40;  // Subtract for Y because screen coordinates increase downward
        int x2 = 60 + sin(rad) * 50;  // Outer point, 50 pixels from center
        int y2 = 60 - cos(rad) * 50;
        
        // Draw the tick mark line
        dialSprite.drawLine(x1, y1, x2, y2, TFT_WHITE);
    }
    
    // Draw pointer triangle
    // Points: Left base (57,20), Right base (63,20), Tip at center (60,60)
    dialSprite.fillTriangle(57, 20, 63, 20, 60, 60, TFT_RED);
}

void loop() {
    // Track rotation angle in degrees
    // Static ensures value persists between calls
    static float angle = 0;
    
    // Clear temporary sprite to prevent artifacts
    tempSprite.fillSprite(TFT_BLACK);
    
    // Rotate dial sprite into temporary sprite
    // Parameters:
    // - Destination sprite (&tempSprite)
    // - Rotation center X (60 = sprite width/2)
    // - Rotation center Y (60 = sprite height/2)
    // - Rotation angle in degrees
    // - X and Y scaling factors (1.0 = no scaling)
    dialSprite.pushRotateZoom(&tempSprite, 60, 60, angle, 1.0, 1.0);
    
    // Center the sprite on display
    // Calculate position to center:
    // (display dimension - sprite dimension) / 2
    tempSprite.pushSprite(
        (display.width() - tempSprite.width()) / 2,
        (display.height() - tempSprite.height()) / 2
    );
    
    // Update rotation angle
    angle += 1.0;  // Increment by 1 degree per frame
    // Keep angle between 0-359 degrees
    if (angle >= 360.0) angle -= 360.0;
}
```

## Moving Icons Example

This example demonstrates sprite movement, rotation, and transparency:

```cpp
// Structure to track sprite properties
struct Icon {
    float x, y;        // Position with sub-pixel precision
    float dx, dy;      // Velocity components (pixels per frame)
    float angle;       // Rotation angle in degrees
    LGFX_Sprite* spr;  // Pointer to sprite object
};

// Vector to store multiple icons
std::vector<Icon> icons;

// Temporary sprite for rotation operations
// Using a single rotation buffer saves memory
LGFX_Sprite tempSprite(&display);

void setup() {
    display.init();
    display.fillScreen(TFT_BLACK);
    
    // Create temporary sprite for rotation
    // 32x32 pixels, 16-bit color = 2KB memory
    tempSprite.setColorDepth(16);
    tempSprite.createSprite(32, 32);
    
    // Create and initialize multiple icons
    // Each icon requires: 32x32 pixels * 2 bytes = 2KB
    for (int i = 0; i < 10; i++) {
        Icon icon;
        
        // Allocate and configure sprite
        icon.spr = new LGFX_Sprite(&display);
        icon.spr->setColorDepth(16);  // RGB565 color mode
        icon.spr->createSprite(32, 32);
        
        // Draw icon design
        // Using smooth circle for anti-aliased edges
        icon.spr->fillSmoothCircle(
            16, 16,     // Center coordinates
            15,         // Radius
            TFT_BLUE    // Fill color
        );
        icon.spr->drawSmoothCircle(
            16, 16,     // Center coordinates
            15,         // Radius
            TFT_WHITE   // Outline color
        );
        
        // Initialize physics properties
        // Random starting position within display bounds
        icon.x = random(display.width());   // X position
        icon.y = random(display.height());  // Y position
        
        // Random velocity between -1.0 and +1.0 pixels per frame
        // random(20) gives 0-19, divide by 10.0 gives 0.0-1.9
        // Subtract 1.0 gives range of -1.0 to +0.9
        icon.dx = random(20) / 10.0 - 1.0;  // X velocity
        icon.dy = random(20) / 10.0 - 1.0;  // Y velocity
        
        // Random initial rotation angle (0-359 degrees)
        icon.angle = random(360);
        
        icons.push_back(icon);
    }
}

void loop() {
    // Start a single transaction for all drawing
    // This reduces SPI overhead
    display.startWrite();
    
    // Clear screen with black background
    display.fillScreen(TFT_BLACK);
    
    // Update and draw each icon
    for (auto& icon : icons) {
        // Update position using velocity
        // Float positions allow for smooth sub-pixel movement
        icon.x += icon.dx;
        icon.y += icon.dy;
        
        // Increment rotation angle
        // 2.0 degrees per frame = full rotation in 180 frames
        icon.angle += 2.0;
        
        // Boundary collision detection
        // Check against display edges minus sprite size
        if (icon.x < 0 || icon.x > display.width() - 32) {
            // Reverse X velocity on collision
            icon.dx = -icon.dx;
        }
        if (icon.y < 0 || icon.y > display.height() - 32) {
            // Reverse Y velocity on collision
            icon.dy = -icon.dy;
        }
        
        // Clear temporary sprite to prevent artifacts
        tempSprite.fillSprite(TFT_BLACK);
        
        // Rotate icon into temporary sprite
        // Parameters:
        // - Destination sprite
        // - Rotation center X (16 = half width)
        // - Rotation center Y (16 = half height)
        // - Rotation angle
        // - No scaling (1.0)
        icon.spr->pushRotateZoom(
            &tempSprite,
            16, 16,           // Center point
            icon.angle,       // Current angle
            1.0, 1.0          // No scaling
        );
        
        // Display rotated sprite with transparency
        // Black pixels (0x0000) are treated as transparent
        tempSprite.pushSprite(
            icon.x,           // X position
            icon.y,           // Y position
            TFT_BLACK         // Transparent color
        );
    }
    
    // End the drawing transaction
    display.endWrite();
}

// Cleanup function to prevent memory leaks
void cleanup() {
    for (auto& icon : icons) {
        // Delete sprite object
        if (icon.spr) {
            icon.spr->deleteSprite();
            delete icon.spr;
        }
    }
    icons.clear();
}
```

## Transition Effects

This example demonstrates smooth screen transitions using sprite manipulation and alpha blending:

```cpp
// Sprite buffers for source and destination screens
// Each buffer requires: width * height * 2 bytes (RGB565)
LGFX_Sprite srcSprite(&display);    // Source screen buffer
LGFX_Sprite dstSprite(&display);    // Destination screen buffer

// Slide transition effect
// Parameters:
// - direction: 0 for left-to-right, 1 for right-to-left
void slideTransition(int direction = 0) {
    // Get display dimensions for calculations
    int width = display.width();
    int steps = 30;  // Number of animation steps (smoothness)
    
    // Perform transition animation
    for (int i = 0; i <= steps; i++) {
        // Start transaction for multiple sprite operations
        display.startWrite();
        
        // Calculate offset for current animation step
        // For left transition (direction = 0):
        //   - Offset goes from 0 to -width
        // For right transition (direction = 1):
        //   - Offset goes from 0 to +width
        int offset = (direction == 0) 
            ? -width * i / steps    // Left movement
            : width * i / steps;    // Right movement
            
        // Draw source sprite with calculated offset
        // As one sprite moves out, the other moves in
        srcSprite.pushSprite(offset, 0);
        
        // Draw destination sprite
        // Position depends on slide direction
        dstSprite.pushSprite(
            // For left slide: start at right edge
            // For right slide: start at left edge
            offset + (direction == 0 ? width : -width), 
            0  // Y position remains constant
        );
        
        // End transaction and add delay for smooth animation
        display.endWrite();
        delay(10);  // 10ms delay = ~100 FPS maximum
    }
}

// Fade transition effect using alpha blending
void fadeTransition() {
    // 32 steps for smooth fade (5-bit alpha channel)
    const int steps = 32;  // 0-31 alpha values
    
    // Perform fade animation
    for (int alpha = 0; alpha < steps; alpha++) {
        // Start transaction for multiple sprite operations
        display.startWrite();
        
        // Draw source sprite (fully opaque)
        srcSprite.pushSprite(0, 0);
        
        // Draw destination sprite with increasing alpha
        // Alpha value is shifted left by 3 bits (0-255 range)
        // This converts 5-bit alpha (0-31) to 8-bit (0-255)
        dstSprite.pushSprite(
            0, 0,           // X, Y coordinates
            alpha << 3      // Alpha value (0-255)
        );
        
        // End transaction and add delay for smooth animation
        display.endWrite();
        delay(10);  // 10ms delay = ~100 FPS maximum
    }
}

// Example usage of transitions
void performTransitions() {
    // Prepare source sprite
    srcSprite.createSprite(display.width(), display.height());
    srcSprite.setColorDepth(16);  // RGB565 color mode
    
    // Prepare destination sprite
    dstSprite.createSprite(display.width(), display.height());
    dstSprite.setColorDepth(16);  // RGB565 color mode
    
    // Draw content in source sprite
    srcSprite.fillScreen(TFT_BLACK);
    srcSprite.drawRect(50, 50, 100, 100, TFT_RED);
    
    // Draw content in destination sprite
    dstSprite.fillScreen(TFT_BLACK);
    dstSprite.fillCircle(100, 100, 50, TFT_BLUE);
    
    // Perform slide transition (left to right)
    slideTransition(0);
    delay(1000);  // Pause to show result
    
    // Perform slide transition (right to left)
    slideTransition(1);
    delay(1000);  // Pause to show result
    
    // Perform fade transition
    fadeTransition();
    
    // Clean up sprites to free memory
    srcSprite.deleteSprite();
    dstSprite.deleteSprite();
}
```

## Clock Animation

This example demonstrates an analog clock implementation with smooth hand movement:

```cpp
// Sprite buffers for clock components
// Clock face remains static, hands are rotated
LGFX_Sprite clockSprite(&display);  // Main clock face buffer
LGFX_Sprite handSprite(&display);   // Buffer for rotating hands

void setup() {
    display.init();
    
    // Create clock face sprite
    // 240x240 pixels, 16-bit color = 115.2KB
    clockSprite.setColorDepth(16);  // RGB565 color mode
    clockSprite.createSprite(240, 240);
    
    // Create hand sprite for rotation
    // 8x120 pixels, 16-bit color = 1.92KB
    // Narrow width reduces memory usage and rotation artifacts
    handSprite.setColorDepth(16);
    handSprite.createSprite(8, 120);
    
    // Draw static clock face
    drawClockFace();
}

void drawClockFace() {
    // Clear sprite with black background
    clockSprite.fillSprite(TFT_BLACK);
    
    // Draw clock bezel
    // Outer circle: Dark blue for depth effect
    clockSprite.fillCircle(120, 120, 118, TFT_NAVY);
    // Inner circle: Lighter blue for face
    clockSprite.fillCircle(120, 120, 115, TFT_BLUE);
    
    // Draw hour markers
    // 12 markers at 30-degree intervals
    for (int i = 0; i < 12; i++) {
        // Convert hour position to angle in radians
        // Each hour = 30 degrees (360/12)
        float angle = i * 30 * DEG_TO_RAD;
        
        // Calculate marker endpoints using trigonometry
        // Inner point: 90 pixels from center
        // Outer point: 110 pixels from center
        int x1 = 120 + sin(angle) * 90;   // Inner X
        int y1 = 120 - cos(angle) * 90;   // Inner Y
        int x2 = 120 + sin(angle) * 110;  // Outer X
        int y2 = 120 - cos(angle) * 110;  // Outer Y
        
        // Draw thick white line for hour marker
        // Width: 5 pixels for visibility
        clockSprite.drawWideLine(
            x1, y1,        // Start point
            x2, y2,        // End point
            5,             // Line width
            TFT_WHITE      // Color
        );
    }
}

// Draw a clock hand with specified angle and length
// Parameters:
// - angle: Rotation angle in degrees
// - length: Hand length in pixels
// - color: Hand color (usually white for hour/minute, red for seconds)
void drawHand(float angle, int length, uint32_t color) {
    // Clear hand sprite to prevent artifacts
    handSprite.fillSprite(TFT_BLACK);
    
    // Draw vertical line for hand
    // Center horizontally (x = 4 in 8-pixel width)
    // Length determines hand size
    handSprite.drawWideLine(
        4, 0,           // Start at top center
        4, length,      // End at specified length
        3,              // Hand width of 3 pixels
        color           // Hand color
    );
    
    // Rotate hand sprite around clock center
    // Uses hardware-accelerated rotation when available
    handSprite.pushRotateZoom(
        120, 120,       // Rotation center (clock center)
        angle,          // Rotation angle
        1.0, 1.0,       // No scaling
        TFT_BLACK       // Transparent color
    );
}

void loop() {
    // Track time between updates
    static uint32_t lastTime = 0;
    uint32_t currentTime = millis();
    
    // Update once per second
    if (currentTime - lastTime >= 1000) {
        lastTime = currentTime;
        
        // Get current time components
        time_t t = time(nullptr);
        struct tm* tm = localtime(&t);
        
        // Start drawing transaction
        display.startWrite();
        
        // Draw static clock face
        clockSprite.pushSprite(0, 0);
        
        // Calculate hand angles
        // Hour hand: 360° / 12 hours = 30° per hour
        //           + 30° * (minutes/60) for smooth movement
        float hoursAngle = (tm->tm_hour % 12 + tm->tm_min / 60.0) * 30 - 90;
        
        // Minute hand: 360° / 60 minutes = 6° per minute
        float minutesAngle = tm->tm_min * 6 - 90;
        
        // Second hand: 360° / 60 seconds = 6° per second
        float secondsAngle = tm->tm_sec * 6 - 90;
        
        // Draw hands from shortest to longest
        // This ensures proper overlay order
        drawHand(hoursAngle, 60, TFT_WHITE);    // Hour hand: shortest
        drawHand(minutesAngle, 80, TFT_WHITE);  // Minute hand: medium
        drawHand(secondsAngle, 100, TFT_RED);   // Second hand: longest
        
        // End drawing transaction
        display.endWrite();
    }
}

// Memory cleanup function
void cleanup() {
    // Free sprite memory
    clockSprite.deleteSprite();
    handSprite.deleteSprite();
}

## Performance Tips

1. Memory Management:
   ```cpp
   // Calculate required memory before creating sprites
   size_t requiredMemory = width * height * (bitsPerPixel / 8);
   if (heap_caps_get_free_size(MALLOC_CAP_DMA) > requiredMemory) {
       sprite.createSprite(width, height);
   }
   ```

2. Color Depth Selection:
   ```cpp
   // Use 16-bit color for better performance
   sprite.setColorDepth(16);  // RGB565: Good balance of quality/speed
   
   // Use 8-bit color for memory-constrained systems
   sprite.setColorDepth(8);   // 256 colors: 1/2 the memory of RGB565
   ```

3. Transaction Optimization:
   ```cpp
   // Batch multiple operations in one transaction
   display.startWrite();
   // Multiple sprite operations here...
   display.endWrite();
   ```

4. DMA Usage:
   ```cpp
   // Enable DMA for hardware-accelerated transfers
   auto cfg = display.config();
   cfg.dma_channel = 1;
   cfg.use_psram = true;  // Use PSRAM for large sprites
   ```

5. Sprite Reuse:
   ```cpp
   // Create sprites once and reuse
   sprite.createSprite(width, height);  // Create in setup
   
   void loop() {
       sprite.fillSprite(TFT_BLACK);    // Clear for reuse
       // Draw new content...
   }
   ```

These examples demonstrate various animation techniques using LovyanGFX sprites. They can be combined and modified to create more complex animations and effects. 