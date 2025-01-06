# ESP32 Platform Guide

This guide covers the ESP32 family implementation in LovyanGFX, including ESP32, ESP32-S2, ESP32-S3, and ESP32-C3.

## Hardware Interfaces

### Display Controllers

1. **ESP32-S3 LCD_CAM Controller**
```
LCD_CAM Architecture:
┌─────────────────────┐
│    LCD_CAM Block    │
├─────────┬───────────┤
│DMA Ctrl │LCD Control│
├─────────┴───────────┤
│  Data Path Control  │
└─────────────────────┘
```

- Features:
  - Native DMA support
  - Hardware acceleration
  - RGB interface support
  - Parallel 8/16-bit support

2. **ESP32/ESP32-S2 I2S Controller**
```
I2S as Parallel Interface:
┌─────────────────────┐
│     I2S Block       │
├─────────┬───────────┤
│DMA Ctrl │Parallel IF│
├─────────┴───────────┤
│    Clock Control    │
└─────────────────────┘
```

- Features:
  - DMA support via I2S
  - 8-bit parallel support
  - Limited to 40MHz

## Bus Configurations

### 1. Parallel 8-bit
```cpp
// Example configuration for 8-bit parallel
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_Parallel8 _bus_instance;

    void init() {
        auto cfg = _bus_instance.config();
        cfg.pin_wr = 47;   // Write strobe
        cfg.pin_rd = 48;   // Read strobe
        cfg.pin_rs = 0;    // Register/Data select
        cfg.pin_d0 = 9;    // Data pin 0
        cfg.pin_d1 = 10;   // Data pin 1
        cfg.pin_d2 = 11;   // Data pin 2
        cfg.pin_d3 = 12;   // Data pin 3
        cfg.pin_d4 = 13;   // Data pin 4
        cfg.pin_d5 = 14;   // Data pin 5
        cfg.pin_d6 = 15;   // Data pin 6
        cfg.pin_d7 = 16;   // Data pin 7
        
        // ESP32-S3 specific
        cfg.freq_write = 80000000;  // 80MHz max
        cfg.freq_read  = 16000000;  // 16MHz max
        
        _bus_instance.config(cfg);
        _panel_instance.setBus(&_bus_instance);
    }
};
```

### 2. Parallel 16-bit (ESP32-S3)
```cpp
// Example configuration for 16-bit parallel
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_Parallel16 _bus_instance;

    void init() {
        auto cfg = _bus_instance.config();
        // Control pins
        cfg.pin_wr = 47;
        cfg.pin_rd = 48;
        cfg.pin_rs = 0;
        
        // Data pins 0-15
        cfg.pin_d0  = 1;   // LSB
        // ... pins 1-14 ...
        cfg.pin_d15 = 16;  // MSB
        
        cfg.freq_write = 40000000;  // 40MHz max
        cfg.freq_read  = 16000000;  // 16MHz max
        
        _bus_instance.config(cfg);
        _panel_instance.setBus(&_bus_instance);
    }
};
```

### 3. RGB Interface (ESP32-S3)
```cpp
// Example configuration for RGB interface
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ST7701 _panel_instance;  // Common RGB panel
    lgfx::Bus_RGB _bus_instance;

    void init() {
        auto cfg = _bus_instance.config();
        // RGB signal pins
        cfg.pin_r0 = 1;    // Red LSB
        cfg.pin_r4 = 5;    // Red MSB
        cfg.pin_g0 = 6;    // Green LSB
        cfg.pin_g5 = 11;   // Green MSB
        cfg.pin_b0 = 12;   // Blue LSB
        cfg.pin_b4 = 16;   // Blue MSB
        
        // Control signals
        cfg.pin_hsync = 17;
        cfg.pin_vsync = 18;
        cfg.pin_de    = 19;
        cfg.pin_pclk  = 20;
        
        cfg.freq_write = 16000000;  // 16MHz typical
        
        _bus_instance.config(cfg);
        _panel_instance.setBus(&_bus_instance);
    }
};
```

## Performance Optimization

### 1. DMA Configuration
```cpp
// Optimal DMA settings
auto cfg = _panel_instance.config();
cfg.dma_channel = 1;      // DMA channel (ESP32-S3: 1-4)
cfg.dma_buffer_size = 2048; // Buffer size in bytes
cfg.dma_desc_count = 4;    // Number of DMA descriptors
_panel_instance.config(cfg);
```

### 2. Clock Settings
```
Recommended Clock Frequencies:
┌────────────┬───────────┬──────────┐
│ Interface  │ ESP32-S3  │  ESP32   │
├────────────┼───────────┼──────────┤
│ 8-bit     │   80MHz   │   40MHz  │
│ 16-bit    │   40MHz   │    N/A   │
│ RGB       │   16MHz   │    N/A   │
│ SPI       │   80MHz   │   80MHz  │
└────────────┴───────────┴──────────┘
```

### 3. Memory Usage
```cpp
// PSRAM configuration
auto cfg = _panel_instance.config();
cfg.use_psram = true;     // Enable PSRAM
cfg.psram_freq = 80;      // PSRAM frequency in MHz
_panel_instance.config(cfg);
```

## Troubleshooting

### 1. Display Issues
```cpp
// Common problems and solutions
if (!display.init()) {
    // Check pin assignments
    log_e("Pin assignment error");
    // Verify power supply
    log_e("Check VCC and GND");
    // Check clock settings
    log_e("Verify clock frequency");
}
```

### 2. Performance Issues
```cpp
// Performance debugging
auto cfg = _bus_instance.config();
// Check clock frequency
if (cfg.freq_write > 80000000) {
    log_w("Clock frequency too high");
}
// Verify DMA settings
if (_panel_instance.config().dma_channel == 0) {
    log_w("DMA not configured");
}
```

## External Resources

### Documentation
- [ESP32-S3 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf)
- [ESP32 LCD Overview](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/lcd.html)
- [ESP32 DMA Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/gdma.html)

### Tools
- [ESP32 Pin Layout Tool](https://esp32.com/pin_layout.html)
- [ESP32 Clock Calculator](https://esp32.com/clock_calculator.html)

### Example Projects
- [Basic Graphics Demo](../examples/HowToUse/README.md)
- [Advanced RGB Panel](../examples/Advanced/README.md)
- [Performance Tests](../examples/Test/README.md) 