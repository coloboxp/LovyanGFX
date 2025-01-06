# Touch Calibration Guide

This guide explains how to configure and calibrate touch functionality in LovyanGFX.

## Touch Architecture

LovyanGFX provides a flexible touch abstraction layer through the `ITouch` interface. The base touch structure is defined as:

```cpp
struct touch_point_t {
    int16_t x = -1;      // X coordinate
    int16_t y = -1;      // Y coordinate
    uint16_t size = 0;   // Touch point size
    uint16_t id = 0;     // Touch point identifier
};
```

## Touch Configuration

### Basic Configuration

Each touch controller requires specific configuration. Here's a typical setup:

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Touch_XPT2046 _touch_instance;

    void init_touch() {
        auto cfg = _touch_instance.config();
        
        // Touch coordinate range
        cfg.x_min = 0;      // Minimum X value (raw)
        cfg.x_max = 3900;   // Maximum X value (raw)
        cfg.y_min = 0;      // Minimum Y value (raw)
        cfg.y_max = 3900;   // Maximum Y value (raw)
        
        // Communication settings
        cfg.freq = 1000000;     // SPI/I2C frequency
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
        
        // Rotation settings
        cfg.offset_rotation = 0;  // Rotation offset (0~7)
        
        _touch_instance.config(cfg);
        _panel_instance.setTouch(&_touch_instance);
    }
};
```

### Controller-Specific Settings

Different touch controllers have default configurations:

```cpp
// XPT2046 (Resistive)
Touch_XPT2046::Touch_XPT2046() {
    _cfg.freq = 1000000;
    _cfg.x_min = 300;
    _cfg.x_max = 3900;
    _cfg.y_min = 400;
    _cfg.y_max = 3900;
}

// FT5x06 (Capacitive)
Touch_FT5x06::Touch_FT5x06() {
    _cfg.i2c_addr = 0x38;
    _cfg.x_min = 0;
    _cfg.x_max = 319;
    _cfg.y_min = 0;
    _cfg.y_max = 319;
    _cfg.freq = 400000;
}

// STMPE610 (Resistive)
Touch_STMPE610::Touch_STMPE610() {
    _cfg.freq = 1000000;
    _cfg.x_min = 240;
    _cfg.x_max = 3750;
    _cfg.y_min = 200;
    _cfg.y_max = 3700;
}
```

## Touch Calibration

### Interactive Calibration

```cpp
void calibrateTouchScreen() {
    uint16_t parameters[8];  // Array to store calibration data
    
    // Start calibration process
    display.calibrateTouch(
        parameters,    // Calibration data array
        TFT_WHITE,    // Foreground color
        TFT_BLACK,    // Background color
        20            // Corner marker size
    );
    
    // Save calibration data (e.g., to preferences)
    #ifdef ESP32
    Preferences preferences;
    preferences.begin("touch_cal", false);
    preferences.putBytes("cal_data", parameters, sizeof(parameters));
    preferences.end();
    #endif
}
```

### Loading Saved Calibration

```cpp
void loadCalibration() {
    uint16_t parameters[8];
    bool calibration_loaded = false;
    
    #ifdef ESP32
    Preferences preferences;
    preferences.begin("touch_cal", true);
    if (preferences.isKey("cal_data")) {
        preferences.getBytes("cal_data", parameters, sizeof(parameters));
        display.setTouchCalibrate(parameters);
        calibration_loaded = true;
    }
    preferences.end();
    #endif
    
    // If no calibration data, perform calibration
    if (!calibration_loaded) {
        calibrateTouchScreen();
    }
}
```

### Calibration Verification

```cpp
bool verifyCalibration() {
    display.fillScreen(TFT_BLACK);
    display.setTextColor(TFT_WHITE, TFT_BLACK);
    display.setTextSize(2);
    
    // Draw test points
    const uint8_t radius = 5;
    const uint8_t points[][2] = {
        {20, 20},                    // Top-left
        {display.width()-20, 20},    // Top-right
        {display.width()/2, display.height()-20}, // Bottom-middle
    };
    
    for (auto& point : points) {
        display.drawCircle(point[0], point[1], radius, TFT_RED);
        
        // Wait for touch
        int16_t x, y;
        uint32_t start = millis();
        bool touched = false;
        
        while (millis() - start < 5000) {  // 5 second timeout
            if (display.getTouch(&x, &y)) {
                // Check if touch is within acceptable range
                int dx = x - point[0];
                int dy = y - point[1];
                if ((dx * dx + dy * dy) <= (radius * radius * 4)) {
                    touched = true;
                    break;
                }
            }
            delay(10);
        }
        
        if (!touched) return false;
        delay(500);  // Debounce
    }
    
    return true;
}
```

## Performance Optimization

### Interrupt-Based Touch Detection

```cpp
volatile bool touch_triggered = false;

void IRAM_ATTR touchInterrupt() {
    touch_triggered = true;
}

void setupTouchInterrupt() {
    auto cfg = _touch_instance.config();
    if (cfg.pin_int >= 0) {
        pinMode(cfg.pin_int, INPUT);
        attachInterrupt(cfg.pin_int, touchInterrupt, FALLING);
    }
}

void loop() {
    if (touch_triggered) {
        int16_t x, y;
        if (display.getTouch(&x, &y)) {
            // Handle touch
        }
        touch_triggered = false;
    }
}
```

### Touch Debouncing

```cpp
class TouchHandler {
    static constexpr uint32_t DEBOUNCE_MS = 50;
    uint32_t last_touch_time = 0;
    int16_t last_x = -1, last_y = -1;
    
public:
    bool getTouch(int16_t* x, int16_t* y) {
        if (millis() - last_touch_time >= DEBOUNCE_MS) {
            if (display.getTouch(x, y)) {
                // Check if touch position has changed significantly
                if (last_x < 0 || last_y < 0 ||
                    abs(*x - last_x) > 5 || 
                    abs(*y - last_y) > 5) {
                    last_touch_time = millis();
                    last_x = *x;
                    last_y = *y;
                    return true;
                }
            } else {
                last_x = last_y = -1;
            }
        }
        return false;
    }
};
```

## Best Practices

1. **Initial Setup**:
```cpp
void initTouch() {
    // Check if touch is available
    if (!display.touch()) {
        Serial.println("Touch not available!");
        return;
    }
    
    // Load or perform calibration
    loadCalibration();
    
    // Verify calibration
    if (!verifyCalibration()) {
        Serial.println("Calibration verification failed!");
        calibrateTouchScreen();
    }
}
```

2. **Error Handling**:
```cpp
bool getTouchPoint(int16_t& x, int16_t& y) {
    if (!display.getTouch(&x, &y)) {
        return false;
    }
    
    // Validate coordinates
    if (x < 0 || x >= display.width() || 
        y < 0 || y >= display.height()) {
        return false;
    }
    
    return true;
}
```

3. **Power Management**:
```cpp
void enterSleepMode() {
    display.touch()->sleep();  // Put touch controller in sleep mode
}

void exitSleepMode() {
    display.touch()->wakeup(); // Wake up touch controller
}
```

These implementations provide a foundation for touch calibration and handling in LovyanGFX. Adapt the code according to your specific touch controller and requirements. 