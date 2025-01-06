# LGFX_Device Core Class

The LGFX_Device class serves as the foundation of LovyanGFX, providing the base implementation for all display devices. This class inherits from LovyanGFX and implements the core functionality for display control, drawing operations, and hardware interfacing.

## Architecture Overview

The LGFX_Device class uses a modular architecture with three main components:

```cpp
class LGFX_Device {
    protected:
        Panel_Device* _panel;      // Display panel interface
        Touch_Device* _touch;      // Touch interface (optional)
        Light_Device* _light;      // Backlight control (optional)
};
```

Each component handles specific functionality:
- Panel_Device: Manages display communication and drawing operations
- Touch_Device: Handles touch input processing (when available)
- Light_Device: Controls display backlight and brightness

## Device Configuration

The LGFX_Device class uses a builder pattern for configuration. Here's a typical setup:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;  // Display controller
    lgfx::Bus_SPI _bus_instance;          // Communication bus
    lgfx::Light_PWM _light_instance;      // Backlight control

public:
    LGFX(void) {
        { // Bus Configuration
            auto cfg = _bus_instance.config();
            cfg.spi_host = VSPI_HOST;     // SPI host selection
            cfg.pin_sclk = 18;            // Clock pin
            cfg.pin_mosi = 23;            // MOSI pin
            cfg.pin_miso = -1;            // MISO pin (unused)
            cfg.freq_write = 40000000;    // Write clock frequency
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        { // Panel Configuration
            auto cfg = _panel_instance.config();
            cfg.pin_cs = 5;               // Chip select pin
            cfg.pin_rst = 33;             // Reset pin
            cfg.pin_dc = 27;              // Data/Command pin
            cfg.memory_width = 240;       // Display width
            cfg.memory_height = 320;      // Display height
            _panel_instance.config(cfg);
        }

        { // Backlight Configuration
            auto cfg = _light_instance.config();
            cfg.pin_bl = 32;              // Backlight control pin
            cfg.invert = false;           // Backlight polarity
            _light_instance.config(cfg);
        }

        setPanel(&_panel_instance);       // Attach panel
        setLight(&_light_instance);       // Attach backlight
    }
};
```

## Core Functions

### Initialization and Control

```cpp
// Device initialization
bool init(void);                    // Full initialization with reset
bool begin(void);                   // Alias for init()
bool init_without_reset(void);      // Initialize without reset sequence

// Display control
void initBus(void);                // Initialize communication bus
void releaseBus(void);             // Release communication bus
void setPanel(Panel_Device* panel); // Set display panel

// Power management
void sleep(void);                  // Enter sleep mode
void wakeup(void);                 // Exit sleep mode
void powerSave(bool flg);          // Set power save mode
void powerSaveOn(void);            // Enable power save
void powerSaveOff(void);           // Disable power save
```

### Display Communication

```cpp
// Low-level communication
void writeCommand(uint8_t cmd);              // Send command byte
void writeCommand16(uint16_t cmd);           // Send 16-bit command
void writeData(uint8_t data);                // Send data byte
void writeData16(uint16_t data);             // Send 16-bit data
void writeData32(uint32_t data);             // Send 32-bit data

// Hardware capabilities
bool isReadable(void) const;                 // Check if display supports reading
bool getSwapBytes(void) const;               // Check if byte swap is needed
bool hasPalette(void) const;                 // Check palette support
```

### Drawing Operations

```cpp
// Basic drawing
void drawPixel(int32_t x, int32_t y, uint32_t color);
void startWrite(void);             // Begin batch drawing
void endWrite(void);               // End batch drawing

// Optimized operations
void writePixels(const uint16_t* data, int32_t len);
void writePixels(const void* data, int32_t len, uint32_t* palette);
void writeImage(const uint8_t* data, int32_t w, int32_t h, uint32_t format);
```

## Platform Support

LGFX_Device supports multiple platforms through specialized implementations:

- ESP32 family (ESP32, ESP32-S2, ESP32-S3)
- Raspberry Pi Pico (RP2040)
- SDL (PC/Desktop)
- OpenCV
- Linux Framebuffer
- STM32

Each platform may have specific configuration requirements and capabilities. Refer to the platform-specific documentation for details.

## Example Usage

Here's a complete example showing display initialization and basic drawing:

```cpp
LGFX display;  // Create display instance

void setup() {
    display.init();           // Initialize display
    display.setRotation(1);   // Set rotation (0-3)
    display.setBrightness(128); // Set brightness (0-255)
    
    // Basic drawing operations
    display.fillScreen(TFT_BLACK);
    display.setTextColor(TFT_WHITE);
    display.drawString("LovyanGFX", 10, 10);
    
    // Hardware accelerated operations
    display.startWrite();
    display.fillRect(0, 0, 100, 100, TFT_BLUE);
    display.endWrite();
}
```

The LGFX_Device class provides the foundation for all display operations in LovyanGFX. Understanding its architecture and proper configuration is essential for optimal performance and functionality. 