# Advanced Features Guide

This guide covers advanced features in LovyanGFX, including custom shaders, hardware acceleration, and advanced animation techniques.

## Custom Shaders

### Pixel Shader Implementation

```cpp
// Custom pixel shader function
void customShader(int32_t x, int32_t y, uint32_t &color) {
    // Gradient effect
    uint8_t r = (color >> 16) & 0xFF;
    uint8_t g = (color >> 8) & 0xFF;
    uint8_t b = color & 0xFF;
    
    float factor = (float)x / display.width();
    
    r = (uint8_t)(r * factor);
    g = (uint8_t)(g * factor);
    b = (uint8_t)(b * factor);
    
    color = (r << 16) | (g << 8) | b;
}

// Apply shader to display
void applyShader() {
    display.setShader(customShader);
    display.fillRect(0, 0, display.width(), display.height(), TFT_WHITE);
    display.removeShader();
}
```

### Advanced Shader Effects

```cpp
// Ripple effect shader
void rippleShader(int32_t x, int32_t y, uint32_t &color) {
    float dx = x - display.width() / 2;
    float dy = y - display.height() / 2;
    float distance = sqrt(dx * dx + dy * dy);
    float angle = atan2(dy, dx);
    
    float wave = sin(distance * 0.1 + millis() * 0.001);
    float intensity = (wave + 1.0f) * 0.5f;
    
    uint8_t r = ((color >> 16) & 0xFF) * intensity;
    uint8_t g = ((color >> 8) & 0xFF) * intensity;
    uint8_t b = (color & 0xFF) * intensity;
    
    color = (r << 16) | (g << 8) | b;
}

// Plasma effect shader
void plasmaShader(int32_t x, int32_t y, uint32_t &color) {
    float time = millis() * 0.001;
    float value = sin(x * 0.1 + time)
                + sin(y * 0.1 + time)
                + sin((x + y) * 0.1 + time)
                + sin(sqrt(x * x + y * y) * 0.1);
    
    value = (value + 4) * 32;
    
    uint8_t r = value * ((color >> 16) & 0xFF) / 255;
    uint8_t g = value * ((color >> 8) & 0xFF) / 255;
    uint8_t b = value * (color & 0xFF) / 255;
    
    color = (r << 16) | (g << 8) | b;
}
```

## Hardware Acceleration

### DMA Operations

```cpp
class DMASpriteManager {
    LGFX_Sprite* sprites[MAX_SPRITES];
    bool dma_busy;
    
public:
    void updateSprite(uint32_t index, uint16_t x, uint16_t y) {
        if (dma_busy || index >= MAX_SPRITES) return;
        
        dma_busy = true;
        sprites[index]->pushSpriteDMA(x, y, [this]() {
            dma_busy = false;
        });
    }
    
    void waitDMA() {
        while (dma_busy) {
            delay(1);
        }
    }
};
```

### Parallel Bus Operations

```cpp
// Configure parallel bus for maximum performance
void setupParallelBus() {
    auto bus = new lgfx::Bus_Parallel8();
    auto cfg = bus->config();
    
    cfg.pin_wr = 4;
    cfg.pin_rd = 5;
    cfg.pin_rs = 6;
    cfg.pin_d0 = 16;
    cfg.pin_d1 = 17;
    cfg.pin_d2 = 18;
    cfg.pin_d3 = 19;
    cfg.pin_d4 = 20;
    cfg.pin_d5 = 21;
    cfg.pin_d6 = 22;
    cfg.pin_d7 = 23;
    
    cfg.freq_write = 40000000;  // 40MHz
    cfg.freq_read = 20000000;   // 20MHz
    
    bus->config(cfg);
    display.setBus(bus);
}
```

## Advanced Animation Techniques

### Particle System

```cpp
struct Particle {
    float x, y;
    float vx, vy;
    float life;
    uint32_t color;
};

class ParticleSystem {
    std::vector<Particle> particles;
    LGFX_Sprite* sprite;
    
public:
    ParticleSystem(LGFX* display) {
        sprite = new LGFX_Sprite(display);
        sprite->createSprite(display->width(), display->height());
    }
    
    void emit(float x, float y, uint32_t color, int count) {
        for (int i = 0; i < count; i++) {
            float angle = random(360) * DEG_TO_RAD;
            float speed = random(20, 50) * 0.1f;
            
            Particle p;
            p.x = x;
            p.y = y;
            p.vx = cos(angle) * speed;
            p.vy = sin(angle) * speed;
            p.life = 1.0f;
            p.color = color;
            
            particles.push_back(p);
        }
    }
    
    void update() {
        sprite->clear();
        
        for (auto it = particles.begin(); it != particles.end();) {
            it->x += it->vx;
            it->y += it->vy;
            it->vy += 0.1f;  // Gravity
            it->life -= 0.01f;
            
            if (it->life <= 0) {
                it = particles.erase(it);
            } else {
                uint8_t alpha = it->life * 255;
                sprite->drawPixel(it->x, it->y, 
                    sprite->color888(
                        ((it->color >> 16) & 0xFF) * it->life,
                        ((it->color >> 8) & 0xFF) * it->life,
                        (it->color & 0xFF) * it->life
                    )
                );
                ++it;
            }
        }
        
        sprite->pushSprite(0, 0);
    }
};
```

### Path Animation

```cpp
class PathAnimator {
    struct Point {
        float x, y;
        float angle;
    };
    
    std::vector<Point> points;
    float progress;
    float speed;
    
public:
    void addPoint(float x, float y) {
        Point p = {x, y, 0};
        if (!points.empty()) {
            Point& prev = points.back();
            p.angle = atan2(y - prev.y, x - prev.x);
        }
        points.push_back(p);
    }
    
    Point getPosition(float t) {
        if (points.size() < 2) return {0, 0, 0};
        
        float totalLength = 0;
        std::vector<float> lengths;
        
        for (size_t i = 1; i < points.size(); i++) {
            float dx = points[i].x - points[i-1].x;
            float dy = points[i].y - points[i-1].y;
            float len = sqrt(dx * dx + dy * dy);
            totalLength += len;
            lengths.push_back(len);
        }
        
        float targetDist = t * totalLength;
        float currentDist = 0;
        
        for (size_t i = 0; i < lengths.size(); i++) {
            if (currentDist + lengths[i] >= targetDist) {
                float segmentT = (targetDist - currentDist) / lengths[i];
                Point& p1 = points[i];
                Point& p2 = points[i+1];
                
                return {
                    p1.x + (p2.x - p1.x) * segmentT,
                    p1.y + (p2.y - p1.y) * segmentT,
                    p1.angle + (p2.angle - p1.angle) * segmentT
                };
            }
            currentDist += lengths[i];
        }
        
        return points.back();
    }
};
```

## Advanced Text Rendering

### Custom Font Renderer

```cpp
class FontRenderer {
    LGFX_Sprite* fontSprite;
    
public:
    void renderText(const char* text, int x, int y, float scale = 1.0f) {
        fontSprite->setTextSize(scale);
        fontSprite->setTextDatum(middle_center);
        
        int width = fontSprite->textWidth(text);
        int height = fontSprite->fontHeight();
        
        fontSprite->createSprite(width, height);
        fontSprite->clear(TFT_BLACK);
        fontSprite->setTextColor(TFT_WHITE);
        fontSprite->drawString(text, width/2, height/2);
        
        // Apply effects
        for (int py = 0; py < height; py++) {
            for (int px = 0; px < width; px++) {
                uint16_t color = fontSprite->readPixel(px, py);
                if (color != TFT_BLACK) {
                    // Apply glow effect
                    display.drawPixel(
                        x + px,
                        y + py,
                        display.color888(
                            255,
                            200 + random(55),
                            100 + random(155)
                        )
                    );
                }
            }
        }
        
        fontSprite->deleteSprite();
    }
};
```

## Best Practices

1. **Shader Performance**
   - Keep shader calculations simple
   - Use lookup tables for complex functions
   - Consider using fixed-point math
   - Profile shader performance

2. **Hardware Acceleration**
   - Use DMA when available
   - Batch similar operations
   - Minimize bus transactions
   - Monitor memory bandwidth

3. **Animation Performance**
   - Use sprite batching
   - Implement object pooling
   - Optimize update logic
   - Use appropriate data structures

4. **Memory Management**
   - Reuse sprite buffers
   - Implement resource pooling
   - Monitor memory fragmentation
   - Clean up unused resources

## Common Issues and Solutions

1. **Shader Performance**
   ```cpp
   // Use lookup tables for expensive calculations
   class FastTrigonometry {
       static constexpr int TABLE_SIZE = 360;
       float sinTable[TABLE_SIZE];
       
   public:
       FastTrigonometry() {
           for (int i = 0; i < TABLE_SIZE; i++) {
               sinTable[i] = sin(i * DEG_TO_RAD);
           }
       }
       
       float fastSin(float angle) {
           int index = ((int)(angle * 180/PI)) % TABLE_SIZE;
           if (index < 0) index += TABLE_SIZE;
           return sinTable[index];
       }
   };
   ```

2. **Memory Management**
   ```cpp
   // Implement sprite pooling
   class SpritePool {
       std::vector<LGFX_Sprite*> available;
       std::vector<LGFX_Sprite*> inUse;
       
   public:
       LGFX_Sprite* acquire() {
           if (available.empty()) {
               auto sprite = new LGFX_Sprite(&display);
               sprite->createSprite(32, 32);
               available.push_back(sprite);
           }
           
           auto sprite = available.back();
           available.pop_back();
           inUse.push_back(sprite);
           return sprite;
       }
       
       void release(LGFX_Sprite* sprite) {
           auto it = std::find(inUse.begin(), inUse.end(), sprite);
           if (it != inUse.end()) {
               inUse.erase(it);
               available.push_back(sprite);
           }
       }
   };
   ```

3. **Animation Glitches**
   ```cpp
   // Implement frame timing
   class AnimationTimer {
       uint32_t lastFrame;
       float accumulator;
       const float timestep = 1.0f / 60.0f;
       
   public:
       void update() {
           uint32_t current = millis();
           float deltaTime = (current - lastFrame) / 1000.0f;
           lastFrame = current;
           
           accumulator += deltaTime;
           
           while (accumulator >= timestep) {
               updatePhysics(timestep);
               accumulator -= timestep;
           }
           
           float alpha = accumulator / timestep;
           render(alpha);
       }
   };
   ``` 