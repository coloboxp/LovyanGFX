# User Settings Example

This example demonstrates how to create custom display configurations in LovyanGFX, including bus setup, panel configuration, and peripheral management.

## Overview

The User Settings example (`examples/HowToUse/2_user_setting/2_user_setting.ino`) shows how to:
- Create a custom display configuration class
- Configure different bus types (SPI, I2C, Parallel)
- Set up display panels with specific parameters
- Configure touch and backlight functionality
- Handle hardware-specific settings

## Custom Configuration Class

Create a custom class derived from `LGFX_Device`:
```cpp
class LGFX : public lgfx::LGFX_Device {
private:
    // Panel instance
    lgfx::Panel_ILI9341 _panel_instance;

    // Bus instance
    lgfx::Bus_SPI _bus_instance;

    // Optional components
    lgfx::Light_PWM _light_instance;
    lgfx::Touch_FT5x06 _touch_instance;

public:
    LGFX(void) {
        // Configuration in constructor
    }
};
```

## Bus Configuration

### SPI Bus Setup
```cpp
auto cfg = _bus_instance.config();

// Basic SPI settings
cfg.spi_host = VSPI_HOST;     // SPI host (VSPI_HOST or HSPI_HOST)
cfg.spi_mode = 0;             // SPI mode (0-3)
cfg.freq_write = 40000000;    // Write clock frequency (max 80MHz)
cfg.freq_read = 16000000;     // Read clock frequency

// SPI pins
cfg.pin_sclk = 18;            // SCLK pin
cfg.pin_mosi = 23;            // MOSI pin
cfg.pin_miso = 19;            // MISO pin
cfg.pin_dc = 27;              // Data/Command pin

// Advanced settings
cfg.spi_3wire = true;         // Use MOSI for both send/receive
cfg.use_lock = true;          // Enable transaction locking
cfg.dma_channel = SPI_DMA_CH_AUTO; // DMA channel
```

### I2C Bus Setup
```cpp
auto cfg = _bus_instance.config();

cfg.i2c_port = 0;          // I2C port (0 or 1)
cfg.freq_write = 400000;   // Write clock frequency
cfg.freq_read = 400000;    // Read clock frequency
cfg.pin_sda = 21;          // SDA pin
cfg.pin_scl = 22;          // SCL pin
cfg.i2c_addr = 0x3C;       // I2C device address
```

### Parallel Bus Setup
```cpp
auto cfg = _bus_instance.config();

cfg.i2s_port = I2S_NUM_0;  // I2S port
cfg.freq_write = 20000000; // Write clock (max 20MHz)
cfg.pin_wr = 4;           // Write pin
cfg.pin_rd = 2;           // Read pin
cfg.pin_rs = 15;          // RS pin
cfg.pin_d0 = 12;          // Data pins D0-D7
cfg.pin_d1 = 13;
// ... D2-D7 pins
```

## Panel Configuration

```cpp
auto cfg = _panel_instance.config();

// Basic settings
cfg.pin_cs = 14;           // CS pin
cfg.pin_rst = 33;          // Reset pin
cfg.pin_busy = -1;         // Busy pin

// Display parameters
cfg.panel_width = 240;     // Display width
cfg.panel_height = 320;    // Display height
cfg.offset_x = 0;          // X offset
cfg.offset_y = 0;          // Y offset

// Advanced settings
cfg.offset_rotation = 0;   // Rotation offset (0-7)
cfg.readable = true;       // Enable reading
cfg.invert = false;        // Invert colors
cfg.rgb_order = false;     // Swap RGB order
cfg.dlen_16bit = false;    // 16-bit data length
cfg.bus_shared = true;     // Shared bus (with SD card)
```

## Backlight Configuration

```cpp
auto cfg = _light_instance.config();

cfg.pin_bl = 32;          // Backlight pin
cfg.invert = false;       // Invert brightness
cfg.freq = 44100;         // PWM frequency
cfg.pwm_channel = 7;      // PWM channel

_light_instance.config(cfg);
_panel_instance.setLight(&_light_instance);
```

## Touch Configuration

```cpp
auto cfg = _touch_instance.config();

// Touch area
cfg.x_min = 0;
cfg.x_max = 239;
cfg.y_min = 0;
cfg.y_max = 319;

// Hardware settings
cfg.pin_int = 38;        // Interrupt pin
cfg.bus_shared = true;   // Shared bus
cfg.offset_rotation = 0; // Rotation offset

// SPI settings
cfg.spi_host = VSPI_HOST;
cfg.freq = 1000000;
cfg.pin_sclk = 18;
cfg.pin_mosi = 23;
cfg.pin_miso = 19;
cfg.pin_cs = 5;

_touch_instance.config(cfg);
_panel_instance.setTouch(&_touch_instance);
```

## Initialization and Usage

```cpp
LGFX display;

void setup() {
    display.init();

    // Optional touch calibration
    if (display.touch()) {
        display.calibrateTouch(nullptr, TFT_WHITE, TFT_BLACK,
            std::max(display.width(), display.height()) >> 3);
    }
}
```

## Best Practices

1. Bus Configuration
   - Choose appropriate bus type for your hardware
   - Set correct pin assignments
   - Configure optimal clock frequencies
   - Enable DMA for better performance

2. Panel Settings
   - Verify panel dimensions
   - Set correct rotation and offset
   - Configure color settings
   - Enable bus sharing if needed

3. Touch Setup
   - Calibrate touch on startup
   - Handle rotation offsets
   - Configure interrupt handling
   - Set appropriate touch area

4. Memory Management
   - Use appropriate color depth
   - Enable DMA when available
   - Share bus resources efficiently

## Common Issues

1. Display Problems
   - Check pin assignments
   - Verify bus configuration
   - Confirm panel parameters
   - Test different rotation settings

2. Touch Issues
   - Run calibration
   - Check touch area settings
   - Verify interrupt handling
   - Test bus sharing configuration

3. Performance
   - Optimize bus frequencies
   - Enable DMA where possible
   - Use appropriate color depth
   - Share bus resources efficiently

## Additional Resources

- [Bus Configuration Guide](../core_concepts/bus_config.md)
- [Panel Setup Guide](../core_concepts/panel_setup.md)
- [Touch Configuration](../core_concepts/touch_config.md)
- [Hardware Compatibility](../hardware_compatibility.md) 