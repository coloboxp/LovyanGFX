# OLED Panel Implementation Guide

This guide covers the implementation of OLED panels in LovyanGFX, focusing on common OLED controllers like SSD1306, SSD1331, and SH110x.

## OLED Panel Architecture

OLED panels typically use I2C or SPI interfaces and have specific memory management requirements due to their monochrome or limited color nature.

## Common OLED Controllers

### SSD1306 Implementation

The SSD1306 is a popular monochrome OLED controller. Here's a typical configuration:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SSD1306 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions (128x64 typical)
        cfg.memory_width  = 128;
        cfg.memory_height = 64;
        cfg.panel_width   = 128;
        cfg.panel_height  = 64;
        
        // Pin configuration
        cfg.pin_rst = 16;   // Reset pin
        cfg.pin_cs  = -1;   // Not used in I2C mode
        
        // Display options
        cfg.invert  = false;  // Normal display
        cfg.offset_x = 0;
        cfg.offset_y = 0;
        
        // I2C specific settings
        cfg.bus_shared = true;  // Bus shared with other devices
        
        _panel_instance.config(cfg);
    }
};
```

### SSD1331 Implementation

The SSD1331 is a color OLED controller. Configuration example:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SSD1331 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions (96x64 typical)
        cfg.memory_width  = 96;
        cfg.memory_height = 64;
        cfg.panel_width   = 96;
        cfg.panel_height  = 64;
        
        // Pin configuration
        cfg.pin_cs   = 5;    // Chip select
        cfg.pin_rst  = 4;    // Reset
        cfg.pin_dc   = 16;   // Data/Command
        
        // Display options
        cfg.invert   = false;
        cfg.rgb_order = true;  // RGB color order
        cfg.offset_rotation = 0;
        
        _panel_instance.config(cfg);
    }
};
```

### SH110x Implementation

The SH110x series (SH1106, SH1107) are monochrome OLED controllers. Example configuration:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SH110x _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions
        cfg.memory_width  = 128;
        cfg.memory_height = 128;  // SH1107 is 128x128
        cfg.panel_width   = 128;
        cfg.panel_height  = 128;
        
        // Pin configuration
        cfg.pin_rst = 16;
        cfg.pin_cs  = -1;   // Not used in I2C mode
        
        // Display options
        cfg.invert  = false;
        cfg.offset_x = 0;
        cfg.offset_y = 0;
        
        _panel_instance.config(cfg);
    }
};
```

## Advanced Configuration

### Memory Management

OLED displays have specific memory layouts. Configure them properly:

```cpp
// For SSD1306 128x32 display
cfg.memory_width  = 128;  // Memory buffer width
cfg.memory_height = 32;   // Memory buffer height
cfg.panel_width   = 128;  // Visible width
cfg.panel_height  = 32;   // Visible height
cfg.offset_x      = 0;    // No memory offset needed
cfg.offset_y      = 0;    // No memory offset needed

// For SSD1306 128x64 display
cfg.memory_width  = 128;
cfg.memory_height = 64;
```

### Communication Interface

#### I2C Configuration

```cpp
// I2C settings
cfg.i2c_port    = 0;     // I2C port number
cfg.i2c_addr    = 0x3C;  // I2C address (typical for SSD1306)
cfg.pin_sda     = 21;    // I2C SDA pin
cfg.pin_scl     = 22;    // I2C SCL pin
cfg.freq        = 400000; // I2C clock frequency (400kHz)
```

#### SPI Configuration

```cpp
// SPI settings
cfg.spi_mode    = 0;      // SPI mode (0-3)
cfg.freq_write  = 8000000; // SPI clock for write (8MHz)
cfg.freq_read   = 4000000; // SPI clock for read (4MHz)
cfg.pin_dc      = 16;     // Data/Command pin
```

## Platform-Specific Features

### ESP32 Optimizations

```cpp
// I2C optimization
cfg.i2c_port = I2C_NUM_1;  // Use I2C peripheral 1
cfg.freq     = 400000;     // 400kHz Fast Mode

// SPI optimization
cfg.spi_host = VSPI_HOST;  // Use VSPI peripheral
cfg.dma_channel = 1;       // DMA channel for SPI
```

### RP2040 Features

```cpp
// I2C configuration
cfg.i2c_port = i2c0;      // Use I2C0 peripheral
cfg.pin_sda  = 4;         // GPIO4 for SDA
cfg.pin_scl  = 5;         // GPIO5 for SCL

// SPI configuration
cfg.spi_port = spi0;      // Use SPI0 peripheral
cfg.pin_dc   = 8;         // GPIO8 for DC
```

## Error Handling

Implement robust error handling:

```cpp
bool init_panel(void) {
    auto cfg = _panel_instance.config();
    
    // Validate communication interface
    if (cfg.i2c_port < 0 && cfg.pin_cs < 0) {
        return false;    // No valid interface configured
    }
    
    // Configure panel
    _panel_instance.config(cfg);
    
    // Initialize with reset
    if (!_panel_instance.init(true)) {
        return false;    // Initialization failed
    }
    
    // Verify communication
    uint8_t id = _panel_instance.readCommand(0x0F);
    if (id != expected_id) {
        return false;    // Wrong or no device found
    }
    
    return true;
}
```

## Best Practices

1. Interface Selection:
   - Choose I2C for simpler wiring (2 pins)
   - Use SPI for faster refresh rates
   - Consider bus sharing with other devices

2. Power Management:
   - Implement proper power-on sequence
   - Use sleep mode for power saving
   - Handle display contrast/brightness

3. Memory Usage:
   - Understand the display's memory layout
   - Handle page/segment addressing correctly
   - Use appropriate buffer sizes

4. Performance Optimization:
   - Use appropriate communication speeds
   - Implement efficient refresh strategies
   - Consider partial updates when possible

5. Error Recovery:
   - Implement proper reset sequences
   - Handle communication timeouts
   - Provide fallback mechanisms

The OLED panel implementation system provides robust support for various OLED controllers while maintaining consistent behavior across different platforms and communication interfaces.
``` 