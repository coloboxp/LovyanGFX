# Display Setup and Configuration

The LovyanGFX library provides a flexible system for display configuration and initialization. This document covers the core concepts and implementation details.

## Basic Display Setup

The simplest way to initialize a display is using the auto-detection feature:

```cpp
#define LGFX_AUTODETECT
#include <LovyanGFX.hpp>

LGFX lcd;  // Create display instance

void setup() {
    lcd.init();        // Initialize the display
    lcd.setRotation(1); // Optional: Set display rotation (0-3)
    lcd.setBrightness(128); // Optional: Set backlight brightness (0-255)
}
```

## Manual Display Configuration

For custom display configurations, create a class derived from `LGFX_Device`:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI       _bus_instance;

public:
    LGFX(void) {
        { // Bus configuration
            auto cfg = _bus_instance.config();
            cfg.spi_host = VSPI_HOST;     // Select SPI peripheral (ESP32)
            cfg.spi_mode = 0;             // Set SPI mode (0-3)
            cfg.freq_write = 40000000;    // Write clock (max 80MHz)
            cfg.freq_read  = 16000000;    // Read clock
            cfg.spi_3wire = false;        // Enable MISO pin
            cfg.use_lock   = true;        // Enable transaction locking
            _bus_instance.config(cfg);    // Apply the configuration
            _panel_instance.setBus(&_bus_instance); // Set bus to panel
        }

        { // Display panel configuration
            auto cfg = _panel_instance.config();
            cfg.pin_cs   = 5;    // CS pin number
            cfg.pin_rst  = 33;   // RST pin number
            cfg.pin_dc   = 27;   // DC pin number
            cfg.memory_width  = 240;  // Maximum width supported by driver IC
            cfg.memory_height = 320;  // Maximum height supported by driver IC
            cfg.panel_width  = 240;   // Actual display width
            cfg.panel_height = 320;   // Actual display height
            cfg.offset_x     = 0;     // Panel offset X
            cfg.offset_y     = 0;     // Panel offset Y
            _panel_instance.config(cfg);
        }
        setPanel(&_panel_instance);
    }
};
```

## Supported Display Controllers

The library supports multiple display controllers including:

- ILI9341
- ST7789
- GC9A01
- SSD1306 (OLED)
- And many others

## Bus Interfaces

Available bus interfaces:

- SPI (Serial Peripheral Interface)
- I2C (Inter-Integrated Circuit)
- 8-bit Parallel
- 16-bit Parallel

## Auto-Detection Support

The auto-detection feature supports common development boards and configurations. Enable it by defining:

```cpp
#define LGFX_AUTODETECT
#include <LGFX_AUTODETECT.hpp>
```

Supported boards include:
- M5Stack
- TTGO T-Display
- ESP-WROVER-KIT
- And many others listed in the examples 