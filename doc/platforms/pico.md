# Raspberry Pi Pico Support

LovyanGFX provides support for the Raspberry Pi Pico microcontroller using the Pico SDK.

## Setup

### CMake Configuration

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.12)

# Initialize Pico SDK
include(pico_sdk_import.cmake)
pico_sdk_init()

project(lgfx_pico)

# Add LovyanGFX
add_subdirectory(LovyanGFX)

add_executable(${PROJECT_NAME}
    main.cpp
)

# Enable USB output, disable UART
pico_enable_stdio_usb(${PROJECT_NAME} 1)
pico_enable_stdio_uart(${PROJECT_NAME} 0)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
    pico_stdlib
    hardware_spi
    hardware_i2c
    LovyanGFX::lvgl
)

# Create UF2 file
pico_add_extra_outputs(${PROJECT_NAME})
```

## SPI Configuration

Example configuration for SPI display:

```cpp
#define LGFX_USE_V1
#include <LovyanGFX.hpp>

class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ST7789 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX(void) {
        { // Configure bus
            auto cfg = _bus_instance.config();
            
            cfg.spi_host = 0;     // SPI0
            cfg.pin_sclk = 18;    // SCK pin
            cfg.pin_mosi = 19;    // MOSI pin
            cfg.pin_miso = -1;    // MISO not used
            cfg.freq_write = 40000000;
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        { // Configure display
            auto cfg = _panel_instance.config();
            
            cfg.pin_cs = 17;      // CS pin
            cfg.pin_rst = 20;     // RST pin
            cfg.pin_dc = 16;      // DC pin
            
            cfg.panel_width = 240;
            cfg.panel_height = 320;
            cfg.offset_x = 0;
            cfg.offset_y = 0;
            
            _panel_instance.config(cfg);
        }
        
        setPanel(&_panel_instance);
    }
};
```

## I2C Configuration

Example configuration for I2C display:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SSD1306 _panel_instance;
    lgfx::Bus_I2C _bus_instance;

public:
    LGFX(void) {
        { // Configure I2C bus
            auto cfg = _bus_instance.config();
            
            cfg.i2c_host = 0;     // I2C0
            cfg.pin_sda = 4;      // SDA pin
            cfg.pin_scl = 5;      // SCL pin
            cfg.freq_write = 400000; // 400kHz
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        { // Configure display
            auto cfg = _panel_instance.config();
            
            cfg.panel_width = 128;
            cfg.panel_height = 64;
            cfg.offset_x = 0;
            cfg.offset_y = 0;
            
            _panel_instance.config(cfg);
        }
        
        setPanel(&_panel_instance);
    }
};
```

## Basic Usage

```cpp
LGFX lcd;

int main() {
    stdio_init_all();
    
    lcd.init();
    lcd.setRotation(1);
    lcd.fillScreen(TFT_BLACK);
    
    while (true) {
        lcd.setCursor(0, 0);
        lcd.setTextColor(TFT_WHITE, TFT_BLACK);
        lcd.println("LovyanGFX on Pico");
        
        sleep_ms(100);
    }
    
    return 0;
}
```

## Performance Optimization

1. Use appropriate SPI frequencies:
```cpp
// Optimize SPI speed for your display
cfg.freq_write = 62500000;  // 62.5MHz
cfg.freq_read = 16000000;   // 16MHz
```

2. Enable DMA for better performance:
```cpp
// In bus configuration
cfg.dma_channel = 1;  // Use DMA channel 1
cfg.use_dma = true;   // Enable DMA transfers
```

## Example: Hardware Test

```cpp
#include <LovyanGFX.hpp>
#include "pico/stdlib.h"

LGFX lcd;

void setup() {
    lcd.init();
    lcd.setRotation(1);
    lcd.fillScreen(TFT_BLACK);
}

void loop() {
    static int x = 0;
    
    lcd.drawFastVLine(x, 0, lcd.height(), TFT_BLUE);
    x = (x + 1) % lcd.width();
    
    if (x == 0) {
        lcd.fillScreen(TFT_BLACK);
    }
    
    sleep_ms(10);
}

int main() {
    setup();
    while(true) loop();
}
``` 