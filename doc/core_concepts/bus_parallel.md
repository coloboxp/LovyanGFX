# Parallel Bus Interfaces

LovyanGFX provides comprehensive parallel bus implementations including 8-bit, 16-bit, and RGB parallel interfaces for high-speed display communication.

## Architecture Overview

The parallel bus implementations consist of three main classes:
- `Bus_Parallel8`: 8-bit parallel interface supporting up to 20MHz transfer rates
- `Bus_Parallel16`: 16-bit parallel interface supporting up to 40MHz transfer rates
- `Bus_RGB`: RGB parallel interface supporting high-resolution displays

## ESP32 Platform Support

### ESP32-S2/S3 Features
- Hardware-accelerated parallel interface using LCD_CAM peripheral
- DMA support for efficient data transfer
- RGB interface support (ESP32-S3)
- Support for all parallel modes (8-bit, 16-bit, RGB)

### ESP32 Original Features
- I2S peripheral for parallel output
- DMA support through I2S peripheral
- 8-bit parallel mode support

## RGB Parallel Bus Configuration

The RGB parallel bus is ideal for high-resolution displays. Here's a real-world example:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_RGB _bus_instance;
    lgfx::Panel_RGB _panel_instance;
    lgfx::Light_PWM _light_instance;

    void init_display(void) {
        // Panel config
        {
            auto cfg = _panel_instance.config();
            cfg.memory_width = 1024;
            cfg.memory_height = 600;
            cfg.panel_width = 1024;
            cfg.panel_height = 600;
            cfg.offset_x = 0;
            cfg.offset_y = 0;
            _panel_instance.config(cfg);
        }

        // Enable PSRAM if available
        {
            auto cfg = _panel_instance.config_detail();
            cfg.use_psram = true;
            _panel_instance.config_detail(cfg);
        }

        // Bus config
        {
            auto cfg = _bus_instance.config();
            cfg.panel = &_panel_instance;
            
            // RGB Data pins
            cfg.pin_d0  = GPIO_NUM_8;   // B0
            cfg.pin_d1  = GPIO_NUM_3;   // B1
            cfg.pin_d2  = GPIO_NUM_46;  // B2
            cfg.pin_d3  = GPIO_NUM_9;   // B3
            cfg.pin_d4  = GPIO_NUM_1;   // B4
            cfg.pin_d5  = GPIO_NUM_5;   // G0
            cfg.pin_d6  = GPIO_NUM_6;   // G1
            cfg.pin_d7  = GPIO_NUM_7;   // G2
            cfg.pin_d8  = GPIO_NUM_15;  // G3
            cfg.pin_d9  = GPIO_NUM_16;  // G4
            cfg.pin_d10 = GPIO_NUM_4;   // G5
            cfg.pin_d11 = GPIO_NUM_45;  // R0
            cfg.pin_d12 = GPIO_NUM_48;  // R1
            cfg.pin_d13 = GPIO_NUM_47;  // R2
            cfg.pin_d14 = GPIO_NUM_21;  // R3
            cfg.pin_d15 = GPIO_NUM_14;  // R4

            // Control signals
            cfg.pin_henable = GPIO_NUM_40;  // HENABLE
            cfg.pin_vsync = GPIO_NUM_41;    // VSYNC
            cfg.pin_hsync = GPIO_NUM_39;    // HSYNC
            cfg.pin_pclk = GPIO_NUM_42;     // PCLK
            
            // Clock settings
            cfg.freq_write = 16000000;  // Pixel clock frequency

            // Timing parameters
            cfg.hsync_polarity = 0;
            cfg.hsync_front_porch = 80;
            cfg.hsync_pulse_width = 4;
            cfg.hsync_back_porch = 16;
            cfg.vsync_polarity = 0;
            cfg.vsync_front_porch = 22;
            cfg.vsync_pulse_width = 4;
            cfg.vsync_back_porch = 4;
            cfg.pclk_idle_high = 1;

            _bus_instance.config(cfg);
        }

        _panel_instance.setBus(&_bus_instance);

        // Backlight configuration
        {
            auto cfg = _light_instance.config();
            cfg.pin_bl = GPIO_NUM_10;
            cfg.invert = true;
            _light_instance.config(cfg);
        }
        _panel_instance.light(&_light_instance);
    }
};
```

## 8-bit Parallel Bus Configuration

The 8-bit parallel bus uses 8 data lines plus control signals. Here's a typical configuration:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_Parallel8 _bus_instance;

    void init_bus(void) {
        auto cfg = _bus_instance.config();
        
        // Data pins (D0-D7)
        cfg.pin_d0 = 0;
        cfg.pin_d1 = 1;
        cfg.pin_d2 = 2;
        cfg.pin_d3 = 3;
        cfg.pin_d4 = 4;
        cfg.pin_d5 = 5;
        cfg.pin_d6 = 6;
        cfg.pin_d7 = 7;
        
        // Control pins
        cfg.pin_wr = 8;    // Write strobe
        cfg.pin_rd = 9;    // Read strobe
        cfg.pin_rs = 10;   // Register select (D/C)
        
        // Communication settings
        cfg.freq_write = 16000000;  // 16MHz write clock
        
        _bus_instance.config(cfg);
    }
};
```

### Platform-Specific Features (8-bit)

ESP32/ESP32-S2:
```cpp
// Uses I2S peripheral for parallel output
cfg.i2s_port = 0;  // I2S peripheral number (0 or 1)
// Available frequencies: 40MHz, 20MHz, 16MHz, 13.3MHz, 11.43MHz, 10MHz, 8.9MHz
cfg.freq_write = 16000000;  // 16MHz default
```

ESP32-S3:
```cpp
// Uses LCD_CAM peripheral
cfg.port = 0;      // LCD_CAM peripheral (only 0 available)
// Supports up to 80MHz clock
cfg.freq_write = 16000000;  // 16MHz default
cfg.freq_read = 8000000;    // 8MHz read clock
```

## 16-bit Parallel Bus Configuration

The 16-bit parallel bus uses 16 data lines plus control signals for higher bandwidth data transfer:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_Parallel16 _bus_instance;

    void init_bus(void) {
        auto cfg = _bus_instance.config();
        
        // Lower byte (D0-D7)
        cfg.pin_d0 = 0;
        cfg.pin_d1 = 1;
        cfg.pin_d2 = 2;
        cfg.pin_d3 = 3;
        cfg.pin_d4 = 4;
        cfg.pin_d5 = 5;
        cfg.pin_d6 = 6;
        cfg.pin_d7 = 7;
        
        // Upper byte (D8-D15)
        cfg.pin_d8 = 8;
        cfg.pin_d9 = 9;
        cfg.pin_d10 = 10;
        cfg.pin_d11 = 11;
        cfg.pin_d12 = 12;
        cfg.pin_d13 = 13;
        cfg.pin_d14 = 14;
        cfg.pin_d15 = 15;
        
        // Control pins
        cfg.pin_wr = 16;   // Write strobe
        cfg.pin_rd = 17;   // Read strobe
        cfg.pin_rs = 18;   // Register select (D/C)
        
        // Communication settings
        cfg.freq_write = 16000000;  // 16MHz write clock
        cfg.freq_read = 8000000;    // 8MHz read clock
        
        _bus_instance.config(cfg);
    }
};
```

### Platform-Specific Features (16-bit)

ESP32-S3:
```cpp
// Uses LCD_CAM peripheral
cfg.port = 0;      // LCD_CAM peripheral (only 0 available)
// Supports up to 40MHz clock in 16-bit mode
cfg.freq_write = 16000000;  // 16MHz default
cfg.freq_read = 8000000;    // 8MHz read clock
```

## DMA Support

Both 8-bit and 16-bit modes support DMA for efficient data transfer:

```cpp
// DMA configuration
cfg.dma_channel = 1;    // DMA channel (1 or 2)
cfg.dma_buffer_size = 1024;  // DMA buffer size
```

## Performance Optimization Tips

1. Clock Frequency Selection:
   - ESP32/ESP32-S2: Up to 40MHz in 8-bit mode
   - ESP32-S3: 
     - Up to 80MHz in 8-bit mode
     - Up to 40MHz in 16-bit mode
     - Up to 16MHz in RGB mode (depends on display)
   - Always validate with oscilloscope for signal integrity

2. Pin Selection:
   - Use adjacent GPIO pins when possible
   - Keep data lines on same GPIO port for better performance
   - Consider signal routing on PCB:
     ```cpp
     // Example of optimal pin grouping
     cfg.pin_d0  = GPIO_NUM_8;   // Group B pins together
     cfg.pin_d1  = GPIO_NUM_3;
     cfg.pin_d2  = GPIO_NUM_46;
     // ... Group G pins together
     cfg.pin_d5  = GPIO_NUM_5;
     cfg.pin_d6  = GPIO_NUM_6;
     cfg.pin_d7  = GPIO_NUM_7;
     ```

3. Memory Management:
   - Enable PSRAM for large frame buffers
   - Use DMA for efficient transfers
   - Align buffers to 32-bit boundaries
   ```cpp
   auto cfg = _panel_instance.config_detail();
   cfg.use_psram = true;  // Enable PSRAM for frame buffer
   _panel_instance.config_detail(cfg);
   ```

4. Timing Parameters:
   - Adjust based on display datasheet
   - Fine-tune for stability:
   ```cpp
   cfg.hsync_front_porch = 80;  // Adjust if display unstable
   cfg.hsync_pulse_width = 4;   // Minimum required by display
   cfg.hsync_back_porch = 16;   // Adjust for position
   ```

## Best Practices

1. Display Initialization:
   - Configure panel before bus
   - Verify timing parameters
   - Enable PSRAM if available
   - Set appropriate backlight control

2. Error Handling:
   - Check initialization status
   - Monitor bus errors
   - Implement recovery mechanisms

3. Performance Monitoring:
   - Verify refresh rates
   - Monitor memory usage
   - Check DMA transfer completion 