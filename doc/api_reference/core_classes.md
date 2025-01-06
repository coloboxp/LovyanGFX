# Core Classes Documentation

## LovyanGFX Class

The main class that provides the core graphics functionality. This class serves as the primary interface for all graphics operations.

### Key Components

- Graphics primitives
- Display control (see [Configuration Options](configuration.md#display-configuration))
- Memory management (see [Hardware Configuration](configuration.md#hardware-configuration))
- Hardware acceleration support (see [Core Concepts: Hardware Acceleration](../core_concepts/hardware_acceleration.md))

## LGFX_Sprite Class

A sprite handling class that enables efficient sprite operations with optional PSRAM support.

### Features

- Buffer management
- Sprite operations (see [Examples: Sprite Operations](../examples/Sprite/README.md))
- Memory allocation control
- Parent-child relationship with main display

## TFT_eSPI Compatibility

LovyanGFX provides compatibility with TFT_eSPI through wrapper classes. For platform-specific details, see [Platform-Specific APIs](platform_specific.md#tft_espi-compatibility-layer).

```cpp
using TFT_eSPI = LGFX;  // Main display class compatibility

class TFT_eSprite : public LGFX_Sprite {
    // Sprite compatibility with automatic PSRAM support
};
```

## Color Definitions

The library provides standard color definitions in both ILI9341 format and generic format. For more details on color configuration, see [Configuration Options](configuration.md#color-settings).

```cpp
namespace lgfx::ili9341_colors {
    static constexpr int BLACK       = 0x0000;  //   0,   0,   0
    static constexpr int NAVY        = 0x000F;  //   0,   0, 128
    static constexpr int DARKGREEN   = 0x03E0;  //   0, 128,   0
    // ... more colors available
}
```

## Display Commands

Standard display control commands are defined in the `tft_command` namespace. For event handling related to these commands, see [Event System](event_system.md#display-events).

```cpp
namespace lgfx::tft_command {
    static constexpr int TFT_DISPOFF = 0x28;  // Display OFF
    static constexpr int TFT_DISPON  = 0x29;  // Display ON
    static constexpr int TFT_SLPIN   = 0x10;  // Sleep mode ON
    static constexpr int TFT_SLPOUT  = 0x11;  // Sleep mode OFF
} 