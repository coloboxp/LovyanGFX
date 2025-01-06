# LovyanGFX Sprite Animation Examples

This guide provides practical examples of sprite animations based on the LovyanGFX example code.

## Basic Animation Concepts

Sprites in LovyanGFX can be used to create smooth animations by:
1. Creating a sprite buffer
2. Drawing to the sprite
3. Manipulating the sprite (rotation, scaling, etc.)
4. Displaying the sprite
5. Repeating the process

## Rotating Dial Example

Based on the RotateDial example, this demonstrates a smooth rotating animation:

```cpp
LGFX_Sprite dialSprite(&display);    // Main dial sprite
LGFX_Sprite tempSprite(&display);    // Temporary sprite for rotation

void setup() {
    display.init();
    display.setRotation(0);
    display.fillScreen(TFT_BLACK);
    
    // Create sprites with appropriate color depth
    dialSprite.setColorDepth(16);
    dialSprite.createSprite(120, 120);
    tempSprite.setColorDepth(16);
    tempSprite.createSprite(120, 120);
    
    // Draw the dial design
    drawDial();
}

void drawDial() {
    dialSprite.fillSprite(TFT_BLACK);
    dialSprite.fillCircle(60, 60, 58, TFT_NAVY);
    dialSprite.fillCircle(60, 60, 52, TFT_BLUE);
    
    // Draw tick marks
    for (int i = 0; i < 360; i += 30) {
        float rad = i * DEG_TO_RAD;
        int x1 = 60 + sin(rad) * 40;
        int y1 = 60 - cos(rad) * 40;
        int x2 = 60 + sin(rad) * 50;
        int y2 = 60 - cos(rad) * 50;
        dialSprite.drawLine(x1, y1, x2, y2, TFT_WHITE);
    }
    
    // Draw pointer
    dialSprite.fillTriangle(57, 20, 63, 20, 60, 60, TFT_RED);
}

void loop() {
    static float angle = 0;
    
    // Clear temporary sprite
    tempSprite.fillSprite(TFT_BLACK);
    
    // Rotate dial sprite into temporary sprite
    dialSprite.pushRotateZoom(&tempSprite, 60, 60, angle, 1.0, 1.0);
    
    // Display the rotated sprite
    tempSprite.pushSprite(
        (display.width() - tempSprite.width()) / 2,
        (display.height() - tempSprite.height()) / 2
    );
    
    angle += 1.0;  // Increment rotation angle
    if (angle >= 360.0) angle -= 360.0;
}
```

## Moving Icons Example

Based on the MovingIcons example, this shows how to create smooth moving sprites:

```cpp
struct Icon {
    float x, y;        // Position
    float dx, dy;      // Movement delta
    float angle;       // Rotation angle
    LGFX_Sprite* spr;  // Sprite pointer
};

std::vector<Icon> icons;
LGFX_Sprite tempSprite(&display);

void setup() {
    display.init();
    display.fillScreen(TFT_BLACK);
    
    // Create temporary sprite for rotation
    tempSprite.setColorDepth(16);
    tempSprite.createSprite(32, 32);
    
    // Create and initialize icons
    for (int i = 0; i < 10; i++) {
        Icon icon;
        icon.spr = new LGFX_Sprite(&display);
        icon.spr->setColorDepth(16);
        icon.spr->createSprite(32, 32);
        
        // Draw icon design
        icon.spr->fillSmoothCircle(16, 16, 15, TFT_BLUE);
        icon.spr->drawSmoothCircle(16, 16, 15, TFT_WHITE);
        
        // Initialize position and movement
        icon.x = random(display.width());
        icon.y = random(display.height());
        icon.dx = random(20) / 10.0 - 1.0;
        icon.dy = random(20) / 10.0 - 1.0;
        icon.angle = random(360);
        
        icons.push_back(icon);
    }
}

void loop() {
    display.startWrite();
    display.fillScreen(TFT_BLACK);
    
    // Update and draw each icon
    for (auto& icon : icons) {
        // Update position
        icon.x += icon.dx;
        icon.y += icon.dy;
        icon.angle += 2.0;
        
        // Bounce off edges
        if (icon.x < 0 || icon.x > display.width() - 32) icon.dx = -icon.dx;
        if (icon.y < 0 || icon.y > display.height() - 32) icon.dy = -icon.dy;
        
        // Rotate and display icon
        tempSprite.fillSprite(TFT_BLACK);
        icon.spr->pushRotateZoom(&tempSprite, 16, 16, icon.angle, 1.0, 1.0);
        tempSprite.pushSprite(icon.x, icon.y, TFT_BLACK);  // Use black as transparent color
    }
    
    display.endWrite();
}
```

## Transition Effects

Based on the TransitionFX example, this demonstrates screen transitions:

```cpp
LGFX_Sprite srcSprite(&display);
LGFX_Sprite dstSprite(&display);

// Slide transition effect
void slideTransition(int direction = 0) {  // 0: left, 1: right
    int width = display.width();
    int steps = 30;  // Number of animation steps
    
    for (int i = 0; i <= steps; i++) {
        display.startWrite();
        
        int offset = (direction == 0) 
            ? -width * i / steps 
            : width * i / steps;
            
        // Draw source sprite with offset
        srcSprite.pushSprite(offset, 0);
        
        // Draw destination sprite
        dstSprite.pushSprite(offset + (direction == 0 ? width : -width), 0);
        
        display.endWrite();
        delay(10);  // Control animation speed
    }
}

// Fade transition effect
void fadeTransition() {
    const int steps = 32;
    
    for (int alpha = 0; alpha < steps; alpha++) {
        display.startWrite();
        
        // Draw source sprite
        srcSprite.pushSprite(0, 0);
        
        // Draw destination sprite with alpha
        dstSprite.pushSprite(0, 0, alpha << 3);
        
        display.endWrite();
        delay(10);
    }
}
```

## Clock Animation

Based on the ClockSample example, this shows how to create an analog clock:

```cpp
LGFX_Sprite clockSprite(&display);
LGFX_Sprite handSprite(&display);

void setup() {
    display.init();
    
    // Create clock face sprite
    clockSprite.setColorDepth(16);
    clockSprite.createSprite(240, 240);
    
    // Create hand sprite for rotation
    handSprite.setColorDepth(16);
    handSprite.createSprite(8, 120);
    
    drawClockFace();
}

void drawClockFace() {
    clockSprite.fillSprite(TFT_BLACK);
    clockSprite.fillCircle(120, 120, 118, TFT_NAVY);
    clockSprite.fillCircle(120, 120, 115, TFT_BLUE);
    
    // Draw hour markers
    for (int i = 0; i < 12; i++) {
        float angle = i * 30 * DEG_TO_RAD;
        int x1 = 120 + sin(angle) * 90;
        int y1 = 120 - cos(angle) * 90;
        int x2 = 120 + sin(angle) * 110;
        int y2 = 120 - cos(angle) * 110;
        clockSprite.drawWideLine(x1, y1, x2, y2, 5, TFT_WHITE);
    }
}

void drawHand(float angle, int length, uint32_t color) {
    handSprite.fillSprite(TFT_BLACK);
    handSprite.drawWideLine(4, 0, 4, length, 3, color);
    handSprite.pushRotateZoom(120, 120, angle, 1.0, 1.0, TFT_BLACK);
}

void loop() {
    static uint32_t lastTime = 0;
    uint32_t currentTime = millis();
    
    if (currentTime - lastTime >= 1000) {
        lastTime = currentTime;
        
        // Get current time
        time_t t = time(nullptr);
        struct tm* tm = localtime(&t);
        
        display.startWrite();
        
        // Draw clock face
        clockSprite.pushSprite(0, 0);
        
        // Calculate hand angles
        float hoursAngle = (tm->tm_hour % 12 + tm->tm_min / 60.0) * 30 - 90;
        float minutesAngle = tm->tm_min * 6 - 90;
        float secondsAngle = tm->tm_sec * 6 - 90;
        
        // Draw hands
        drawHand(hoursAngle, 60, TFT_WHITE);    // Hour hand
        drawHand(minutesAngle, 80, TFT_WHITE);  // Minute hand
        drawHand(secondsAngle, 100, TFT_RED);   // Second hand
        
        display.endWrite();
    }
}
```

## Performance Tips

1. Use appropriate color depth:
   ```cpp
   sprite.setColorDepth(16);  // 16-bit color is faster than 24-bit
   ```

2. Minimize sprite size:
   ```cpp
   sprite.createSprite(minWidth, minHeight);  // Use smallest size needed
   ```

3. Use startWrite/endWrite for multiple operations:
   ```cpp
   display.startWrite();
   // Multiple sprite operations
   display.endWrite();
   ```

4. Use DMA when available:
   ```cpp
   auto cfg = display.config();
   cfg.dma_channel = 1;
   cfg.use_psram = true;  // Use PSRAM for large sprites
   ```

5. Reuse sprites instead of creating/deleting:
   ```cpp
   // Create once in setup
   sprite.createSprite(width, height);
   
   // Reuse in loop
   sprite.fillSprite(TFT_BLACK);  // Clear for reuse
   ```

These examples demonstrate various animation techniques using LovyanGFX sprites. They can be combined and modified to create more complex animations and effects. 