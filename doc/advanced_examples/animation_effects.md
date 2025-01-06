# Animation and Effects Guide

This guide demonstrates advanced animation and effects techniques using LovyanGFX's sprite system and hardware acceleration features.

## Sprite-Based Animation System

### Animation Framework

```cpp
class AnimationSystem {
private:
    LGFX* _display;
    LGFX_Sprite* _buffer;
    LGFX_Sprite* _frames;
    size_t _frame_count;
    uint32_t _last_update;
    uint32_t _frame_duration;
    bool _use_double_buffer;

public:
    AnimationSystem(LGFX* display, size_t frame_count, uint32_t fps = 30) 
    : _display(display)
    , _frame_count(frame_count)
    , _frame_duration(1000 / fps)
    , _last_update(0)
    {
        // Create double buffer for smooth rendering
        _buffer = new LGFX_Sprite(_display);
        _buffer->createSprite(_display->width(), _display->height());
        
        // Create animation frames
        _frames = new LGFX_Sprite[frame_count](_buffer);
        for (size_t i = 0; i < frame_count; i++) {
            _frames[i].createSprite(_display->width(), _display->height());
            _frames[i].setColorDepth(16);
        }
    }

    void update() {
        uint32_t current_time = millis();
        if (current_time - _last_update >= _frame_duration) {
            _last_update = current_time;
            renderNextFrame();
        }
    }

    void renderNextFrame() {
        static size_t current_frame = 0;
        
        // Clear buffer
        _buffer->fillScreen(TFT_BLACK);
        
        // Draw current frame
        _frames[current_frame].pushSprite(0, 0);
        
        // Push to display
        _buffer->pushSprite(0, 0);
        
        // Move to next frame
        current_frame = (current_frame + 1) % _frame_count;
    }
};
```

## Transition Effects

### Fade Transition

```cpp
class FadeTransition {
private:
    LGFX* _display;
    LGFX_Sprite* _from_sprite;
    LGFX_Sprite* _to_sprite;
    uint32_t _duration;
    uint32_t _start_time;
    bool _in_transition;

public:
    FadeTransition(LGFX* display, uint32_t duration_ms = 500)
    : _display(display)
    , _duration(duration_ms)
    , _in_transition(false)
    {
        _from_sprite = new LGFX_Sprite(_display);
        _to_sprite = new LGFX_Sprite(_display);
        
        _from_sprite->createSprite(_display->width(), _display->height());
        _to_sprite->createSprite(_display->width(), _display->height());
    }

    void start(LGFX_Sprite* from, LGFX_Sprite* to) {
        _from_sprite->pushImage(0, 0, _display->width(), _display->height(), from);
        _to_sprite->pushImage(0, 0, _display->width(), _display->height(), to);
        _start_time = millis();
        _in_transition = true;
    }

    bool update() {
        if (!_in_transition) return false;
        
        uint32_t current_time = millis();
        uint32_t elapsed = current_time - _start_time;
        
        if (elapsed >= _duration) {
            _to_sprite->pushSprite(0, 0);
            _in_transition = false;
            return false;
        }
        
        float progress = (float)elapsed / _duration;
        uint8_t alpha = progress * 255;
        
        _from_sprite->pushSprite(0, 0);
        _to_sprite->pushSprite(0, 0, alpha);
        
        return true;
    }
};
```

## Visual Effects

### Particle System

```cpp
struct Particle {
    float x, y;
    float vx, vy;
    float life;
    uint32_t color;
};

class ParticleSystem {
private:
    LGFX* _display;
    LGFX_Sprite* _buffer;
    std::vector<Particle> _particles;
    uint32_t _last_update;
    uint32_t _update_interval;

public:
    ParticleSystem(LGFX* display, size_t max_particles = 100)
    : _display(display)
    , _last_update(0)
    , _update_interval(16)  // ~60fps
    {
        _buffer = new LGFX_Sprite(_display);
        _buffer->createSprite(_display->width(), _display->height());
        _particles.reserve(max_particles);
    }

    void addParticle(float x, float y, float vx, float vy, uint32_t color) {
        if (_particles.size() >= _particles.capacity()) return;
        
        Particle p = {
            .x = x,
            .y = y,
            .vx = vx,
            .vy = vy,
            .life = 1.0f,
            .color = color
        };
        _particles.push_back(p);
    }

    void update() {
        uint32_t current_time = millis();
        if (current_time - _last_update < _update_interval) return;
        _last_update = current_time;
        
        _buffer->fillScreen(TFT_BLACK);
        
        for (auto it = _particles.begin(); it != _particles.end();) {
            it->x += it->vx;
            it->y += it->vy;
            it->life -= 0.01f;
            
            if (it->life <= 0) {
                it = _particles.erase(it);
                continue;
            }
            
            uint8_t alpha = it->life * 255;
            _buffer->drawPixel(it->x, it->y, it->color);
            ++it;
        }
        
        _buffer->pushSprite(0, 0);
    }
};
```

## Performance Optimization

### Memory Management

1. Use PSRAM for large sprites:
```cpp
sprite.setPsram(true);  // Enable PSRAM usage
```

2. Reuse sprite buffers:
```cpp
// Create once
sprite.createSprite(width, height);

// Reuse in animation loop
sprite.fillScreen(TFT_BLACK);  // Clear for reuse
```

3. Monitor memory usage:
```cpp
size_t free_heap = heap_caps_get_free_size(MALLOC_CAP_DMA);
size_t required = width * height * (bits_per_pixel / 8);
if (free_heap < required) {
    // Handle low memory condition
}
```

### Hardware Acceleration

1. Enable DMA for faster transfers:
```cpp
auto cfg = display.config();
cfg.dma_channel = 1;  // Use DMA channel 1
cfg.use_psram = true; // Enable PSRAM
```

2. Batch drawing operations:
```cpp
display.startWrite();  // Start batch
// Multiple drawing operations
display.endWrite();    // End batch
```

3. Use appropriate color depth:
```cpp
sprite.setColorDepth(16);  // Use 16-bit color for better performance
```

## Best Practices

1. Animation Timing
   - Use consistent frame timing
   - Implement frame rate control
   - Handle frame drops gracefully

2. Memory Management
   - Monitor heap usage
   - Implement proper cleanup
   - Handle allocation failures

3. Effect Combinations
   - Layer effects appropriately
   - Manage transition timing
   - Control effect priorities

4. Error Handling
   - Validate sprite creation
   - Check memory availability
   - Implement graceful degradation 