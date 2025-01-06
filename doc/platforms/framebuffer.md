# Linux Framebuffer Platform Support

The Linux Framebuffer platform support in LovyanGFX enables direct rendering to the Linux framebuffer device. This platform is particularly useful for embedded Linux systems, single-board computers, and systems without a full graphics stack.

## Architecture Overview

The framebuffer platform implementation consists of several key components:

- `Panel_fb`: Core display panel implementation for Linux framebuffer
- Direct memory mapping of framebuffer device
- Support for various color depths and pixel formats
- Automatic framebuffer device detection

## Configuration

### Basic Setup

```cpp
#include <LGFX_Linux_FB.hpp>

static LGFX display;

void setup() {
    // Configure the framebuffer device
    auto cfg = display.config();
    cfg.memory_width = 800;    // Display width
    cfg.memory_height = 480;   // Display height
    cfg.panel_width = 800;     // Visible width
    cfg.panel_height = 480;    // Visible height
    display.config(cfg);
    
    // Initialize the display
    display.init();
}
```

### Device Selection

The framebuffer platform can be configured to use specific devices:

```cpp
// Use default framebuffer device
display.setDeviceName("/dev/fb0");

// Use specific framebuffer device
display.setDeviceName("/dev/fb1");

// Use device by name (auto-detection)
display.setDeviceName("st7789");  // Will search for matching device
```

## Color Formats

### Supported Color Depths

The framebuffer platform supports multiple color formats:

```cpp
// 16-bit RGB565
display.setColorDepth(color_depth_t::rgb565_2Byte);

// 24-bit RGB888
display.setColorDepth(color_depth_t::rgb888_3Byte);

// 32-bit ARGB8888
display.setColorDepth(color_depth_t::argb8888_4Byte);
```

### Color Space Handling

The platform handles color format conversion automatically:

```cpp
void drawPixel(uint_fast16_t x, uint_fast16_t y, uint32_t color) {
    switch (_write_depth) {
        case color_depth_t::rgb565_2Byte:
            // 16-bit color handling
            break;
        case color_depth_t::rgb888_3Byte:
            // 24-bit color handling
            break;
        case color_depth_t::argb8888_4Byte:
            // 32-bit color with alpha
            break;
    }
}
```

## Memory Management

### Framebuffer Mapping

The platform uses memory mapping for efficient framebuffer access:

```cpp
bool initFrameBuffer() {
    // Open framebuffer device
    _fbfd = open(_config_detail.device_name, O_RDWR);
    
    // Get fixed and variable screen information
    ioctl(_fbfd, FBIOGET_FSCREENINFO, &_fix_info);
    ioctl(_fbfd, FBIOGET_VSCREENINFO, &_var_info);
    
    // Map framebuffer to memory
    _screensize = _fix_info.smem_len;
    _fbp = (char*)mmap(0, _screensize, 
                       PROT_READ | PROT_WRITE, 
                       MAP_SHARED, _fbfd, 0);
    return true;
}
```

### Memory Operations

Efficient memory operations for drawing:

```cpp
void copyRect(uint_fast16_t dst_x, uint_fast16_t dst_y,
              uint_fast16_t w, uint_fast16_t h,
              uint_fast16_t src_x, uint_fast16_t src_y) {
    size_t bytes = _write_bits >> 3;
    size_t len = w * bytes;
    int32_t add = _var_info.xres * bytes;
    
    char* src = _fbp + (src_x * bytes + src_y * _fix_info.line_length);
    char* dst = _fbp + (dst_x * bytes + dst_y * _fix_info.line_length);
    
    do {
        memmove(dst, src, len);
        src += add;
        dst += add;
    } while (--h);
}
```

## Performance Features

### Direct Memory Access

The platform provides direct memory access for efficient rendering:

- Memory-mapped framebuffer access
- Block transfer operations
- Hardware-aligned memory operations

### Optimization Techniques

```cpp
void writeBlock(uint32_t rawcolor, uint32_t length) {
    // Optimize for consecutive writes
    uint32_t h = 1;
    auto w = std::min<uint32_t>(length, _xe + 1 - _xpos);
    if (length >= (w << 1) && _xpos == _xs) {
        h = std::min<uint32_t>(length / w, _ye + 1 - _ypos);
    }
    writeFillRectPreclipped(_xpos, _ypos, w, h, rawcolor);
}
```

## Advanced Usage

### Device Auto-detection

The platform can automatically detect and use the appropriate framebuffer device:

```cpp
bool detectFramebuffer() {
    DIR* sysfs_graphics = opendir("/sys/class/graphics");
    struct dirent* entry;
    
    while((entry = readdir(sysfs_graphics)) != NULL) {
        if (entry->d_type == DT_LNK) {
            // Check device name match
            std::string path = "/sys/class/graphics/";
            path.append(entry->d_name);
            path.append("/name");
            
            // Read and match device name
            std::ifstream fs(path.c_str());
            if (fs.is_open()) {
                // Match device name
                if (buffer.str().find(_config_detail.device_name) 
                    != std::string::npos) {
                    return true;
                }
            }
        }
    }
    return false;
}
```

### Rotation Support

Handle display rotation with memory reorganization:

```cpp
void setRotation(uint_fast8_t r) {
    r &= 7;
    _rotation = r;
    _internal_rotation = ((r + _cfg.offset_rotation) & 3) 
                      | ((r & 4) ^ (_cfg.offset_rotation & 4));
    
    _width = _cfg.panel_width;
    _height = _cfg.panel_height;
    if (_internal_rotation & 1) {
        std::swap(_width, _height);
    }
}
```

## Limitations and Considerations

1. Hardware Dependencies
   - Requires Linux framebuffer driver support
   - Color depth limitations based on hardware
   - Performance depends on hardware capabilities

2. Platform Specifics
   - Linux-specific implementation
   - Requires root access for some operations
   - Device-specific quirks may exist

3. Memory Usage
   - Direct memory mapping requirements
   - System memory limitations apply
   - Shared memory considerations

## Best Practices

1. Device Setup
   - Check framebuffer device permissions
   - Verify color depth support
   - Handle device detection gracefully

2. Performance Optimization
   - Use block operations when possible
   - Minimize memory copies
   - Align operations to hardware boundaries

3. Error Handling
   - Check device availability
   - Handle memory mapping failures
   - Manage device permissions

## Example Applications

### Basic Drawing

```cpp
void drawPattern() {
    // Draw a test pattern
    for (int y = 0; y < display.height(); y++) {
        for (int x = 0; x < display.width(); x++) {
            uint32_t color = (x ^ y) & 0xFF;
            display.drawPixel(x, y, color);
        }
    }
}
```

### Hardware Acceleration

```cpp
void fillScreen() {
    // Use hardware-accelerated fill when available
    if (_var_info.accel_flags & FB_ACCEL_FILL_RECT) {
        // Use hardware fill
        struct fb_fillrect rect;
        rect.dx = 0;
        rect.dy = 0;
        rect.width = _width;
        rect.height = _height;
        rect.color = color;
        ioctl(_fbfd, FBIO_FILLRECT, &rect);
    } else {
        // Fallback to software fill
        writeFillRectPreclipped(0, 0, _width, _height, color);
    }
}
``` 