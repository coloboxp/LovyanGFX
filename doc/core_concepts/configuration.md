# LovyanGFX Configuration Guide

## Color Handling

### Color Types and Formats

LovyanGFX supports multiple color formats:

```cpp
// Basic color types
using rgb332_t  = uint8_t;    // 8-bit RGB (3:3:2)
using rgb565_t  = uint16_t;   // 16-bit RGB (5:6:5)
using rgb888_t  = uint32_t;   // 24-bit RGB (8:8:8)
using argb8888_t = uint32_t;  // 32-bit ARGB (8:8:8:8)
using grayscale_t = uint8_t;  // 8-bit grayscale
```

### Color Depth Configuration

Set color depth for display or sprite:

```cpp
// Available color depths
enum color_depth_t {
    palette_1bit = 1,    // 2 colors, monochrome
    palette_2bit = 2,    // 4 colors
    palette_4bit = 4,    // 16 colors
    palette_8bit = 8,    // 256 colors
    rgb565_2Byte = 16,  // 65536 colors (RGB565)
    rgb888_3Byte = 24,  // 16.7M colors (RGB888)
    argb8888_4Byte = 32 // 16.7M colors with alpha
};

// Set color depth
void setColorDepth(color_depth_t depth);
```

### Color Order Configuration

Configure RGB/BGR color order in panel configuration:

```cpp
auto cfg = _panel_instance.config();
cfg.rgb_order = true;    // true for RGB order, false for BGR
_panel_instance.config(cfg);
```

## Touch Configuration

### Touch Calibration

LovyanGFX provides built-in touch calibration support:

```cpp
// Calibrate touch screen interactively
void calibrateTouch(uint16_t *parameters, const T& color_fg, const T& color_bg, uint8_t size = 10);

// Apply saved calibration parameters
void setTouchCalibrate(uint16_t *parameters);
```

### Touch Parameters

Configure touch interface with specific parameters:

```cpp
auto cfg = touch->config();

// Touch area boundaries
cfg.x_min = 0;
cfg.x_max = 319;
cfg.y_min = 0;
cfg.y_max = 479;

// Communication settings
cfg.freq = 400000;      // I2C/SPI frequency
cfg.i2c_addr = 0x38;    // I2C address (if applicable)
cfg.pin_int = GPIO_NUM_38;   // Interrupt pin
cfg.pin_rst = GPIO_NUM_16;   // Reset pin

// Bus configuration
cfg.bus_shared = false;  // Whether bus is shared with display

// Rotation offset
cfg.offset_rotation = 0; // Touch coordinate rotation offset

touch->config(cfg);
```

## Display Parameters

### Panel Configuration

Essential display parameters:

```cpp
auto cfg = _panel_instance.config();

// Display dimensions
cfg.memory_width  = 240;  // Frame buffer width
cfg.memory_height = 320;  // Frame buffer height
cfg.panel_width   = 240;  // Visible display width
cfg.panel_height  = 320;  // Visible display height

// Display position adjustment
cfg.offset_x = 0;      // X offset for display area
cfg.offset_y = 0;      // Y offset for display area

// Display options
cfg.readable    = true;   // Enable read operations
cfg.invert      = false;  // Invert display colors
cfg.rgb_order   = true;   // RGB color order (vs BGR)
cfg.dlen_16bit  = false;  // 16-bit data length mode
cfg.bus_shared  = true;   // Bus shared with other devices

_panel_instance.config(cfg);
```

### Backlight Control

Configure backlight settings:

```cpp
// Set brightness (0-255)
void setBrightness(uint8_t brightness);

// Power management
void sleep();           // Enter sleep mode
void wakeup();         // Exit sleep mode
void powerSave(bool);  // Enable/disable power save mode
```

## Performance Optimization

### Bus Configuration

Optimize communication speed:

```cpp
// SPI configuration
cfg.spi_mode = 0;             // SPI mode (0-3)
cfg.freq_write = 40000000;    // Write clock frequency (Hz)
cfg.freq_read  = 16000000;    // Read clock frequency (Hz)
cfg.spi_3wire = true;         // Enable 3-wire SPI mode

// I2C configuration
cfg.i2c_port = 0;             // I2C port number
cfg.freq = 400000;            // I2C clock frequency (Hz)

// Parallel interface
cfg.pin_wr = 4;               // Write strobe pin
cfg.pin_rd = 5;               // Read strobe pin
cfg.pin_rs = 6;               // Register select pin
cfg.freq_write = 20000000;    // Write clock frequency (Hz)
```

### DMA Settings

Enable DMA for improved performance:

```cpp
auto cfg = _panel_instance.config_detail();
cfg.use_psram = true;         // Use PSRAM if available
cfg.dma_channel = 1;          // DMA channel (platform specific)
cfg.dma_buffer_size = 1024;   // DMA buffer size
_panel_instance.config_detail(cfg);
```

## Error Handling

Implement proper error checking:

```cpp
// Initialize display
if (!display.init()) {
    // Handle initialization failure
}

// Touch initialization
if (!touch->init()) {
    // Handle touch initialization failure
}

// Bus transaction
if (!bus->beginTransaction()) {
    // Handle bus error
}
```

The configuration options provide extensive control over the display, touch, and communication interfaces. Proper configuration is essential for optimal performance and functionality. 