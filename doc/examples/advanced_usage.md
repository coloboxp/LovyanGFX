# Advanced Usage Guide

This guide covers advanced features and integrations in LovyanGFX, including LVGL integration, power management, and specialized display support.

## LVGL Integration

LovyanGFX provides seamless integration with LVGL (Light and Versatile Graphics Library) for creating sophisticated user interfaces.

### Basic Setup

```cpp
#define LGFX_AUTODETECT
#include <LGFX_AUTODETECT.hpp>
#include <lvgl.h>

// Display configuration
static const uint16_t screenWidth = 320;
static const uint16_t screenHeight = 240;

// LVGL buffer setup
static lv_disp_draw_buf_t draw_buf;
static lv_color_t buf[2][screenWidth * 10];  // Double buffering

// LovyanGFX instance
LGFX gfx;

// Display flush callback
void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
    if (gfx.getStartCount() == 0) {
        gfx.startWrite();
    }
    
    gfx.pushImageDMA(
        area->x1,
        area->y1,
        area->x2 - area->x1 + 1,
        area->y2 - area->y1 + 1,
        (lgfx::swap565_t*)&color_p->full
    );
    
    lv_disp_flush_ready(disp);
}

// Touch input callback
void my_touchpad_read(lv_indev_drv_t *indev_driver, lv_indev_data_t *data) {
    uint16_t touchX, touchY;
    data->state = LV_INDEV_STATE_REL;
    
    if (gfx.getTouch(&touchX, &touchY)) {
        data->state = LV_INDEV_STATE_PR;
        data->point.x = touchX;
        data->point.y = touchY;
    }
}

void setup() {
    // Initialize display
    gfx.begin();
    
    // Initialize LVGL
    lv_init();
    lv_disp_draw_buf_init(&draw_buf, buf[0], buf[1], screenWidth * 10);
    
    // Configure display driver
    static lv_disp_drv_t disp_drv;
    lv_disp_drv_init(&disp_drv);
    disp_drv.hor_res = screenWidth;
    disp_drv.ver_res = screenHeight;
    disp_drv.flush_cb = my_disp_flush;
    disp_drv.draw_buf = &draw_buf;
    lv_disp_drv_register(&disp_drv);
    
    // Configure touch input
    static lv_indev_drv_t indev_drv;
    lv_indev_drv_init(&indev_drv);
    indev_drv.type = LV_INDEV_TYPE_POINTER;
    indev_drv.read_cb = my_touchpad_read;
    lv_indev_drv_register(&indev_drv);
}

void loop() {
    lv_timer_handler();  // Handle LVGL tasks
    delay(1);
}
```

### LVGL with DMA Support

```cpp
// Enable DMA for faster display updates
void my_disp_flush_dma(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
    uint32_t size = (area->x2 - area->x1 + 1) * (area->y2 - area->y1 + 1);
    
    gfx.startWrite();
    gfx.setAddrWindow(area->x1, area->y1, area->x2 - area->x1 + 1, area->y2 - area->y1 + 1);
    gfx.pushPixelsDMA((uint16_t*)color_p, size);
    
    lv_disp_flush_ready(disp);
}
```

## Power Management

### Deep Sleep Support

```cpp
class LGFX_Sleep : public LGFX {
public:
    // Prepare display for sleep
    void prepareSleep() {
        startWrite();
        writeCommand(0x10);  // Enter sleep mode
        endWrite();
        delay(5);
    }
    
    // Wake up display
    void wakeup() {
        startWrite();
        writeCommand(0x11);  // Exit sleep mode
        endWrite();
        delay(120);
    }
};

LGFX_Sleep display;

void enterDeepSleep() {
    display.prepareSleep();
    
    // Configure ESP32 deep sleep
    esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);
    esp_deep_sleep_start();
}

void wakeupFromSleep() {
    display.wakeup();
    display.setBrightness(128);
}
```

## E-Paper Display Support

### Basic EPD Setup

```cpp
class LGFX_EPD : public LGFX {
    // EPD-specific configuration
    lgfx::Panel_IT8951 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX_EPD() {
        // Configure SPI bus
        auto bus_cfg = _bus_instance.config();
        bus_cfg.spi_host = VSPI_HOST;
        bus_cfg.freq_write = 40000000;
        bus_cfg.pin_sclk = 18;
        bus_cfg.pin_mosi = 23;
        bus_cfg.pin_miso = 19;
        bus_cfg.pin_dc = 27;
        _bus_instance.config(bus_cfg);
        _panel_instance.setBus(&_bus_instance);
        
        // Configure panel
        auto panel_cfg = _panel_instance.config();
        panel_cfg.pin_cs = 5;
        panel_cfg.pin_rst = 12;
        panel_cfg.pin_busy = 13;
        _panel_instance.config(panel_cfg);
        
        setPanel(&_panel_instance);
    }
};
```

### EPD Update Modes

```cpp
void updateEPD() {
    // Full update
    display.setEpdMode(epd_mode_t::epd_quality);
    display.fillScreen(TFT_WHITE);
    display.display();  // Apply changes
    
    // Partial update
    display.setEpdMode(epd_mode_t::epd_fastest);
    display.fillRect(0, 0, 100, 100, TFT_BLACK);
    display.display();  // Apply changes
}
```

## Advanced Drawing Techniques

### Custom Shader Effects

```cpp
// Implement custom pixel shader
void customShader(int32_t x, int32_t y, uint32_t &color) {
    // Gradient effect
    uint8_t r = (color >> 16) & 0xFF;
    uint8_t g = (color >> 8) & 0xFF;
    uint8_t b = color & 0xFF;
    
    float factor = (float)x / display.width();
    
    r = (uint8_t)(r * factor);
    g = (uint8_t)(g * factor);
    b = (uint8_t)(b * factor);
    
    color = (r << 16) | (g << 8) | b;
}

// Apply shader
void applyShader() {
    display.setShader(customShader);
    display.fillRect(0, 0, display.width(), display.height(), TFT_WHITE);
    display.removeShader();
}
```

### Custom Fonts and Glyphs

```cpp
// Define custom font data
static const uint8_t customFont[] = {
    // Font data here
};

void setupCustomFont() {
    // Create font instance
    lgfx::GFXfont font;
    font.glyph = customFont;
    
    // Set font
    display.setFont(&font);
    display.setTextSize(1);
}
```

## Best Practices

1. **LVGL Integration**
   - Use appropriate buffer sizes
   - Enable DMA when possible
   - Handle touch input properly
   - Maintain consistent frame timing

2. **Power Management**
   - Implement proper sleep/wake sequences
   - Handle display state correctly
   - Consider partial updates
   - Monitor power consumption

3. **EPD Handling**
   - Use appropriate update modes
   - Minimize full refreshes
   - Handle busy states correctly
   - Consider temperature effects

4. **Performance**
   - Use DMA when available
   - Optimize update regions
   - Batch drawing operations
   - Monitor memory usage

## Common Issues and Solutions

1. **LVGL Display Issues**
   ```cpp
   // Handle display tearing
   disp_drv.full_refresh = 1;  // Enable full refresh
   disp_drv.sw_rotate = 0;     // Use hardware rotation
   ```

2. **EPD Refresh Problems**
   ```cpp
   // Handle incomplete updates
   if (!display.waitDisplay()) {
       // Handle timeout
       display.reset();
   }
   ```

3. **Memory Management**
   ```cpp
   // Handle memory constraints
   #define LV_MEM_SIZE (32 * 1024)  // Adjust LVGL memory pool
   static lv_color_t buf[LV_HOR_RES_MAX * 10];  // Adjust buffer size
   ``` 