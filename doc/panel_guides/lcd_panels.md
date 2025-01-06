# LCD Panel Implementation Guide

This guide covers the implementation of LCD panels in LovyanGFX, focusing on common LCD controllers like ILI9341, ST7789, and GC9A01.

## LCD Panel Architecture

LCD panels inherit from the `Panel_LCD` class, which provides common LCD-specific functionality:

```cpp
class Panel_LCD : public Panel_Device {
public:
    bool init(bool use_reset);
    void beginTransaction(void);
    void endTransaction(void);
    void setRotation(uint_fast8_t r);
    void setInvert(bool invert);
    void setSleep(bool flg);
    void setPowerSave(bool flg);
};
```

## Common LCD Controllers

### ILI9341 Implementation

The ILI9341 is a popular 240x320 TFT LCD controller. Here's a typical configuration:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions
        cfg.memory_width  = 240;
        cfg.memory_height = 320;
        cfg.panel_width   = 240;
        cfg.panel_height  = 320;
        
        // Pin configuration
        cfg.pin_cs   = 5;     // Chip select
        cfg.pin_rst  = 33;    // Reset
        cfg.pin_busy = -1;    // No busy pin
        
        // Display options
        cfg.readable     = true;   // Enable read operations
        cfg.invert      = false;   // Normal display
        cfg.rgb_order   = false;   // RGB color order
        cfg.dlen_16bit  = false;   // 8-bit data length
        
        // Performance settings
        cfg.dummy_read_pixel = 8;
        cfg.dummy_read_bits  = 1;
        
        _panel_instance.config(cfg);
    }
};
```

### ST7789 Implementation

The ST7789 is commonly used in 240x240 displays. Configuration example:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ST7789 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions
        cfg.memory_width  = 240;
        cfg.memory_height = 320;
        cfg.panel_width   = 240;
        cfg.panel_height  = 240;   // Common 1.3" display
        
        // Memory layout
        cfg.offset_x     = 0;
        cfg.offset_y     = 80;    // Y offset for 240x240 on 240x320
        
        // Display options
        cfg.invert      = true;   // Typically inverted
        cfg.rgb_order   = true;   // RGB color order
        cfg.readable    = true;   // Enable read operations
        
        _panel_instance.config(cfg);
    }
};
```

### GC9A01 Implementation

The GC9A01 is used in round displays. Configuration example:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_GC9A01 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions (240x240 round)
        cfg.memory_width  = 240;
        cfg.memory_height = 240;
        cfg.panel_width   = 240;
        cfg.panel_height  = 240;
        
        // Display options
        cfg.invert      = true;    // Typically inverted
        cfg.rgb_order   = true;    // RGB color order
        cfg.readable    = false;   // Read operations not supported
        
        _panel_instance.config(cfg);
    }
};
```

## Advanced Configuration

### Memory Management

LCD panels often have a memory buffer larger than the visible area. Configure this using:

```cpp
cfg.memory_width   = 240;  // Full frame buffer width
cfg.memory_height  = 320;  // Full frame buffer height
cfg.panel_width    = 240;  // Visible display width
cfg.panel_height   = 240;  // Visible display height
cfg.offset_x       = 0;    // X offset in memory
cfg.offset_y       = 40;   // Y offset in memory
```

### Color Configuration

Control color handling with these settings:

```cpp
cfg.rgb_order   = false;  // false = RGB, true = BGR
cfg.invert      = false;  // false = normal, true = inverted
cfg.dlen_16bit  = false;  // false = 8-bit, true = 16-bit data length
```

### Performance Optimization

Optimize read operations with proper dummy cycle configuration:

```cpp
cfg.dummy_read_pixel = 8;  // Dummy cycles for pixel reads
cfg.dummy_read_bits  = 1;  // Dummy cycles for command reads
```

## Platform-Specific Features

### ESP32 Optimizations

ESP32 platforms support additional features:

```cpp
// SPI optimization
cfg.spi_3wire  = true;    // Enable 3-wire SPI mode
cfg.freq_write = 40000000; // 40MHz write clock
cfg.freq_read  = 16000000; // 16MHz read clock

// DMA support
cfg.dma_channel = 1;      // DMA channel selection
cfg.use_dma     = true;   // Enable DMA transfers
```

### RP2040 Features

Raspberry Pi Pico specific settings:

```cpp
// PIO configuration
cfg.pio_num   = 0;        // PIO block selection
cfg.sm_num    = 0;        // State machine number
cfg.pin_dc    = 8;        // Data/Command pin
```

## Error Handling

Implement proper error handling in your panel configuration:

```cpp
bool init_panel(void) {
    auto cfg = _panel_instance.config();
    
    // Validate pin configuration
    if (cfg.pin_cs < 0 || cfg.pin_rst < 0) {
        return false;    // Required pins not configured
    }
    
    // Configure panel
    _panel_instance.config(cfg);
    
    // Initialize panel
    if (!_panel_instance.init(true)) {
        return false;    // Initialization failed
    }
    
    return true;
}
```

## Best Practices

1. Pin Selection:
   - Use hardware-specific pins when available
   - Implement proper pin initialization
   - Consider signal integrity for high-speed operation

2. Memory Configuration:
   - Set correct memory dimensions
   - Configure proper offsets for partial displays
   - Handle rotation offsets correctly

3. Performance Optimization:
   - Use appropriate clock speeds
   - Enable DMA when available
   - Configure proper dummy cycles

4. Error Handling:
   - Validate pin configurations
   - Check initialization results
   - Implement timeout mechanisms

The LCD panel implementation provides a flexible foundation for supporting various display controllers while maintaining consistent behavior across different platforms.
``` 