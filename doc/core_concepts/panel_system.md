# Panel System in LovyanGFX

The Panel system in LovyanGFX provides a comprehensive interface for managing display panels. This document explains the core concepts and implementation of the panel system, focusing on the `Panel_Device` class which serves as the base for specific panel implementations.

## Panel Configuration

The `Panel_Device` class uses a configuration structure that defines all panel parameters:

```cpp
struct config_t {
    // Pin Configuration
    int16_t pin_cs = -1;    // Chip Select pin
    int16_t pin_rst = -1;   // Reset pin
    int16_t pin_busy = -1;  // Busy signal pin
    
    // Display Dimensions
    uint16_t memory_width = 240;   // Maximum width supported by driver
    uint16_t memory_height = 240;  // Maximum height supported by driver
    uint16_t panel_width = 240;    // Actual display width
    uint16_t panel_height = 240;   // Actual display height
    
    // Display Offsets
    uint16_t offset_x = 0;         // X-axis offset
    uint16_t offset_y = 0;         // Y-axis offset
    uint8_t offset_rotation = 0;   // Rotation offset (0-7, 4-7 inverted)
    
    // Read Configuration
    uint8_t dummy_read_pixel = 8;  // Dummy bits before pixel read
    uint8_t dummy_read_bits = 1;   // Dummy bits before data read
    uint16_t end_read_delay_us = 0;// Delay after read (e.g., ST7796)
    
    // Panel Properties
    bool readable = true;          // Data can be read back
    bool invert = false;          // Color inversion (true for IPS)
    bool rgb_order = false;       // RGB (true) or BGR (false) order
    bool dlen_16bit = false;      // 16-bit data alignment
    bool bus_shared = true;       // Bus shared with filesystem
};
```

## Core Components

### 1. Bus Interface
```cpp
void initBus(void);              // Initialize communication bus
void releaseBus(void);          // Release bus resources
void setBus(IBus* bus);         // Set bus interface
IBus* getBus(void) const;       // Get current bus interface
```

### 2. Display Control
```cpp
bool init(bool use_reset);      // Initialize panel
void writeCommand(uint32_t data, uint_fast8_t length);
void writeData(uint32_t data, uint_fast8_t length);
void display(uint_fast16_t x, uint_fast16_t y, 
             uint_fast16_t w, uint_fast16_t h);
```

### 3. Touch Support
```cpp
bool initTouch(void);           // Initialize touch interface
void setTouch(ITouch* touch);   // Set touch controller
uint_fast8_t getTouchRaw(touch_point_t* tp, uint_fast8_t count);
void touchCalibrate(void);      // Calibrate touch input
```

### 4. Backlight Control
```cpp
void setLight(ILight* light);   // Set backlight controller
void setBrightness(uint8_t brightness);
```

## DMA Support

The panel system includes DMA (Direct Memory Access) support for efficient data transfer:

```cpp
void initDMA(void);            // Initialize DMA
void waitDMA(void);            // Wait for DMA completion
bool dmaBusy(void);            // Check DMA status
```

## Protected Methods

These methods can be overridden by specific panel implementations:

### Pin Control
```cpp
virtual void init_cs(void);     // Initialize CS pin
virtual void cs_control(bool level);  // Control CS pin
virtual void init_rst(void);    // Initialize RST pin
virtual void rst_control(bool level); // Control RST pin
```

### Panel-Specific Implementation
```cpp
virtual const uint8_t* getInitCommands(uint8_t listno) const;
virtual fastread_dir_t get_fastread_dir(void) const;
```

## Example Implementation

Here's an example of implementing a custom panel:

```cpp
class MyPanel : public Panel_Device {
public:
    MyPanel(void) {
        // Set default configuration
        _cfg.panel_width = 320;
        _cfg.panel_height = 240;
        _cfg.memory_width = 320;
        _cfg.memory_height = 240;
    }

protected:
    const uint8_t* getInitCommands(uint8_t listno) const override {
        static constexpr uint8_t init_cmds[] = {
            0xEF, 3, 0x03, 0x80, 0x02,
            0xCF, 3, 0x00, 0xC1, 0x30,
            0xED, 4, 0x64, 0x03, 0x12, 0x81,
            0xE8, 3, 0x85, 0x00, 0x78,
            0xCB, 5, 0x39, 0x2C, 0x00, 0x34, 0x02,
            0xF7, 1, 0x20,
            0xEA, 2, 0x00, 0x00,
            CMD_INIT_DELAY, 150,    // 150ms delay
            0, 0xFF                 // End of list
        };
        if (listno == 0) return init_cmds;
        return nullptr;
    }
};
```

## Usage Example

```cpp
LGFX display;
MyPanel panel;

void setup() {
    // Configure panel
    auto cfg = panel.config();
    cfg.pin_cs = 5;     // CS pin
    cfg.pin_rst = 27;   // Reset pin
    cfg.pin_busy = 19;  // Busy pin
    
    // Set bus configuration
    cfg.bus_shared = true;
    cfg.dlen_16bit = false;
    
    // Apply configuration
    panel.config(cfg);
    
    // Initialize display with panel
    display.setPanel(&panel);
    display.init();
}
```

## Best Practices

1. **Initialization**
   - Always check init() return value
   - Handle reset properly if used
   - Configure pins before initialization

2. **Bus Management**
   - Handle shared bus scenarios correctly
   - Release bus when not in use
   - Consider DMA for large transfers

3. **Power Management**
   - Implement proper sleep/wake sequences
   - Handle backlight efficiently
   - Use power save modes when appropriate

4. **Error Handling**
   - Check communication status
   - Handle initialization failures
   - Validate configuration parameters 