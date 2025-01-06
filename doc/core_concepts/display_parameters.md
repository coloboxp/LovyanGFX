# Display Parameters and Configuration

LovyanGFX provides extensive configuration options for display panels through a structured configuration system. This document covers the essential display parameters and their configuration.

## Display Configuration Structure

The display configuration is managed through the `config_t` structure in the `Panel_Device` class. Here's a detailed overview of the key parameters:

```cpp
struct config_t {
    // Display Dimensions
    uint16_t memory_width;     // Driver IC's supported maximum width
    uint16_t memory_height;    // Driver IC's supported maximum height
    uint16_t panel_width;      // Actual displayable width
    uint16_t panel_height;     // Actual displayable height
    
    // Display Offsets
    int16_t offset_x;         // Panel's X-axis offset
    int16_t offset_y;         // Panel's Y-axis offset
    uint8_t offset_rotation;  // Rotation offset value (0-7, 4-7 for inverted)
    
    // Read Operations
    uint8_t dummy_read_pixel; // Dummy read bits before pixel data
    uint8_t dummy_read_bits;  // Dummy read bits before non-pixel data
    bool readable;            // Enable data read operations
    
    // Display Options
    bool invert;             // Invert panel brightness
    bool rgb_order;          // true for RGB order, false for BGR
    bool dlen_16bit;         // Use 16-bit data length
    bool bus_shared;         // Bus shared with SD card
};
```

## Basic Display Setup

The simplest way to initialize a display is using auto-detection:

```cpp
#define LGFX_AUTODETECT
#include <LovyanGFX.hpp>

LGFX lcd;

void setup() {
    lcd.init();
    lcd.setRotation(1);        // Set rotation (0-3)
    lcd.setBrightness(128);    // Set brightness (0-255)
}
```

## Manual Display Configuration

For custom display configurations, create a class derived from `LGFX_Device`:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Panel Configuration
        {
            auto cfg = _panel_instance.config();
            
            // Display dimensions
            cfg.memory_width  = 240;    // Maximum supported width
            cfg.memory_height = 320;    // Maximum supported height
            cfg.panel_width   = 240;    // Actual display width
            cfg.panel_height  = 320;    // Actual display height
            
            // Display offsets and rotation
            cfg.offset_x = 0;           // X-axis offset
            cfg.offset_y = 0;           // Y-axis offset
            cfg.offset_rotation = 0;    // Rotation offset
            
            // Read operations
            cfg.dummy_read_pixel = 8;   // Dummy bits before pixel read
            cfg.dummy_read_bits = 1;    // Dummy bits before other reads
            cfg.readable = true;        // Enable read operations
            
            // Display options
            cfg.invert = false;         // Panel invert
            cfg.rgb_order = false;      // RGB/BGR order
            cfg.dlen_16bit = false;     // 16-bit data length
            cfg.bus_shared = true;      // Bus shared with SD card
            
            _panel_instance.config(cfg);
        }
    }
};
```

## Display Parameters

### Resolution and Memory Layout

The display configuration includes both the physical panel dimensions and the memory layout:

- `memory_width` and `memory_height`: Define the maximum dimensions supported by the display controller
- `panel_width` and `panel_height`: Specify the actual visible display area
- `offset_x` and `offset_y`: Set the starting position within the controller's memory

### Color Configuration

Color handling can be configured through several parameters:

- `rgb_order`: Determines the color byte order (RGB or BGR)
- `invert`: Inverts the display colors when true
- `color_depth`: Set through `setColorDepth()` method

```cpp
// Available color depths
enum color_depth_t {
    palette_1bit = 1,     // 2 colors
    palette_2bit = 2,     // 4 colors
    palette_4bit = 4,     // 16 colors
    palette_8bit = 8,     // 256 colors
    rgb565_2Byte = 16,    // 65536 colors
    rgb888_3Byte = 24,    // 16.7M colors
    argb8888_4Byte = 32   // 16.7M colors with alpha
};
```

### Display Rotation

Rotation can be configured through two parameters:

- `offset_rotation`: Base rotation offset (0-7, where 4-7 indicate inverted display)
- Runtime rotation: Set through `setRotation(uint_fast8_t r)` method (0-3)

### Read Operations

Some displays support reading back pixel data. Configure read operations with:

- `readable`: Enable/disable read functionality
- `dummy_read_pixel`: Number of dummy bits before pixel data
- `dummy_read_bits`: Number of dummy bits before non-pixel data

### Bus Configuration

When sharing the display bus with other devices:

- `bus_shared`: Set to true when sharing bus with SD card
- `dlen_16bit`: Use 16-bit data length for communication

## Performance Optimization

To optimize display performance:

1. Set appropriate bus speeds for read and write operations
2. Use DMA when available for faster data transfer
3. Minimize read operations when possible
4. Batch updates using `startWrite()` and `endWrite()`
5. Use appropriate color depth for your application

## Common Issues and Solutions

1. Inverted Colors
   - Check `invert` parameter
   - Verify `rgb_order` setting

2. Display Offset
   - Adjust `offset_x` and `offset_y`
   - Verify `memory_width` and `memory_height`

3. Rotation Issues
   - Check `offset_rotation` value
   - Verify rotation setting in `setRotation()`

4. Communication Problems
   - Verify bus configuration
   - Check `bus_shared` setting if using SD card
   - Adjust dummy read values if seeing garbage data 