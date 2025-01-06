# SDL Platform Guide

This guide covers the SDL (Simple DirectMedia Layer) implementation in LovyanGFX, which provides a simulation environment for display development and testing on desktop platforms.

## Overview

The SDL implementation allows you to:
- Develop and test display code on your desktop
- Debug graphics algorithms without hardware
- Simulate various display configurations
- Profile performance on your development machine

## Architecture

```
SDL Implementation Architecture:
┌─────────────────────────┐
│    LovyanGFX Core       │
├─────────────┬───────────┤
│ SDL Window  │ SDL       │
│ Management  │ Renderer  │
├─────────────┼───────────┤
│ Event      │ Frame     │
│ Handling   │ Buffer    │
└─────────────┴───────────┘
```

## Configuration

### Display Configuration
```cpp
struct Panel_SDL::config_t {
    uint16_t memory_width;      // Framebuffer width
    uint16_t memory_height;     // Framebuffer height
    uint16_t panel_width;       // Display panel width
    uint16_t panel_height;      // Display panel height
    uint_fast8_t offset_x;      // X offset of panel
    uint_fast8_t offset_y;      // Y offset of panel
    uint_fast8_t offset_rotation; // Display rotation offset
    uint_fast8_t rotation;      // Display rotation (0-3)
    bool rgb_order;             // True = RGB order, False = BGR order
    bool invert;                // Color inversion
    uint32_t freq_write;        // Simulation refresh rate
    union {
        uint8_t raw;
        struct {
            uint8_t reverse_invert : 1;
            uint8_t rgb666_override : 1;
        };
    } cfg;
};
```

### Window Configuration
```cpp
struct LGFX_SDL::config_t {
    std::string title;          // Window title
    int32_t window_width;       // Window width
    int32_t window_height;      // Window height
    int32_t window_x;          // Window X position
    int32_t window_y;          // Window Y position
    float scale_x;             // X scaling factor
    float scale_y;             // Y scaling factor
    uint32_t refresh_rate;     // Window refresh rate
    bool high_dpi;            // Enable high DPI support
};
```

## Implementation Features

### Window Management
```cpp
// Window initialization
void LGFX_SDL::init_impl(bool use_reset) {
    if (!_initialized) {
        SDL_Init(SDL_INIT_VIDEO);
        _window = SDL_CreateWindow(
            _config.title.c_str(),
            _config.window_x,
            _config.window_y,
            _config.window_width,
            _config.window_height,
            SDL_WINDOW_SHOWN | (_config.high_dpi ? SDL_WINDOW_ALLOW_HIGHDPI : 0)
        );
        _renderer = SDL_CreateRenderer(_window, -1, SDL_RENDERER_ACCELERATED);
        _texture = SDL_CreateTexture(
            _renderer,
            SDL_PIXELFORMAT_RGB888,
            SDL_TEXTUREACCESS_STREAMING,
            _panel_width,
            _panel_height
        );
        _initialized = true;
    }
}
```

### Event Handling
```cpp
// Process SDL events
void processEvents(void) {
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        switch (event.type) {
            case SDL_QUIT:
                exit(0);
                break;
            case SDL_MOUSEBUTTONDOWN:
                touch_point.x = event.button.x / _config.scale_x;
                touch_point.y = event.button.y / _config.scale_y;
                touch_point.size = 1;
                touch_point.id = 0;
                _touch_state = true;
                break;
            case SDL_MOUSEBUTTONUP:
                _touch_state = false;
                break;
            case SDL_MOUSEMOTION:
                if (_touch_state) {
                    touch_point.x = event.motion.x / _config.scale_x;
                    touch_point.y = event.motion.y / _config.scale_y;
                }
                break;
        }
    }
}
```

### Frame Buffer Management
```cpp
// Update display buffer
void Panel_SDL::writePixels(pixelcopy_t* param, uint32_t len, bool use_dma) {
    uint32_t* buffer = (uint32_t*)_write_buffer;
    uint32_t index = _write_y * panel_width + _write_x;
    
    param->fp_copy(buffer + index, 0, len, param);
    _write_x += len;
    if (_write_x >= panel_width) {
        _write_y++;
        _write_x = 0;
    }
}

// Render frame
void Panel_SDL::display(uint_fast16_t x, uint_fast16_t y, uint_fast16_t w, uint_fast16_t h) {
    SDL_UpdateTexture(_texture, NULL, _write_buffer, panel_width * sizeof(uint32_t));
    SDL_RenderClear(_renderer);
    SDL_RenderCopy(_renderer, _texture, NULL, NULL);
    SDL_RenderPresent(_renderer);
}
```

## Usage Example

Basic setup for SDL simulation:

```cpp
class LGFX : public lgfx::LGFX_SDL {
public:
    LGFX(void) {
        // Configure SDL window
        auto cfg = config();
        cfg.title = "LovyanGFX SDL Window";
        cfg.window_width = 320;
        cfg.window_height = 240;
        cfg.window_x = SDL_WINDOWPOS_CENTERED;
        cfg.window_y = SDL_WINDOWPOS_CENTERED;
        cfg.refresh_rate = 60;
        cfg.high_dpi = true;
        setConfig(cfg);
        
        // Configure panel
        auto panel = new lgfx::Panel_SDL();
        auto panel_cfg = panel->config();
        panel_cfg.memory_width = 320;
        panel_cfg.memory_height = 240;
        panel_cfg.panel_width = 320;
        panel_cfg.panel_height = 240;
        panel_cfg.offset_x = 0;
        panel_cfg.offset_y = 0;
        panel_cfg.offset_rotation = 0;
        panel_cfg.rgb_order = true;
        panel->setConfig(panel_cfg);
        setPanel(panel);
    }
};

LGFX display;

int main(int argc, char* argv[]) {
    display.init();
    display.setRotation(0);
    display.setBrightness(128);
    
    // Basic drawing
    display.fillScreen(TFT_BLACK);
    display.setTextColor(TFT_WHITE);
    display.drawString("Hello SDL!", 10, 10);
    
    // Main loop
    while (1) {
        display.processEvents();
        // Your drawing code here
        display.display();
        SDL_Delay(1000 / 60);  // Cap at 60 FPS
    }
    
    return 0;
}
```

## Best Practices

1. **Window Management**
   - Set appropriate window size for your display
   - Enable high DPI support for Retina displays
   - Handle window events properly
   - Maintain aspect ratio when scaling

2. **Performance Optimization**
   - Use hardware acceleration when available
   - Implement double buffering
   - Batch drawing operations
   - Control frame rate for consistent performance

3. **Event Handling**
   - Process all SDL events in the main loop
   - Implement touch/mouse input simulation
   - Handle window resize events
   - Clean up resources on exit

4. **Debug Features**
   - Add debug overlays
   - Implement frame timing
   - Log drawing operations
   - Simulate different display configurations

## Common Issues and Solutions

1. **Window Creation**
```cpp
// Handle window creation failure
if (!_window) {
    printf("SDL_CreateWindow Error: %s\n", SDL_GetError());
    return false;
}

// Handle renderer creation failure
if (!_renderer) {
    SDL_DestroyWindow(_window);
    printf("SDL_CreateRenderer Error: %s\n", SDL_GetError());
    return false;
}
```

2. **Performance Issues**
```cpp
// Enable vsync for smooth rendering
SDL_SetHint(SDL_HINT_RENDER_VSYNC, "1");

// Use hardware acceleration
SDL_SetHint(SDL_HINT_RENDER_DRIVER, "opengl");

// Batch rendering operations
void batchDraw() {
    SDL_RenderClear(_renderer);
    // Perform all drawing operations here
    SDL_RenderPresent(_renderer);
}
```

3. **Event Handling**
```cpp
// Handle window events
case SDL_WINDOWEVENT:
    switch (event.window.event) {
        case SDL_WINDOWEVENT_RESIZED:
            updateWindowSize(event.window.data1, event.window.data2);
            break;
        case SDL_WINDOWEVENT_EXPOSED:
            redrawWindow();
            break;
    }
    break;
```

4. **Resource Management**
```cpp
// Clean up resources
void cleanup() {
    if (_texture) SDL_DestroyTexture(_texture);
    if (_renderer) SDL_DestroyRenderer(_renderer);
    if (_window) SDL_DestroyWindow(_window);
    SDL_Quit();
}
```

## Debug Features

### Frame Timing
```cpp
// Measure frame time
void measureFrameTime() {
    static uint32_t last_time = 0;
    uint32_t current_time = SDL_GetTicks();
    float frame_time = (current_time - last_time) / 1000.0f;
    last_time = current_time;
    
    char buf[32];
    sprintf(buf, "Frame Time: %.2fms", frame_time * 1000.0f);
    display.drawString(buf, 10, display.height() - 20);
}
```

### Debug Overlay
```cpp
// Draw debug information
void drawDebugOverlay() {
    display.setTextColor(TFT_YELLOW);
    display.drawString("Debug Mode", 10, 10);
    
    // Display configuration
    char buf[64];
    sprintf(buf, "Size: %dx%d", display.width(), display.height());
    display.drawString(buf, 10, 30);
    
    sprintf(buf, "Rotation: %d", display.getRotation());
    display.drawString(buf, 10, 50);
    
    // Touch/mouse position
    if (_touch_state) {
        sprintf(buf, "Touch: %d,%d", touch_point.x, touch_point.y);
        display.drawString(buf, 10, 70);
    }
}
```

## External Resources

### Documentation
- [SDL Documentation](https://wiki.libsdl.org/)
- [SDL2 API Reference](https://wiki.libsdl.org/SDL2/CategoryAPI)
- [SDL2 Migration Guide](https://wiki.libsdl.org/SDL2/MigrationGuide)

### Tools
- [SDL2 Development Libraries](https://github.com/libsdl-org/SDL/releases)
- [SDL2 Test Suite](https://github.com/libsdl-org/SDL/tree/main/test)

### Example Projects
- [Basic SDL Example](../examples_for_PC/SDL/README.md)
- [Touch Input Simulation](../examples_for_PC/Touch/README.md)
- [Performance Tests](../examples_for_PC/Benchmark/README.md) 