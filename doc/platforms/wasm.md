# WebAssembly Support

LovyanGFX provides WebAssembly support for running graphics applications in web browsers.

## Setup

### CMake Configuration

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.12)

project(lgfx_wasm)

# Configure Emscripten
set(CMAKE_TOOLCHAIN_FILE $ENV{EMSDK}/upstream/emscripten/cmake/Modules/Platform/Emscripten.cmake)

# Add LovyanGFX
add_subdirectory(LovyanGFX)

add_executable(${PROJECT_NAME} 
    main.cpp
    user_code.cpp
)

# WASM-specific settings
set_target_properties(${PROJECT_NAME} PROPERTIES
    SUFFIX ".html"
    LINK_FLAGS "-s USE_SDL=2 -s WASM=1 -s USE_SDL_IMAGE=2"
)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
    LovyanGFX::lvgl
)
```

## Basic Implementation

```cpp
#define LGFX_USE_V1
#include <LGFX_AUTODETECT.hpp>

class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_WASM _panel_instance;
    
public:
    LGFX(void) {
        auto cfg = _panel_instance.config();
        
        cfg.memory_width = 320;
        cfg.memory_height = 240;
        cfg.panel_width = 320;
        cfg.panel_height = 240;
        cfg.canvas_id = "lgfx-canvas";  // HTML canvas element ID
        
        _panel_instance.config(cfg);
        setPanel(&_panel_instance);
    }
};

LGFX lcd;

void setup() {
    lcd.init();
    lcd.setRotation(1);
    lcd.fillScreen(TFT_BLACK);
}

void loop() {
    lcd.display();  // Update canvas
    emscripten_sleep(16);  // ~60fps
}
```

## HTML Integration

```html
<!DOCTYPE html>
<html>
<head>
    <title>LovyanGFX WASM</title>
</head>
<body>
    <canvas id="lgfx-canvas"></canvas>
    <script>
        var Module = {
            canvas: document.getElementById('lgfx-canvas')
        };
    </script>
    <script src="lgfx_wasm.js"></script>
</body>
</html>
```

## Event Handling

Handle browser events:

```cpp
// JavaScript event callback
EM_BOOL mouse_callback(int eventType, const EmscriptenMouseEvent* e, void* userData) {
    switch(eventType) {
        case EMSCRIPTEN_EVENT_MOUSEDOWN:
            // Handle mouse press
            break;
        case EMSCRIPTEN_EVENT_MOUSEUP:
            // Handle mouse release
            break;
        case EMSCRIPTEN_EVENT_MOUSEMOVE:
            // Handle mouse movement
            break;
    }
    return true;
}

// Register event handlers
void setup_events() {
    emscripten_set_mousedown_callback("#lgfx-canvas", nullptr, true, mouse_callback);
    emscripten_set_mouseup_callback("#lgfx-canvas", nullptr, true, mouse_callback);
    emscripten_set_mousemove_callback("#lgfx-canvas", nullptr, true, mouse_callback);
}
```

## Performance Optimization

1. Minimize canvas updates:
```cpp
// Only update changed regions
lcd.setClipRect(x, y, w, h);
lcd.display();
lcd.clearClipRect();
```

2. Use appropriate data types:
```cpp
// Use web-optimized types
using web_color_t = uint32_t;
std::vector<web_color_t> buffer;
```

## Example: Interactive Graphics

```cpp
#include <LGFX_AUTODETECT.hpp>
#include <emscripten.h>

LGFX lcd;
int touch_x = 0;
int touch_y = 0;
bool is_touching = false;

void mainloop() {
    lcd.fillScreen(TFT_BLACK);
    
    if (is_touching) {
        lcd.fillCircle(touch_x, touch_y, 20, TFT_RED);
    }
    
    lcd.display();
}

int main() {
    lcd.init();
    
    // Set main loop
    emscripten_set_main_loop(mainloop, 0, 1);
    
    return 0;
}
```

## Browser Compatibility

Ensure proper browser support:

```cpp
void check_browser_support() {
    #ifdef __EMSCRIPTEN__
    if (!emscripten_webgl_init_context_attributes(&attr)) {
        printf("Failed to initialize WebGL context\n");
        return;
    }
    #endif
}
```

## Building and Deployment

Build commands:

```bash
# Configure with Emscripten
emcmake cmake -B build

# Build
cmake --build build

# Output files
# - lgfx_wasm.html
# - lgfx_wasm.js
# - lgfx_wasm.wasm
``` 