# Panel Implementations

LovyanGFX provides a flexible panel abstraction layer through specialized panel implementations. Each panel type inherits from the base `Panel_Device` class and implements specific initialization sequences and configuration requirements.

## Panel Architecture

The panel system uses a hierarchical class structure:

```cpp
namespace lgfx {
    class IPanel;                  // Interface defining core panel operations
    class Panel_Device;           // Base panel implementation
    class Panel_LCD;             // Base LCD panel implementation
    class Panel_HasBuffer;       // Panel with frame buffer support
    class Panel_HasResolution;   // Panel with defined resolution
    
    // Specific panel implementations
    class Panel_ILI9341;        // ILI9341 controller
    class Panel_ST7789;         // ST7789 controller
    class Panel_GC9A01;         // GC9A01 controller
    class Panel_SSD1306;        // SSD1306 OLED controller
}
```

## Panel Configuration Structure

The `Panel_Device` class defines a comprehensive configuration structure:

```cpp
struct config_t {
    // Pin Configuration
    int16_t pin_cs;        // Chip select pin
    int16_t pin_rst;       // Reset pin
    int16_t pin_busy;      // Busy signal pin
    
    // Display Properties
    uint16_t memory_width;   // Memory buffer width
    uint16_t memory_height;  // Memory buffer height
    uint16_t panel_width;    // Visible display width
    uint16_t panel_height;   // Visible display height
    uint16_t offset_x;       // X offset in memory
    uint16_t offset_y;       // Y offset in memory
    
    // Display Options
    bool readable;           // Enable read operations
    bool invert;            // Invert display colors
    bool rgb_order;         // RGB/BGR color order
    bool dlen_16bit;        // 16-bit data length mode
    bool bus_shared;        // Bus sharing with filesystem
};
```

## Core Panel Operations

### Initialization and Setup

```cpp
// Basic initialization
bool init(bool use_reset);     // Initialize with optional reset
void initBus(void);           // Initialize communication bus
void releaseBus(void);        // Release communication bus

// Display Control
void setWindow(int32_t x, int32_t y, int32_t w, int32_t h);
void setRotation(uint8_t r);
void setInvert(bool invert);
void setBrightness(uint8_t brightness);

// Power Management
void setSleep(bool sleep);
void setPowerSave(bool save);
```

### Data Transfer Operations

```cpp
// Command and Data Writing
void writeCommand(uint32_t cmd, uint_fast8_t length);
void writeData(uint32_t data, uint_fast8_t length);

// Display Updates
void display(uint_fast16_t x, uint_fast16_t y, uint_fast16_t w, uint_fast16_t h);
void waitDisplay(void);
bool displayBusy(void);

// DMA Support
void initDMA(void);
void waitDMA(void);
bool dmaBusy(void);
```

## Panel Implementation Examples

### Basic LCD Panel Configuration

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
        cfg.offset_rotation = 0;
        cfg.readable     = true;
        cfg.invert      = false;
        cfg.rgb_order   = false;
        cfg.dlen_16bit  = false;
        
        _panel_instance.config(cfg);
    }
};
```

### OLED Display Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SSD1306 _panel_instance;

    void init_panel(void) {
        auto cfg = _panel_instance.config();
        
        // OLED display settings
        cfg.panel_width   = 128;
        cfg.panel_height  = 64;
        cfg.memory_width  = 128;
        cfg.memory_height = 64;
        
        // Pin configuration
        cfg.pin_rst = 16;
        cfg.pin_cs  = -1;    // Not used in I2C mode
        
        // Display options
        cfg.invert  = false;
        cfg.offset_x = 0;
        cfg.offset_y = 0;
        
        _panel_instance.config(cfg);
    }
};
```

## Platform-Specific Optimizations

### ESP32 Family Support

The panel implementations include optimizations for ESP32 platforms:

- Hardware-specific DMA support
- Optimized SPI and parallel interfaces
- Multiple bus configurations (SPI, I2C, Parallel)

### Raspberry Pi Pico (RP2040) Support

RP2040-specific features include:

- PIO-based parallel interface
- DMA-enabled data transfers
- Flexible pin mapping

## Best Practices

1. Pin Selection:
   - Use hardware-specific pins when available
   - Consider signal integrity for high-speed interfaces
   - Implement proper pin initialization and control

2. Bus Configuration:
   - Choose appropriate bus speed for display type
   - Enable DMA for large data transfers
   - Handle bus sharing properly

3. Display Initialization:
   - Follow manufacturer-recommended startup sequence
   - Implement proper reset timing
   - Configure display parameters correctly

4. Error Handling:
   - Check initialization return values
   - Implement timeout mechanisms
   - Handle bus errors gracefully

The panel implementation system provides a robust foundation for supporting various display types while maintaining consistent interface and behavior across different platforms. 