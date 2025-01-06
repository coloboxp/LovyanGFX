# Enumeration Types

LovyanGFX uses various enumeration types to define display configurations, color modes, and hardware settings.

## Display Configuration

```cpp
namespace lgfx
{
    enum panel_type_t {
        panel_none,
        panel_ili9163,
        panel_ili9341,
        panel_ili9342,
        panel_ili9481,
        panel_ili9486,
        panel_ili9488,
        panel_st7735,
        panel_st7789,
        panel_st7796,
        panel_ssd1351,
        panel_ssd1963,
        panel_gc9a01,
    };

    enum bus_type_t {
        bus_unknown,
        bus_spi,
        bus_i2c,
        bus_parallel8,
        bus_parallel16,
    };
}
```

## Color Modes

```cpp
enum color_mode_t {
    color_mode_rgb332,   // 8bit  RGB 3:3:2
    color_mode_rgb565,   // 16bit RGB 5:6:5
    color_mode_rgb888,   // 24bit RGB 8:8:8
    color_mode_argb8888, // 32bit ARGB 8:8:8:8
    color_mode_palette,  // Indexed color mode
};

enum swap_type_t {
    swap_none = 0,
    swap_rgb = 1,      // Swap R and B
    swap_bytes = 2,    // Swap bytes
    swap_bits = 4,     // Reverse bit order
};
```

## Drawing Configuration

```cpp
enum text_style_t {
    text_style_none    = 0,
    text_style_bold    = 1,
    text_style_italic  = 2,
    text_style_full    = 3,
};

enum datum_t {
    top_left      = 0,
    top_center    = 1,
    top_right     = 2,
    middle_left   = 4,
    middle_center = 5,
    middle_right  = 6,
    bottom_left   = 8,
    bottom_center = 9,
    bottom_right  = 10,
    baseline_left = 16,
    baseline_center = 17,
    baseline_right = 18,
};
```

## Hardware Control

```cpp
enum pin_status_t {
    pin_no_change = -1,  // Don't change pin state
    pin_low = 0,         // Set pin LOW
    pin_high = 1,        // Set pin HIGH
    pin_input = 2,       // Set pin as INPUT
    pin_output = 3,      // Set pin as OUTPUT
};

enum spi_mode_t {
    spi_mode0 = 0,  // CPOL=0, CPHA=0
    spi_mode1 = 1,  // CPOL=0, CPHA=1
    spi_mode2 = 2,  // CPOL=1, CPHA=0
    spi_mode3 = 3,  // CPOL=1, CPHA=1
};
```

## Display Rotation

```cpp
enum rotation_t : uint_fast8_t {
    rotation_0   = 0,    // No rotation
    rotation_90  = 1,    // 90 degree clockwise
    rotation_180 = 2,    // 180 degree rotation
    rotation_270 = 3,    // 270 degree clockwise
};

enum direction_t : uint_fast8_t {
    direction_up    = 0,
    direction_right = 1,
    direction_down  = 2,
    direction_left  = 3,
};
```

## Memory Management

```cpp
enum memory_type_t {
    memory_normal = 0,      // Normal memory allocation
    memory_dma = 1,         // DMA capable memory
    memory_psram = 2,       // External PSRAM
};

enum endian_t {
    endian_little,          // Little endian
    endian_big,            // Big endian
};
``` 