# Configuration Options

## Display Configuration

The library supports configuration through the `lgfx_user` directory. For platform-specific details, see [Platform-Specific APIs](platform_specific.md#user-configuration).

### Panel Configuration
```cpp
struct Panel_Device {
    bus_cfg;        // Bus configuration
    panel_cfg;      // Panel-specific settings
    pin_cfg;        // Pin assignments
    dma_cfg;        // DMA settings (see Core Concepts: DMA Operations)
};
```

### Bus Settings
- SPI Configuration (see [Examples: SPI Implementation](../examples_for_picosdk/spi/README.md))
- I2C Configuration (see [Examples: I2C Implementation](../examples_for_picosdk/i2c/README.md))
- Parallel Interface
- RGB Interface

### Color Settings
```cpp
namespace lgfx::ili9341_colors {
    // Standard color definitions
    BLACK       = 0x0000
    NAVY        = 0x000F
    DARKGREEN   = 0x03E0
    // ... more colors
}
```

### Display Commands
```cpp
namespace lgfx::tft_command {
    TFT_DISPOFF = 0x28  // Display OFF
    TFT_DISPON  = 0x29  // Display ON
    TFT_SLPIN   = 0x10  // Sleep mode ON
    TFT_SLPOUT  = 0x11  // Sleep mode OFF
}
```

## Hardware Configuration

### Platform-Specific Settings
- ESP32 Configuration (see [ESP32 Platform](../platforms/esp32.md))
- RP2040 Settings (see [RP2040 Platform](../platforms/rp2040.md))
- PC Platform Options (see [Framebuffer](../platforms/framebuffer.md))

### Memory Management
- PSRAM Usage (see [Core Classes: LGFX_Sprite](core_classes.md#lgfx_sprite-class))
- DMA Configuration (see [Core Concepts: DMA Operations](../core_concepts/dma_operations.md))
- Buffer Management

## Performance Options
- Hardware Acceleration (see [Core Concepts: Hardware Acceleration](../core_concepts/hardware_acceleration.md))
- Color Depth Settings
- Refresh Rate Control
- DMA Transfer Settings 