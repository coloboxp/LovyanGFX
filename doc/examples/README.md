# LovyanGFX Examples

This directory contains practical examples demonstrating various features and capabilities of the LovyanGFX library. For a comprehensive overview of all examples and their purposes, see the [Examples Overview](overview.md).

## Quick Start Examples

### [Simple Use](HowToUse/1_simple_use)
Basic graphics operations and display initialization.
```cpp
// Basic display initialization
LGFX display;
display.init();
display.setRotation(1);
display.setBrightness(128);

// Basic drawing operations
display.fillScreen(TFT_BLACK);
display.drawLine(0, 0, 240, 320, TFT_WHITE);
display.drawRect(10, 10, 220, 300, TFT_BLUE);
```

### [User Settings](HowToUse/2_user_setting)
Custom display configuration and hardware setup.
```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Bus configuration
        {
            auto cfg = _bus_instance.config();
            cfg.spi_host = VSPI_HOST;
            cfg.freq_write = 40000000;
            cfg.pin_sclk = 18;
            cfg.pin_mosi = 23;
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            cfg.pin_cs = 14;
            cfg.pin_rst = 33;
            cfg.pin_dc = 27;
            _panel_instance.config(cfg);
        }

        setPanel(&_panel_instance);
    }
};
```

### [Font Usage](HowToUse/3_fonts)
Text rendering and font handling.
```cpp
// Font configuration
display.setFont(&fonts::DejaVu18);
display.setTextColor(TFT_WHITE, TFT_BLACK);
display.setTextSize(1);

// Text rendering
display.drawString("Hello, LovyanGFX!", 10, 10);
```

### [Unicode Fonts](HowToUse/4_unicode_fonts)
Multi-language text support and Unicode handling.
```cpp
// Unicode text rendering
display.setFont(&fonts::lgfxJapanGothic_16);
display.drawString("こんにちは", 10, 10);
display.drawString("안녕하세요", 10, 30);
display.drawString("你好", 10, 50);
```

### [Image Handling](HowToUse/5_images)
Image loading and manipulation.
```cpp
// Create sprite for image handling
LGFX_Sprite sprite(&display);
sprite.createSprite(100, 100);
sprite.drawJpgFile(SD, "/image.jpg");
sprite.pushSprite(0, 0);
```

## Advanced Examples

### Graphics and Animation
- [Sprite System](advanced_examples/sprite_system.md)
- [Animation Effects](advanced_examples/animation_effects.md)
- [Complex Graphics](advanced_examples/graphics_text.md)

### User Interface
- [Touch Input](advanced_examples/touch_ui.md)
- [Custom Widgets](advanced_examples/widgets.md)
- [Gesture Recognition](advanced_examples/gestures.md)

### Performance Optimization
- [DMA Operations](advanced_examples/dma_usage.md)
- [Memory Management](advanced_examples/memory_management.md)
- [Double Buffering](advanced_examples/double_buffering.md)

## Platform-Specific Examples

### ESP32 Family
- [Basic ESP32 Setup](platforms/esp32_basic.md)
- [ESP32-S2/S3 Features](platforms/esp32_s2s3.md)
- [ESP32 DMA Usage](platforms/esp32_dma.md)

### Raspberry Pi Pico
- [RP2040 Setup](platforms/rp2040_setup.md)
- [PIO Usage](platforms/rp2040_pio.md)
- [Performance Tips](platforms/rp2040_performance.md)

### SAMD21/SAMD51
- [Basic Setup](platforms/samd_setup.md)
- [Memory Optimization](platforms/samd_memory.md)
- [Display Configuration](platforms/samd_display.md)

## Usage Guidelines

1. Example Selection
   - Start with [Simple Use](HowToUse/1_simple_use) for basic concepts
   - Progress to [User Settings](HowToUse/2_user_setting) for custom setups
   - Explore advanced examples based on your needs

2. Code Adaptation
   - Adjust pin configurations for your hardware
   - Modify timing parameters as needed
   - Scale dimensions for your display
   - Test performance with your setup

3. Performance Considerations
   - Choose appropriate color depth
   - Enable hardware acceleration when available
   - Implement double buffering for smooth graphics
   - Monitor memory usage

4. Error Handling
   - Check initialization success
   - Validate parameters
   - Handle touch errors gracefully
   - Monitor memory allocation
   - Implement proper cleanup

## Additional Resources

- [Examples Overview](overview.md) - Detailed guide to all examples
- [Troubleshooting Guide](../troubleshooting.md)
- [Performance Guide](../performance_guide.md)
- [API Reference](../api_reference.md) 