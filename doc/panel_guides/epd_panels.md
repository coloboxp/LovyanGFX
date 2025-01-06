# E-Paper (EPD) Panel Implementation Guide

This guide covers the implementation of E-Paper (EPD) panels in LovyanGFX, focusing on common EPD controllers like IT8951, GDEW, and other e-ink displays.

## EPD Panel Architecture

E-Paper displays have unique characteristics:
- Slow refresh rates but very low power consumption
- Multiple display modes (full refresh, partial refresh)
- Typically monochrome or grayscale
- Persistent display without power

## Common EPD Controllers

### IT8951 Implementation

The IT8951 is a versatile EPD controller supporting various panel sizes. Here's a typical configuration:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_IT8951 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions (for 6-inch 800x600)
        cfg.memory_width  = 800;
        cfg.memory_height = 600;
        cfg.panel_width   = 800;
        cfg.panel_height  = 600;
        
        // Pin configuration
        cfg.pin_cs   = 5;    // Chip select
        cfg.pin_rst  = 16;   // Reset
        cfg.pin_busy = 4;    // Busy signal
        
        // Display options
        cfg.offset_rotation = 0;
        cfg.epd_mode = epd_mode_t::epd_quality;  // Default to quality mode
        
        _panel_instance.config(cfg);
    }
};
```

### GDEW Implementation

GDEW displays are common in smaller e-ink applications. Configuration example:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_GDEW0154M09 _panel_instance;  // 1.54" 200x200

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // Display dimensions
        cfg.memory_width  = 200;
        cfg.memory_height = 200;
        cfg.panel_width   = 200;
        cfg.panel_height  = 200;
        
        // Pin configuration
        cfg.pin_cs   = 5;    // Chip select
        cfg.pin_rst  = 16;   // Reset
        cfg.pin_busy = 4;    // Busy signal
        cfg.pin_dc   = 17;   // Data/Command
        
        // EPD specific settings
        cfg.busy_active = true;  // Busy signal is active high
        cfg.epd_mode = epd_mode_t::epd_fast;  // Default to fast mode
        
        _panel_instance.config(cfg);
    }
};
```

## Advanced Configuration

### Display Modes

EPD panels support different refresh modes:

```cpp
// Quality mode (full refresh)
cfg.epd_mode = epd_mode_t::epd_quality;  // Best image quality
cfg.refresh_rate = 4;                     // Slower refresh

// Fast mode (partial refresh)
cfg.epd_mode = epd_mode_t::epd_fast;     // Faster updates
cfg.refresh_rate = 1;                     // Quicker refresh

// Text mode (optimized for text)
cfg.epd_mode = epd_mode_t::epd_text;     // Clear text rendering
```

### Memory Management

EPD displays often require specific memory handling:

```cpp
// Buffer configuration
cfg.memory_width   = 800;   // Full panel width
cfg.memory_height  = 600;   // Full panel height
cfg.offset_x       = 0;     // No offset needed
cfg.offset_y       = 0;     // No offset needed

// Color depth settings
cfg.write_depth.bits = 4;   // 4-bit grayscale
cfg.write_depth.has_color = false;  // Monochrome display
```

### Communication Interface

#### SPI Configuration

```cpp
// SPI settings for EPD
cfg.spi_mode    = 0;       // SPI mode (usually 0)
cfg.freq_write  = 4000000; // 4MHz SPI clock (conservative)
cfg.freq_read   = 4000000; // 4MHz read clock
cfg.pin_dc      = 17;      // Data/Command pin
```

## Platform-Specific Features

### ESP32 Optimizations

```cpp
// SPI optimization
cfg.spi_host = VSPI_HOST;   // Use VSPI peripheral
cfg.dma_channel = 1;        // DMA channel for SPI
cfg.freq_write = 20000000;  // 20MHz on capable hardware

// Power management
cfg.power_pin = 32;         // Power control pin
cfg.power_on_level = true;  // Active high power control
```

### RP2040 Features

```cpp
// SPI configuration
cfg.spi_port = spi0;       // Use SPI0 peripheral
cfg.pin_dc   = 8;          // GPIO8 for DC
cfg.pio_num  = 0;          // PIO block for custom timing
```

## Error Handling

Implement robust error handling for EPD panels:

```cpp
bool init_panel(void) {
    auto cfg = _panel_instance.config();
    
    // Validate pin configuration
    if (cfg.pin_cs < 0 || cfg.pin_busy < 0) {
        return false;    // Required pins not configured
    }
    
    // Configure panel
    _panel_instance.config(cfg);
    
    // Initialize with power cycle
    digitalWrite(cfg.power_pin, cfg.power_on_level);
    delay(100);  // Power stabilization
    
    if (!_panel_instance.init(true)) {
        digitalWrite(cfg.power_pin, !cfg.power_on_level);
        return false;    // Initialization failed
    }
    
    // Verify panel is ready
    if (!wait_busy(5000)) {  // 5 second timeout
        return false;    // Panel not responding
    }
    
    return true;
}

bool wait_busy(uint32_t timeout_ms) {
    uint32_t start = millis();
    while (digitalRead(_cfg.pin_busy) == _cfg.busy_active) {
        if (millis() - start > timeout_ms) {
            return false;
        }
        delay(1);
    }
    return true;
}
```

## Best Practices

1. Power Management:
   - Implement proper power sequencing
   - Use sleep mode between updates
   - Handle deep sleep for long-term storage

2. Display Updates:
   - Choose appropriate refresh mode
   - Minimize full refreshes
   - Buffer updates when possible

3. Memory Usage:
   - Understand frame buffer requirements
   - Handle partial updates efficiently
   - Consider RAM limitations

4. Performance Optimization:
   - Use appropriate refresh modes
   - Implement efficient update strategies
   - Balance quality vs. speed

5. Error Recovery:
   - Handle busy timeouts
   - Implement power cycling
   - Provide visual feedback

6. Temperature Considerations:
   - Monitor operating temperature
   - Adjust timings if needed
   - Handle temperature-related artifacts

The EPD panel implementation system provides robust support for various e-paper displays while handling their unique characteristics and requirements. 