# Animation Techniques in LovyanGFX

This guide explains various animation techniques and best practices for creating smooth, efficient animations using LovyanGFX. Each technique is explained in detail with practical examples.

## Smooth Movement

Understanding fixed-point arithmetic for smooth motion. This technique prevents jerky movement by using sub-pixel precision:

```cpp
// Position variables store sub-pixel precision using fixed-point format
// The format is: whole_number * 256 + fraction
// For example:
// - 160.0 = 160 * 256 = 40960
// - 160.5 = (160 * 256) + 128 = 41088
int32_t current_x = 0;        // Starting at 0.0
int32_t current_y = 0;        // Starting at 0.0
int32_t target_x = 160 * 256; // Moving to x = 160.0
int32_t target_y = 120 * 256; // Moving to y = 120.0

// Velocity (speed and direction of movement)
// Positive values move right/down, negative values move left/up
int32_t velocity_x = 0;       // No horizontal movement initially
int32_t velocity_y = 0;       // No vertical movement initially

void updateMovement() {
    // Update position by adding velocity
    // Since both position and velocity use the same fixed-point format,
    // we can add them directly
    current_x += velocity_x;
    current_y += velocity_y;
    
    // Gradually change velocity to create smooth acceleration
    // If we're not at the target position, adjust velocity by 1 unit
    // This creates a smooth acceleration and deceleration effect
    velocity_x += (current_x < target_x) ? 1 : -1;
    velocity_y += (current_y < target_y) ? 1 : -1;
    
    // Convert fixed-point positions to screen coordinates
    // We divide by 256 (shift right by 8) to get the pixel position
    int16_t screen_x = current_x >> 8;
    int16_t screen_y = current_y >> 8;
    
    // Draw at the calculated position
    lcd.fillCircle(screen_x, screen_y, 5, TFT_WHITE);
}
```

## Sprite Animation

Creating smooth sprite animations with rotation and scaling. This system manages multiple animation frames and applies transformations:

```cpp
class AnimatedSprite {
private:
    // Array to hold animation frame sprites
    // Each frame is a separate sprite that can be displayed
    LGFX_Sprite* _frames[10];
    
    float _angle;              // Current rotation angle (in degrees)
    float _scale;              // Current scale factor (1.0 = normal size)
    uint8_t _current_frame;    // Index of current animation frame
    uint32_t _last_update;     // Time of last frame change (milliseconds)
    
    // Time between frame changes in milliseconds
    // 50ms = 20 frames per second
    const uint32_t FRAME_TIME = 50;

public:
    AnimatedSprite(LGFX* display) {
        // Initialize animation frames
        // Each frame is a separate sprite that can hold part of the animation
        for (int i = 0; i < 10; i++) {
            _frames[i] = new LGFX_Sprite(display);
            // Use 16-bit color (RGB565) for better quality
            _frames[i]->setColorDepth(16);
        }
        
        // Initialize animation state
        _angle = 0.0f;         // Start with no rotation
        _scale = 1.0f;         // Start at normal size
        _current_frame = 0;    // Start with first frame
        _last_update = 0;      // Initialize timer
    }
    
    void update() {
        // Get current time in milliseconds
        uint32_t now = millis();
        
        // Check if it's time to advance to next frame
        // This creates consistent animation speed regardless of
        // how fast the rest of the code runs
        if (now - _last_update >= FRAME_TIME) {
            // Move to next frame, wrap around to 0 if at end
            _current_frame = (_current_frame + 1) % 10;
            _last_update = now;  // Reset timer
        }
        
        // Get sprite for current frame
        LGFX_Sprite* current = _frames[_current_frame];
        
        // Calculate center of screen
        // We divide width/height by 2 using right shift
        // This is the point we'll rotate/scale around
        int16_t center_x = lcd.width() >> 1;
        int16_t center_y = lcd.height() >> 1;
        
        // Draw current frame with rotation and scaling
        // pushRotateZoom handles both rotation and scaling in one operation
        current->pushRotateZoom(
            &lcd,              // Target display
            center_x,          // X center of rotation
            center_y,          // Y center of rotation
            _angle,            // Current rotation angle
            _scale, _scale,    // X and Y scaling factors
            TFT_BLACK          // Color to treat as transparent
        );
        
        // Update animation parameters
        
        // Rotate 2 degrees each frame
        _angle += 2.0f;
        // Keep angle between 0 and 360 degrees
        if (_angle >= 360.0f) _angle -= 360.0f;
        
        // Create pulsing scale effect using sine wave
        // sin() returns values between -1 and 1
        // We adjust this to scale between 0.5 and 2.0
        // millis()/1000.0f creates a smooth 1-second cycle
        _scale = 1.0f + 0.5f * sin(millis() / 1000.0f);
    }
};
```

## Transition Effects

Implementing smooth screen transitions. These effects make screen changes look more polished:

```cpp
class ScreenTransition {
private:
    LGFX_Sprite _source;      // Current screen content
    LGFX_Sprite _destination; // New screen content
    
    // Number of steps in transition animation
    // More steps = smoother animation but slower transition
    const uint8_t STEPS = 30;

public:
    // Different types of transitions available
    enum class Type {
        FADE,        // Fade from one screen to another
        SLIDE_LEFT,  // Slide new screen in from right
        SLIDE_RIGHT, // Slide new screen in from left
        ZOOM        // Zoom in/out between screens
    };
    
    // Slide transition effect
    void slideTransition(Type direction) {
        // Calculate how far screen needs to move
        // For slide left: negative width (move left)
        // For slide right: positive width (move right)
        int16_t distance = (direction == Type::SLIDE_LEFT) ?
                          -lcd.width() : lcd.width();
        
        // Perform transition animation
        for (uint8_t step = 0; step <= STEPS; step++) {
            // Calculate current position in the transition
            // step/STEPS gives percentage completion (0.0 to 1.0)
            // multiply by distance to get current offset
            int16_t offset = distance * step / STEPS;
            
            // Start drawing transaction for efficiency
            lcd.startWrite();
            
            // Draw both old and new screens with offset
            // This creates sliding effect
            _source.pushSprite(offset, 0);
            _destination.pushSprite(
                // For slide left: new screen starts at right edge
                // For slide right: new screen starts at left edge
                offset + (direction == Type::SLIDE_LEFT ?
                         lcd.width() : -lcd.width()),
                0
            );
            
            // End transaction and add small delay
            // 10ms delay gives ~100fps animation
            lcd.endWrite();
            delay(10);
        }
    }
    
    // Fade transition effect
    void fadeTransition() {
        // Alpha ranges from 0 (transparent) to 31 (opaque)
        // We use 5-bit alpha because that's what hardware supports
        for (uint8_t alpha = 0; alpha < 32; alpha++) {
            lcd.startWrite();
            
            // Draw old screen (fully opaque)
            _source.pushSprite(0, 0);
            
            // Draw new screen with increasing opacity
            // Convert 5-bit alpha (0-31) to 8-bit (0-255)
            // by multiplying by 8 (left shift by 3)
            _destination.pushSprite(0, 0, alpha << 3);
            
            lcd.endWrite();
            delay(10);  // 10ms delay for smooth animation
        }
    }
    
    // Zoom transition effect
    void zoomTransition(bool zoomIn) {
        for (uint8_t step = 0; step <= STEPS; step++) {
            lcd.startWrite();
            
            // Calculate current scale factor
            float scale;
            if (zoomIn) {
                // Start small (0.1) and grow to full size (1.0)
                scale = 0.1f + (0.9f * step / STEPS);
            } else {
                // Start full size (1.0) and shrink to small (0.1)
                scale = 1.0f - (0.9f * step / STEPS);
            }
            
            // Draw scaled content
            if (zoomIn) {
                // For zoom in: draw old screen, then scaled new screen
                _source.pushSprite(0, 0);
                _destination.pushRotateZoom(
                    &lcd,
                    lcd.width() >> 1,    // Center X, >> 1 is divide by 2
                    lcd.height() >> 1,   // Center Y, >> 1 is divide by 2
                    0,                   // No rotation
                    scale, scale         // Current scale
                );
            } else {
                // For zoom out: draw new screen, then scaled old screen
                _destination.pushSprite(0, 0);
                _source.pushRotateZoom(
                    &lcd,
                    lcd.width() >> 1,    // Center X, >> 1 is divide by 2
                    lcd.height() >> 1,   // Center Y, >> 1 is divide by 2
                    0,                   // No rotation
                    scale, scale         // Current scale
                );
            }
            
            lcd.endWrite();
            delay(10);
        }
    }
};
```

## Particle Systems

Creating dynamic particle effects. This system simulates many small objects moving independently:

```cpp
class Particle {
public:
    float x, y;           // Current position
    float vx, vy;         // Velocity (speed and direction)
    float life;           // Remaining lifetime (1.0 to 0.0)
    uint32_t color;       // RGBA color (including alpha)
    uint8_t size;         // Particle size in pixels
    
    void update() {
        // Update position based on velocity
        // This creates the basic movement
        x += vx;  // Move horizontally
        y += vy;  // Move vertically
        
        // Apply gravity effect
        // Increase downward velocity by 0.1 each frame
        // This creates a falling effect
        vy += 0.1f;
        
        // Gradually reduce lifetime
        // 0.02 reduction means particle lasts ~50 frames
        life -= 0.02f;
        
        // Update particle transparency based on remaining life
        // As life approaches 0, particle becomes more transparent
        // color format: 0xAARRGGBB (AA = alpha channel)
        uint8_t alpha = life * 255;  // Convert life to alpha (0-255)
        // Clear old alpha bits and set new alpha value
        color = (color & 0x00FFFFFF) | (alpha << 24); // 0x00FFFFFF is 0x000000FF with 24 bits of 0
    }
    
    // Check if particle should be removed
    bool isDead() {
        return life <= 0.0f;  // Remove when life reaches 0
    }
};

class ParticleSystem {
private:
    // Dynamic array to store active particles
    std::vector<Particle> _particles;
    // Buffer for drawing particles
    LGFX_Sprite _buffer;
    
public:
    // Create new particles at specified position
    void emit(float x, float y, uint32_t color, uint8_t count) {
        for (uint8_t i = 0; i < count; i++) {
            Particle p;
            // Set initial position
            p.x = x;
            p.y = y;
            
            // Calculate random velocity in circular pattern
            // This creates a explosion-like effect
            float angle = random(360) * DEG_TO_RAD;  // Random direction
            float speed = random(20, 50) / 10.0f;    // Random speed
            // Convert angle and speed to X/Y velocity
            p.vx = cos(angle) * speed;  // Horizontal velocity
            p.vy = sin(angle) * speed;  // Vertical velocity
            
            p.life = 1.0f;              // Start with full life
            p.color = color;            // Use specified color
            p.size = random(2, 5);      // Random size
            
            // Add to particle list
            _particles.push_back(p);
        }
    }
    
    void update() {
        // Clear drawing buffer
        // This removes previous frame's particles
        _buffer.fillScreen(TFT_BLACK);
        
        // Update and draw all particles
        for (auto it = _particles.begin();
             it != _particles.end();) {
            // Update particle physics and properties
            it->update();
            
            if (it->isDead()) {
                // Remove dead particles
                it = _particles.erase(it);
            } else {
                // Draw live particles
                _buffer.fillCircle(
                    it->x, it->y,     // Position
                    it->size,         // Size
                    it->color         // Color with alpha
                );
                ++it;  // Move to next particle
            }
        }
        
        // Display updated particle buffer
        _buffer.pushSprite(0, 0);
    }
};
```

## Performance Tips

1. Use Fixed-Point Math for Smooth Animation
   ```cpp
   // AVOID: Floating point calculations are slow
   float x = 160.5f;
   x += 0.1f;
   
   // BETTER: Use fixed-point math
   // Numbers are multiplied by 256 to store fractions
   int32_t x = 160 * 256 + 128;  // 160.5 in fixed-point
   x += 26;                       // 0.1 in fixed-point (0.1 * 256)
   ```

2. Optimize Sprite Memory Usage
   ```cpp
   // Create a reusable sprite buffer
   // This prevents memory fragmentation
   static LGFX_Sprite temp_buffer(&lcd);
   
   // Only clear the parts that changed
   // This is faster than clearing the whole sprite
   sprite.fillRect(
       dirty_x, dirty_y,    // Region that changed
       dirty_w, dirty_h,    // Size of changed region
       TFT_BLACK           // Background color
   );
   
   // Use appropriate color depth for content
   // 8-bit = 256 colors, uses less memory
   // 16-bit = 65536 colors, better quality
   sprite.setColorDepth(8);  // Use 8-bit if possible
   ```

3. Batch Drawing Operations for Speed
   ```cpp
   // AVOID: Multiple separate drawing operations
   // Each operation requires a new SPI transaction
   background.pushSprite(0, 0);
   particles.pushSprite(0, 0);
   ui.pushSprite(0, 0);
   
   // BETTER: Batch operations in single transaction
   lcd.startWrite();
   {
       // All drawing happens in one transaction
       // This is much faster on SPI displays
       background.pushSprite(0, 0);
       particles.pushSprite(0, 0);
       ui.pushSprite(0, 0);
   }
   lcd.endWrite();
   ```

4. Smart Memory Management
   ```cpp
   // Allocate sprites once and reuse them
   // This prevents memory fragmentation
   sprite.createSprite(width, height);
   
   // Monitor available memory
   size_t free_memory = heap_caps_get_free_size(MALLOC_CAP_DMA);
   
   // Adapt quality based on available memory
   if (free_memory < threshold) {
       // Reduce effects or quality to prevent crashes
       // Options:
       // - Use lower color depth
       // - Reduce sprite sizes
       // - Disable complex effects
   }
   ```

These examples demonstrate how to create efficient animations while managing system resources effectively. The key is to understand the hardware limitations and optimize accordingly. 