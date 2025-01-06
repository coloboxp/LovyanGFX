# LovyanGFX Practical Usage Guide

This guide provides practical examples and common usage patterns based on the LovyanGFX example code.

## Basic Display Setup

### Simple Initialization

The most basic way to use LovyanGFX:

```cpp
#include <LovyanGFX.hpp>
#include <LGFX_AUTODETECT.hpp>  // Auto-detection support

static LGFX display;            // Display instance
static LGFX_Sprite sprite(&display); // Sprite instance (if needed)

void setup() {
    display.init();
    display.setRotation(1);     // 0-7 (4-7 for inverted)
    display.setBrightness(128); // 0-255
    display.setColorDepth(24);  // 16 or 24
}
```

### Custom Display Configuration

For custom hardware setups:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;
    lgfx::Light_PWM _light_instance;
    lgfx::Touch_FT5x06 _touch_instance;

public:
    LGFX(void) {
        // Bus configuration
        {
            auto cfg = _bus_instance.config();
            cfg.spi_host = VSPI_HOST;     // SPI host selection
            cfg.spi_mode = 0;             // SPI mode (0-3)
            cfg.freq_write = 40000000;    // Write clock (max 80MHz)
            cfg.freq_read = 16000000;     // Read clock
            cfg.spi_3wire = true;         // MOSI for both send/receive
            cfg.use_lock = true;          // Use transaction lock
            cfg.dma_channel = SPI_DMA_CH_AUTO;
            cfg.pin_sclk = 18;            // SCLK pin
            cfg.pin_mosi = 23;            // MOSI pin
            cfg.pin_miso = 19;            // MISO pin
            cfg.pin_dc = 27;              // Data/Command pin
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            cfg.pin_cs = 14;              // CS pin
            cfg.pin_rst = 33;             // Reset pin
            cfg.pin_busy = -1;            // Busy pin (-1 if unused)
            cfg.panel_width = 240;        // Display width
            cfg.panel_height = 320;       // Display height
            cfg.offset_x = 0;             // X offset
            cfg.offset_y = 0;             // Y offset
            cfg.readable = true;          // Enable read operations
            cfg.invert = false;           // Color inversion
            cfg.rgb_order = false;        // RGB order (false for BGR)
            _panel_instance.config(cfg);
        }

        // Backlight configuration
        {
            auto cfg = _light_instance.config();
            cfg.pin_bl = 32;              // Backlight pin
            cfg.invert = false;           // Invert brightness
            cfg.freq = 44100;             // PWM frequency
            cfg.pwm_channel = 7;          // PWM channel
            _light_instance.config(cfg);
            _panel_instance.setLight(&_light_instance);
        }

        setPanel(&_panel_instance);
    }
};
```

## Drawing Operations

### Basic Drawing

```cpp
// Basic shapes
display.fillScreen(TFT_BLACK);           // Clear screen
display.drawPixel(x, y, color);          // Draw pixel
display.drawLine(x0, y0, x1, y1, color); // Draw line
display.drawRect(x, y, w, h, color);     // Draw rectangle
display.fillRect(x, y, w, h, color);     // Fill rectangle
display.drawCircle(x, y, r, color);      // Draw circle
display.fillCircle(x, y, r, color);      // Fill circle

// Color handling
uint32_t color = display.color888(r, g, b);    // RGB color
uint32_t color = display.color565(r, g, b);    // RGB565 color
display.setColor(color);                       // Set draw color
```

### Optimized Drawing

```cpp
// Use startWrite/endWrite for multiple operations
display.startWrite();
for (int i = 0; i < 100; i++) {
    display.drawPixel(x + i, y + i, color);
}
display.endWrite();

// Fast pixel writing in a loop
display.startWrite();
for (uint32_t y = 0; y < height; ++y) {
    for (uint32_t x = 0; x < width; ++x) {
        display.writePixel(x, y, color);
    }
}
display.endWrite();
```

## Sprite Usage

### Basic Sprite Operations

```cpp
LGFX_Sprite sprite(&display);  // Create sprite

// Initialize sprite
sprite.createSprite(width, height);    // Create sprite buffer
sprite.setColorDepth(16);              // Set color depth (1/8/16/24)
sprite.fillSprite(TFT_BLACK);          // Fill sprite background

// Draw on sprite
sprite.drawRect(0, 0, 50, 50, TFT_RED);
sprite.fillCircle(25, 25, 20, TFT_BLUE);

// Display sprite
sprite.pushSprite(x, y);               // Copy to display
sprite.pushRotateZoom(                 // Rotate and zoom
    center_x, center_y,                // Rotation center
    angle,                             // Rotation angle (0-360)
    zoom_x, zoom_y                     // Zoom factors
);

// Clean up
sprite.deleteSprite();                 // Free sprite memory
```

### Animation Example

```cpp
LGFX_Sprite sprite(&display);
LGFX_Sprite rotated(&display);

void setup() {
    display.init();
    sprite.createSprite(100, 100);
    rotated.createSprite(100, 100);
}

void loop() {
    static float angle = 0;
    
    // Draw on base sprite
    sprite.fillSprite(TFT_BLACK);
    sprite.drawRect(10, 10, 80, 80, TFT_RED);
    
    // Rotate to second sprite
    rotated.fillSprite(TFT_BLACK);
    sprite.pushRotateZoom(&rotated, 50, 50, angle, 1.0, 1.0);
    
    // Display rotated sprite
    rotated.pushSprite(
        (display.width() - rotated.width()) / 2,
        (display.height() - rotated.height()) / 2
    );
    
    angle += 1.0;
}
```

## Touch Handling

### Touch Configuration and Calibration

```cpp
// Configure touch
auto cfg = touch->config();
cfg.x_min = 0;
cfg.x_max = 319;
cfg.y_min = 0;
cfg.y_max = 479;
cfg.pin_int = 38;   // Interrupt pin
cfg.bus_shared = true;
touch->config(cfg);

// Calibrate touch
uint16_t parameters[8];
display.calibrateTouch(
    parameters,    // Calibration data array
    TFT_WHITE,    // Foreground color
    TFT_BLACK,    // Background color
    20            // Corner marker size
);

// Save calibration for later use
display.setTouchCalibrate(parameters);
```

### Touch Input Handling

```cpp
void loop() {
    if (display.getTouch(&x, &y)) {
        // Touch detected at x,y
        display.fillCircle(x, y, 3, TFT_RED);
    }
}
```

## Performance Optimization

### Memory Management

```cpp
// Use DMA when available
auto cfg = panel->config_detail();
cfg.use_psram = true;         // Use PSRAM if available
cfg.dma_channel = 1;          // DMA channel
cfg.dma_buffer_size = 1024;   // DMA buffer size

// Optimize sprite operations
sprite.setColorDepth(16);     // Use 16-bit color for better performance
sprite.createSprite(width, height);

// Use fast drawing operations
display.startWrite();         // Start SPI transaction
display.writeFillRect(x, y, w, h, color);  // Fast filled rectangle
display.endWrite();          // End SPI transaction
```

### Bus Optimization

```cpp
// Optimize SPI settings
auto cfg = bus->config();
cfg.freq_write = 40000000;    // 40MHz SPI clock
cfg.freq_read = 16000000;     // 16MHz read clock
cfg.spi_3wire = true;         // Use 3-wire SPI
cfg.use_lock = true;          // Use transaction locking
bus->config(cfg);
```

## Error Handling

```cpp
// Initialize with error checking
if (!display.init()) {
    Serial.println("Display initialization failed!");
    return;
}

// Memory allocation check
LGFX_Sprite sprite(&display);
if (!sprite.createSprite(width, height)) {
    Serial.println("Sprite creation failed!");
    return;
}

// Bus transaction error checking
display.startWrite();
if (!display.writeCommand(cmd)) {
    Serial.println("Command write failed!");
}
display.endWrite();
```

These examples demonstrate common usage patterns and best practices for LovyanGFX. They can be adapted and combined to create more complex applications. 