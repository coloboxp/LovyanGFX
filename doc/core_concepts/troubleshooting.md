# LovyanGFX Troubleshooting Guide

This guide provides solutions for common issues encountered when using LovyanGFX, along with diagnostic steps and best practices for problem resolution.

## Display Initialization Issues

### Display Not Responding

1. **Hardware Connection**
```cpp
// Verify pin configurations
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Double-check pin assignments
        auto cfg = _bus_instance.config();
        cfg.spi_host = VSPI_HOST;     // Verify SPI host
        cfg.pin_sclk = 18;            // Verify SCLK pin
        cfg.pin_mosi = 23;            // Verify MOSI pin
        cfg.pin_miso = 19;            // Verify MISO pin (if used)
        cfg.pin_dc   = 27;            // Verify DC pin
        cfg.pin_cs   = 5;             // Verify CS pin
        cfg.pin_rst  = 33;            // Verify RST pin
        _bus_instance.config(cfg);
    }
};
```

2. **Initialization Sequence**
```cpp
void setup() {
    Serial.begin(115200);
    
    // Add delay for stable power-up
    delay(100);
    
    if (!display.init()) {
        Serial.println("Display initialization failed");
        // Implement fallback or error handling
    }
    
    // Verify basic operations
    display.fillScreen(TFT_BLACK);
    display.setRotation(0);
    display.setBrightness(128);
}
```

### Display Shows Incorrect Colors

1. **Color Format Configuration**
```cpp
// Check color depth and format
void configureDisplay() {
    // Set appropriate color depth
    display.setColorDepth(16);  // Most common: 16-bit color
    
    // Configure RGB/BGR order if colors are inverted
    auto cfg = _panel_instance.config();
    cfg.rgb_order = true;    // Try toggling between true/false
    _panel_instance.config(cfg);
}
```

2. **Color Conversion**
```cpp
// Use correct color conversion methods
uint16_t convertColor(uint8_t r, uint8_t g, uint8_t b) {
    return display.color565(r, g, b);  // Proper 16-bit color conversion
}
```

## Memory Management Issues

### Sprite Creation Failures

1. **Memory Allocation**
```cpp
LGFX_Sprite sprite(&display);

bool createSafeSprite(int width, int height) {
    // Calculate required memory
    size_t required = width * height * 2;  // 16-bit color depth
    
    // Check available memory
    if (heap_caps_get_free_size(MALLOC_CAP_DMA) < required) {
        Serial.println("Insufficient memory for sprite");
        return false;
    }
    
    // Create sprite with error checking
    if (!sprite.createSprite(width, height)) {
        Serial.println("Sprite creation failed");
        return false;
    }
    
    return true;
}
```

2. **Resource Cleanup**
```cpp
void manageSpriteResources() {
    // Delete sprite when no longer needed
    sprite.deleteSprite();
    
    // Wait for memory to be freed
    delay(10);
    
    // Create new sprite
    if (!createSafeSprite(width, height)) {
        // Handle failure
    }
}
```

## Performance Issues

### Slow Drawing Operations

1. **Batch Operations**
```cpp
void optimizedDrawing() {
    display.startWrite();  // Start transaction
    
    // Multiple drawing operations
    for (int i = 0; i < 100; i++) {
        display.drawPixel(x[i], y[i], color[i]);
    }
    
    display.endWrite();  // End transaction
}
```

2. **DMA Configuration**
```cpp
void configureDMA() {
    auto cfg = _bus_instance.config();
    
    // Enable DMA for better performance
    cfg.dma_channel = 1;     // Use DMA channel 1
    cfg.spi_3wire  = false;  // Must be false for DMA
    cfg.use_lock   = true;   // Enable transaction locking
    
    // Optimize SPI speeds
    cfg.freq_write = 80000000;    // 80MHz for write
    cfg.freq_read  = 16000000;    // 16MHz for read
    
    _bus_instance.config(cfg);
}
```

## Touch Screen Issues

### Touch Calibration Problems

1. **Calibration Setup**
```cpp
void calibrateTouch() {
    // Store calibration data
    std::array<int32_t, 8> calibData;
    
    // Perform calibration
    display.touch()->calibrate();
    
    // Save calibration data
    display.touch()->getCal(&calibData[0]);
    
    // Store in non-volatile memory if needed
    // preferences.putBytes("calibData", &calibData[0], sizeof(calibData));
}
```

2. **Touch Validation**
```cpp
void validateTouch() {
    int32_t x, y;
    if (display.touch()->getTouch(&x, &y)) {
        // Convert touch coordinates
        display.convertRawXY(&x, &y);
        
        // Validate coordinates
        if (x < 0 || x >= display.width() || y < 0 || y >= display.height()) {
            Serial.println("Touch coordinates out of bounds");
        }
    }
}
```

## Hardware Compatibility

### Platform-Specific Issues

1. **ESP32 Configuration**
```cpp
#ifdef ESP32
void configureESP32() {
    // Use ESP32's hardware SPI
    auto cfg = _bus_instance.config();
    cfg.spi_host = VSPI_HOST;
    cfg.dma_channel = 1;
    _bus_instance.config(cfg);
}
#endif
```

2. **Memory Constraints**
```cpp
void handleMemoryConstraints() {
    #ifdef ESP32
        if (psramFound()) {
            // Use PSRAM for large sprites
            sprite.createSprite(width, height, MALLOC_CAP_SPIRAM);
        } else {
            // Reduce sprite size for internal RAM
            sprite.createSprite(width/2, height/2);
        }
    #endif
}
```

## Common Issues and Solutions

1. **Display Initialization**
   - Verify power supply stability
   - Check pin configurations
   - Validate SPI/I2C settings
   - Ensure proper hardware connections

2. **Color and Graphics**
   - Verify color depth settings
   - Check RGB/BGR configuration
   - Use appropriate color conversion
   - Validate coordinate systems

3. **Memory Management**
   - Monitor heap usage
   - Implement proper cleanup
   - Use appropriate color depths
   - Handle allocation failures

4. **Performance**
   - Enable hardware acceleration
   - Use batch operations
   - Optimize sprite usage
   - Configure DMA properly

5. **Touch Input**
   - Calibrate touch screen
   - Validate coordinates
   - Handle touch events properly
   - Store calibration data

## Diagnostic Tools

1. **Performance Monitoring**
```cpp
class PerformanceMonitor {
    unsigned long start_time;
    unsigned long frame_count;
    
public:
    void begin() {
        start_time = micros();
        frame_count = 0;
    }
    
    void countFrame() {
        frame_count++;
    }
    
    float getFPS() {
        unsigned long duration = micros() - start_time;
        return (frame_count * 1000000.0f) / duration;
    }
    
    void report() {
        Serial.printf("FPS: %.2f\n", getFPS());
    }
};
```

2. **Memory Monitoring**
```cpp
void monitorMemory() {
    Serial.printf("Free heap: %d\n", ESP.getFreeHeap());
    Serial.printf("Free PSRAM: %d\n", ESP.getFreePsram());
    Serial.printf("Heap fragmentation: %d%%\n", ESP.getHeapFragmentation());
}
```

## Best Practices

1. **Error Handling**
   - Implement proper error checking
   - Add diagnostic logging
   - Create fallback mechanisms
   - Handle hardware limitations

2. **Resource Management**
   - Clean up unused resources
   - Monitor memory usage
   - Implement proper initialization
   - Handle allocation failures

3. **Performance Optimization**
   - Use hardware acceleration
   - Batch drawing operations
   - Optimize memory usage
   - Monitor frame rates

4. **Hardware Configuration**
   - Verify pin assignments
   - Check hardware compatibility
   - Validate bus configurations
   - Test different settings 