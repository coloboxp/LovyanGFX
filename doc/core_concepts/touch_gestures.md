# Touch Gesture System Guide

This guide explains how to implement touch gestures in LovyanGFX, covering both basic touch handling and advanced gesture recognition.

## Touch Architecture

LovyanGFX provides a flexible touch abstraction layer through the `ITouch` interface and various touch controller implementations. The base touch structure is defined as:

```cpp
struct touch_point_t {
    int16_t x = -1;      // X coordinate
    int16_t y = -1;      // Y coordinate
    uint16_t size = 0;   // Touch point size
    uint16_t id = 0;     // Touch point identifier
};
```

## Supported Touch Controllers

LovyanGFX supports multiple touch controller types:

```cpp
// Resistive touch controllers
lgfx::Touch_XPT2046    // Common 4-wire resistive touch
lgfx::Touch_RA8875     // Integrated LCD + touch controller

// Capacitive touch controllers
lgfx::Touch_FT5x06     // FocalTech capacitive touch
lgfx::Touch_GT911      // Goodix capacitive touch
lgfx::Touch_CST816S    // Hynitron capacitive touch with gesture support
lgfx::Touch_GSLx680    // Silead capacitive touch
lgfx::Touch_TT21xxx    // Focal Tech TT21xxx series
lgfx::Touch_NS2009     // NS2009 resistive touch
lgfx::Touch_STMPE610   // STMPE610 resistive touch
```

## Touch Configuration

Each touch controller requires specific configuration. Here's a typical setup:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Touch_XPT2046 _touch_instance;

    void init_touch() {
        auto cfg = _touch_instance.config();
        
        // Touch coordinate range
        cfg.x_min = 300;    // Raw minimum X value
        cfg.x_max = 3900;   // Raw maximum X value
        cfg.y_min = 400;    // Raw minimum Y value
        cfg.y_max = 3900;   // Raw maximum Y value
        
        // Communication settings
        cfg.freq = 1000000;     // SPI clock frequency
        cfg.bus_shared = true;  // Share bus with display
        cfg.pin_int = -1;       // Interrupt pin (-1 = disable)
        
        // SPI settings (if using SPI)
        cfg.pin_sclk = 18;    // SCLK pin
        cfg.pin_mosi = 23;    // MOSI pin
        cfg.pin_miso = 19;    // MISO pin
        cfg.pin_cs   = 22;    // CS pin
        
        // I2C settings (if using I2C)
        cfg.i2c_addr = 0x38;  // I2C address
        cfg.pin_sda  = 21;    // SDA pin
        cfg.pin_scl  = 22;    // SCL pin
        
        _touch_instance.config(cfg);
        _panel_instance.setTouch(&_touch_instance);
    }
};
```

## Basic Touch Handling

### Single Touch Detection

```cpp
void loop() {
    int16_t x, y;
    if (display.getTouch(&x, &y)) {
        // Touch detected at (x,y)
        display.fillCircle(x, y, 5, TFT_RED);
    }
}
```

### Multi-Touch Detection

```cpp
void loop() {
    static constexpr uint8_t MAX_TOUCHES = 5;
    lgfx::touch_point_t points[MAX_TOUCHES];
    
    uint8_t count = display.getTouchRaw(points, MAX_TOUCHES);
    for (uint8_t i = 0; i < count; i++) {
        // Handle each touch point
        display.fillCircle(points[i].x, points[i].y, 5, TFT_RED);
    }
}
```

## Advanced Gesture Recognition

### CST816S Gesture Support

The CST816S controller provides built-in gesture recognition:

```cpp
enum CS816S_GESTURE {
    NONE         = 0x00,
    SWIPE_UP     = 0x01,
    SWIPE_DOWN   = 0x02,
    SWIPE_LEFT   = 0x03,
    SWIPE_RIGHT  = 0x04,
    SINGLE_CLICK = 0x05,
    DOUBLE_CLICK = 0x0B,
    LONG_PRESS   = 0x0C
};
```

### Custom Gesture Detection

For controllers without built-in gesture support, implement custom gesture detection:

```cpp
class TouchGesture {
    struct TouchPoint {
        int16_t x, y;           // Current position
        int16_t start_x, start_y; // Starting position
        uint32_t start_time;    // Touch start time
        bool active;            // Touch point active
    };
    
    // Gesture thresholds
    struct {
        uint32_t tap_duration = 200;      // Max tap duration (ms)
        uint32_t long_press = 500;        // Long press duration
        int16_t swipe_threshold = 50;     // Min swipe distance
        int16_t zoom_threshold = 10;      // Min zoom distance
        int16_t rotation_threshold = 5;   // Min rotation angle
    } config;
    
    void update() {
        int16_t x, y;
        if (display.getTouch(&x, &y)) {
            if (!point.active) {
                // Touch start
                point.active = true;
                point.start_x = point.x = x;
                point.start_y = point.y = y;
                point.start_time = millis();
            } else {
                // Touch move
                int16_t dx = x - point.start_x;
                int16_t dy = y - point.start_y;
                uint32_t duration = millis() - point.start_time;
                
                // Detect gestures
                if (duration < config.tap_duration && abs(dx) < 10 && abs(dy) < 10) {
                    onTap(x, y);
                } else if (duration >= config.long_press) {
                    onLongPress(x, y);
                } else if (abs(dx) >= config.swipe_threshold || abs(dy) >= config.swipe_threshold) {
                    if (abs(dx) > abs(dy)) {
                        onSwipe(dx > 0 ? SWIPE_RIGHT : SWIPE_LEFT);
                    } else {
                        onSwipe(dy > 0 ? SWIPE_DOWN : SWIPE_UP);
                    }
                }
            }
        } else if (point.active) {
            // Touch end
            point.active = false;
        }
    }
};
```

## Performance Optimization

1. **Interrupt Handling**: Use touch interrupts when available to reduce polling:
```cpp
void IRAM_ATTR touchInterrupt() {
    touch_triggered = true;
}

void setup() {
    // Configure touch interrupt
    if (touch.config().pin_int >= 0) {
        pinMode(touch.config().pin_int, INPUT);
        attachInterrupt(touch.config().pin_int, touchInterrupt, FALLING);
    }
}
```

2. **Debouncing**: Implement touch debouncing to prevent false triggers:
```cpp
uint32_t last_touch_time = 0;
const uint32_t DEBOUNCE_MS = 50;

void loop() {
    if (millis() - last_touch_time >= DEBOUNCE_MS) {
        if (display.getTouch(&x, &y)) {
            last_touch_time = millis();
            // Handle touch
        }
    }
}
```

3. **Coordinate Filtering**: Smooth touch coordinates for better accuracy:
```cpp
void filterCoordinates(int16_t& x, int16_t& y) {
    static const uint8_t FILTER_SAMPLES = 4;
    static int16_t x_samples[FILTER_SAMPLES];
    static int16_t y_samples[FILTER_SAMPLES];
    static uint8_t sample_index = 0;
    
    x_samples[sample_index] = x;
    y_samples[sample_index] = y;
    sample_index = (sample_index + 1) % FILTER_SAMPLES;
    
    int32_t x_sum = 0, y_sum = 0;
    for (uint8_t i = 0; i < FILTER_SAMPLES; i++) {
        x_sum += x_samples[i];
        y_sum += y_samples[i];
    }
    
    x = x_sum / FILTER_SAMPLES;
    y = y_sum / FILTER_SAMPLES;
}
```

## Best Practices

1. **Touch Calibration**: Always calibrate touch coordinates for accurate mapping:
```cpp
void calibrateTouchScreen() {
    display.calibrateTouch(nullptr, TFT_WHITE, TFT_RED, 30);
}
```

2. **Error Handling**: Implement robust error checking:
```cpp
bool getTouchPoint(int16_t& x, int16_t& y) {
    if (!display.getTouch(&x, &y)) {
        return false;
    }
    
    // Validate coordinates
    if (x < 0 || x >= display.width() || y < 0 || y >= display.height()) {
        return false;
    }
    
    return true;
}
```

3. **Power Management**: Implement touch sleep/wake functionality:
```cpp
void sleepTouch() {
    touch.sleep();  // Put touch controller in sleep mode
}

void wakeupTouch() {
    touch.wakeup(); // Wake up touch controller
}
```

These implementations provide a foundation for building touch-enabled interfaces in LovyanGFX. Adapt the code according to your specific touch controller and requirements. 