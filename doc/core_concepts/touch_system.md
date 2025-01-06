# Touch System in LovyanGFX

The touch system in LovyanGFX provides a flexible interface for handling touch input on various display types. This document explains the core concepts and implementation of the touch system.

## Touch Point Structure

The `touch_point_t` structure represents a single touch point:

```cpp
struct touch_point_t {
    int16_t x = -1;      // X coordinate (-1 = invalid)
    int16_t y = -1;      // Y coordinate (-1 = invalid)
    uint16_t size = 0;   // Touch point size
    uint16_t id = 0;     // Touch point identifier
};
```

## Touch Interface

The `ITouch` interface defines the contract for touch controllers:

### Configuration

```cpp
struct config_t {
    uint32_t freq = 1000000;  // Communication frequency
    uint16_t x_min = 0;       // Minimum X value
    uint16_t x_max = 3600;    // Maximum X value
    uint16_t y_min = 0;       // Minimum Y value
    uint16_t y_max = 3600;    // Maximum Y value
    bool bus_shared = true;   // True if panel and touch share the same bus
    int16_t pin_int = -1;     // Interrupt pin
    int16_t pin_rst = -1;     // Reset pin
    uint8_t offset_rotation = 0; // Touch rotation offset
    
    // Communication interface (SPI or I2C)
    union {
        // SPI Configuration
        struct {
            int8_t spi_host;    // SPI host (VSPI_HOST or HSPI_HOST)
            int16_t pin_sclk;   // Clock pin
            int16_t pin_mosi;   // MOSI pin
            int16_t pin_miso;   // MISO pin
        };
        // I2C Configuration
        struct {
            int8_t i2c_port;    // I2C port number
            int16_t pin_scl;    // SCL pin
            int16_t pin_sda;    // SDA pin
            int16_t i2c_addr;   // I2C address
        };
    };
    int16_t pin_cs = -1;      // Chip select pin (SPI only)
};
```

### Core Methods

```cpp
class ITouch {
public:
    // Initialize the touch controller
    virtual bool init(void) = 0;
    
    // Wake up from sleep mode
    virtual void wakeup(void) = 0;
    
    // Enter sleep mode
    virtual void sleep(void) = 0;
    
    // Check if touch is enabled
    virtual bool isEnable(void) { return true; }
    
    // Get raw touch points
    virtual uint_fast8_t getTouchRaw(touch_point_t* tp, 
                                   uint_fast8_t count) = 0;
    
    // Configuration access
    config_t config(void) const;
    void config(const config_t& config);
    
    // Check if using SPI interface
    bool isSPI(void) const;
};
```

## Implementation Example

Here's a basic example of implementing a touch controller:

```cpp
class MyTouchController : public ITouch {
public:
    bool init(void) override {
        if (_inited) return true;
        
        // Initialize communication
        if (isSPI()) {
            // Setup SPI
            setupSPI(_cfg.spi_host, _cfg.pin_sclk, 
                    _cfg.pin_mosi, _cfg.pin_miso, _cfg.pin_cs);
        } else {
            // Setup I2C
            setupI2C(_cfg.i2c_port, _cfg.pin_scl, 
                    _cfg.pin_sda, _cfg.i2c_addr);
        }
        
        // Setup interrupt if available
        if (_cfg.pin_int >= 0) {
            pinMode(_cfg.pin_int, INPUT);
        }
        
        // Setup reset if available
        if (_cfg.pin_rst >= 0) {
            pinMode(_cfg.pin_rst, OUTPUT);
            digitalWrite(_cfg.pin_rst, HIGH);
        }
        
        _inited = true;
        return true;
    }
    
    uint_fast8_t getTouchRaw(touch_point_t* tp, 
                            uint_fast8_t count) override {
        uint_fast8_t points = 0;
        
        // Read touch points
        if (readTouchData()) {
            for (uint_fast8_t i = 0; i < count; ++i) {
                if (getTouchPoint(&tp[i])) {
                    ++points;
                }
            }
        }
        
        return points;
    }
    
    void wakeup(void) override {
        // Implementation specific wake up
    }
    
    void sleep(void) override {
        // Implementation specific sleep
    }
};
```

## Usage Example

```cpp
LGFX display;
touch_point_t tp;

void setup() {
    // Configure touch
    auto touch_cfg = display.touch()->config();
    touch_cfg.x_min = 0;
    touch_cfg.x_max = 320;
    touch_cfg.y_min = 0;
    touch_cfg.y_max = 240;
    touch_cfg.pin_int = 36;  // Touch interrupt pin
    display.touch()->config(touch_cfg);
    
    // Initialize display and touch
    display.init();
}

void loop() {
    // Read single touch point
    if (display.touch()->getTouchRaw(&tp, 1)) {
        Serial.printf("Touch: x=%d y=%d\n", tp.x, tp.y);
    }
}
```

## Multi-Touch Support

For controllers supporting multi-touch:

```cpp
touch_point_t tp[5];  // Array for multiple touch points
uint_fast8_t count = display.touch()->getTouchRaw(tp, 5);

for (uint_fast8_t i = 0; i < count; ++i) {
    Serial.printf("Touch %d: x=%d y=%d size=%d id=%d\n",
                 i, tp[i].x, tp[i].y, tp[i].size, tp[i].id);
}
```

## Best Practices

1. **Initialization**
   - Always check init() return value
   - Configure touch parameters before init
   - Handle shared bus configurations properly

2. **Power Management**
   - Use sleep() when touch input not needed
   - Implement proper wake-up sequence
   - Handle interrupt efficiently

3. **Coordinate Handling**
   - Consider screen rotation
   - Implement proper coordinate mapping
   - Handle touch boundaries correctly

4. **Error Handling**
   - Check for invalid touch points
   - Implement timeout for touch reads
   - Handle communication errors gracefully 