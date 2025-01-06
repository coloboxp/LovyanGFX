# Platform-Specific APIs

## Autodetection System

The library includes an autodetection system (`LGFX_AUTODETECT.hpp`) that automatically configures the display based on the platform and connected hardware.

### Supported Platforms

1. ESP32 Platform
   - Standard ESP32
   - ESP32-S2
   - ESP32-S3
   - ESP32-C3

2. RP2040 (Raspberry Pi Pico)
   - SPI Interface
   - I2C Interface

3. PC Platforms
   - SDL
   - OpenCV
   - Framebuffer
   - WebAssembly

## TFT_eSPI Compatibility Layer

The `LGFX_TFT_eSPI.hpp` provides compatibility with the popular TFT_eSPI library:

```cpp
// Basic compatibility
using TFT_eSPI = LGFX;

// Sprite compatibility
class TFT_eSprite : public LGFX_Sprite {
    TFT_eSprite() : LGFX_Sprite() { _psram = true; }
    TFT_eSprite(LovyanGFX* parent) : LGFX_Sprite(parent) { _psram = true; }
    void* frameBuffer(uint8_t) { return getBuffer(); }
};
```

## User Configuration

The `lgfx_user` directory allows for custom configurations:

- Display panel settings
- Bus configurations (SPI, I2C, parallel)
- Pin assignments
- Timing parameters
- Hardware-specific optimizations 