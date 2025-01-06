# Game Development Examples

This guide demonstrates game development techniques using LovyanGFX through practical examples.

## Ball Maze Game

A physics-based maze game showcasing sprite manipulation, touch input, and game physics.

### Basic Structure

```cpp
#include <LovyanGFX.hpp>

static LGFX lcd;                    // Display instance
static LGFX_Sprite sp(&lcd);       // Game sprite
static std::int32_t px, py;        // Player position
static float ox, oy;               // Origin coordinates
static float cx, cy, cr;           // Ball position and radius
static float ax, ay;               // Acceleration
static float zoom;                 // View zoom level
static float angle, rad;           // Rotation angles
static float add_angle = 0;        // Additional angle
static float add_advance = 0;      // Movement advance
static std::uint32_t draw_count = 0;
static std::uint32_t skip_count = 0;

// Physics constants
static constexpr float deg_to_rad = 0.017453292519943295769236907684886;
static constexpr float gravity = -0.005;
static constexpr float zoom_speed = 0.02;
```

### Game Initialization

```cpp
static void game_init(void) {
    // Center display coordinates
    px = lcd.width() >> 1;
    py = lcd.height() >> 3;

    // Create maze layout
    create_maze();
    
    // Set initial positions
    ox = sp.getPivotX();
    oy = sp.getPivotY();
    cx = (sp.width() >> 1);
    cy = (sp.height() >> 1);
    cr = 0.2;
    zoom = zoom_min;
}

static void create_maze(void) {
    sp.clear(0);
    
    // Draw maze borders
    sp.setColor(1);
    sp.fillRect(0, 0, sp.width(), 16);
    sp.fillRect(0, sp.height() - 16, sp.width(), 16);
    sp.fillRect(0, 0, 16, sp.height()-16);
    sp.fillRect(sp.width()-16, 0, 16, sp.height()-16);
    
    // Generate maze pattern
    sp.setColor(3);
    for (int y = 1; y < sp.height(); y += 2) {
        for (int x = 1; x < sp.width(); x += 2) {
            sp.writePixel(x, y);
            int xx = x;
            int yy = y;
            do {
                xx = x;
                yy = y;
                switch (random(4)) {
                case 0: xx = x + 1; break;
                case 1: xx = x - 1; break;
                case 2: yy = y + 1; break;
                case 3: yy = y - 1; break;
                }
            } while (3 == sp.readPixelValue(xx, yy));
            sp.writePixel(xx, yy);
        }
    }
}
```

### Game Physics

```cpp
static bool game_main(void) {
    // Handle touch input
    std::int32_t tx, ty, tc;
    tc = lcd.getTouch(&tx, &ty);
    
    // Handle zoom control
    if (tc > 1) {
        zoom *= 1.0 - zoom_speed;
        if (zoom < zoom_min) zoom = zoom_min;
    } else {
        zoom *= 1.0 + zoom_speed;
        if (zoom > zoom_max) zoom = zoom_max;
        
        // Handle rotation
        add_angle -= add_angle / 10;
        if (tc) {
            add_angle += (tx > lcd.width()>>1) ? 0.6 : -0.6;
        }
    }

    // Update rotation
    angle += add_angle;
    add_angle = add_angle * 9 / 10;
    rad = angle * -deg_to_rad;

    // Apply gravity
    ax += sinf(rad) * gravity;
    ay -= cosf(rad) * gravity;
    ax = ax * 9.7 / 10;
    ay = ay * 9.7 / 10;

    // Handle collisions
    float addy = (ay < 0.0) ? -cr : cr;
    auto tmpy = roundf(cy + ay + addy);
    
    if (3 == sp.readPixelValue(roundf(cx), tmpy)) {
        cy = tmpy - addy + (ay < 0.0 ? 0.5 : -0.5);
        ay = -ay * 9.0 / 10;
    } else {
        cy += ay;
    }

    float addx = (ax < 0.0) ? -cr : cr;
    auto tmpx = roundf(cx + ax + addx);
    if (3 == sp.readPixelValue(tmpx, roundf(cy))) {
        cx = tmpx - addx + (ax < 0.0 ? 0.5 : -0.5);
        ax = -ax * 9.0 / 10;
    } else {
        cx += ax;
    }

    // Check game state
    std::uint32_t pv = sp.readPixelValue(roundf(cx), roundf(cy));
    if (0 == pv) {
        sp.drawPixel(roundf(cx), roundf(cy), 2);
    } else if (1 == pv) return true;

    // Handle display updates
    if (++skip_count == draw_cycle) {
        skip_count = 0;
        draw();
    }
    return false;
}
```

### Rendering

```cpp
static void draw(void) {
    draw_count += draw_cycle;
    
    // Update palette colors for animation
    std::uint_fast8_t blink = 127 + abs(((int)(draw_count<<1) & 255)-128);
    sp.setPaletteColor(1, 127, 127, blink);
    sp.setPaletteColor(2, 127, blink, 127);

    // Calculate sprite position
    float fx = (cx - ox);
    float fy = (cy - oy);
    float len = sqrtf(fx * fx + fy * fy) * zoom;
    float theta = atan2f(fx, fy) + rad;
    
    // Draw rotated and zoomed sprite
    sp.pushRotateZoom(
        px - sinf(theta) * len,
        py - cosf(theta) * len,
        angle,
        zoom,
        zoom
    );
}
```

## Game Development Patterns

### State Management

```cpp
enum GameState {
    MENU,
    PLAYING,
    PAUSED,
    GAME_OVER
};

class GameManager {
    GameState current_state;
    
public:
    void update() {
        switch (current_state) {
            case MENU:
                updateMenu();
                break;
            case PLAYING:
                updateGame();
                break;
            case PAUSED:
                updatePause();
                break;
            case GAME_OVER:
                updateGameOver();
                break;
        }
    }
    
    void draw() {
        switch (current_state) {
            case MENU:
                drawMenu();
                break;
            case PLAYING:
                drawGame();
                break;
            case PAUSED:
                drawPause();
                break;
            case GAME_OVER:
                drawGameOver();
                break;
        }
    }
};
```

### Input Handling

```cpp
class InputManager {
    uint16_t touch_x, touch_y;
    bool touch_active;
    
public:
    void update() {
        touch_active = lcd.getTouch(&touch_x, &touch_y);
    }
    
    bool isTouchPressed() {
        return touch_active;
    }
    
    bool isTouchInRect(int x, int y, int w, int h) {
        if (!touch_active) return false;
        return touch_x >= x && touch_x < x + w &&
               touch_y >= y && touch_y < y + h;
    }
};
```

### Animation System

```cpp
class Animation {
    float current_value;
    float target_value;
    float speed;
    
public:
    void update() {
        if (current_value != target_value) {
            float diff = target_value - current_value;
            current_value += diff * speed;
            
            if (abs(diff) < 0.01f) {
                current_value = target_value;
            }
        }
    }
    
    void setTarget(float target) {
        target_value = target;
    }
    
    float getValue() {
        return current_value;
    }
};
```

## Best Practices

1. **Game Loop**
   - Maintain consistent frame rate
   - Separate update and draw logic
   - Handle input appropriately
   - Implement proper state management

2. **Physics**
   - Use fixed timestep for physics
   - Implement proper collision detection
   - Handle edge cases
   - Consider performance impact

3. **Memory Management**
   - Reuse sprites when possible
   - Clear unused resources
   - Monitor memory usage
   - Implement efficient resource loading

4. **Performance**
   - Use sprite batching
   - Optimize collision checks
   - Implement view culling
   - Monitor frame rate

## Common Issues and Solutions

1. **Frame Rate Issues**
   ```cpp
   // Implement frame rate control
   const uint32_t frame_time = 1000 / 60;  // 60 FPS
   static uint32_t last_frame = 0;
   
   if (millis() - last_frame >= frame_time) {
       updateGame();
       drawGame();
       last_frame = millis();
   }
   ```

2. **Physics Glitches**
   ```cpp
   // Use fixed timestep for physics
   const float physics_step = 1.0f / 60.0f;
   float physics_accumulator = 0;
   
   void update(float dt) {
       physics_accumulator += dt;
       while (physics_accumulator >= physics_step) {
           updatePhysics(physics_step);
           physics_accumulator -= physics_step;
       }
   }
   ```

3. **Memory Management**
   ```cpp
   // Implement resource management
   class ResourceManager {
       std::vector<LGFX_Sprite*> sprites;
       
   public:
       void cleanup() {
           for (auto sprite : sprites) {
               sprite->deleteSprite();
               delete sprite;
           }
           sprites.clear();
       }
   };
   ``` 