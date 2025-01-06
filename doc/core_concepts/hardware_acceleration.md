# Hardware Acceleration Features

This guide explains the hardware acceleration features available in LovyanGFX and how to use them effectively.

## Hardware-Accelerated Operations

### Basic Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Enable hardware acceleration
        {
            auto cfg = _panel_instance.config();
            
            cfg.use_psram = true;        // Use PSRAM if available
            cfg.dma_channel = 1;         // Enable DMA
            cfg.readable = true;         // Enable hardware reading
            cfg.bus_shared = false;      // Dedicated bus for better performance
            
            _panel_instance.config(cfg);
        }
        
        // Configure SPI for maximum performance
        {
            auto cfg = _bus_instance.config();
            cfg.freq_write = 80000000;   // 80MHz SPI clock
            cfg.freq_read  = 20000000;   // 20MHz read clock
            cfg.spi_mode = 0;            // Mode 0 for most displays
            cfg.use_lock = true;         // Enable SPI transaction locking
            
            _bus_instance.config(cfg);
        }
    }
};
```

### Hardware Fill Operations

```cpp
void hardwareFill() {
    // Hardware-accelerated screen fill
    display.fillScreen(TFT_BLACK);
    
    // Hardware-accelerated rectangle fill
    display.fillRect(0, 0, 100, 100, TFT_RED);
    
    // Hardware-accelerated area fill
    uint16_t color = TFT_BLUE;
    display.startWrite();
    display.setWindow(0, 0, 239, 319);
    display.writeColor(color, 240 * 320);  // Fill entire window
    display.endWrite();
}
```

### Hardware Line Drawing

```cpp
void hardwareLines() {
    // Hardware-accelerated horizontal line
    display.drawFastHLine(0, 0, 240, TFT_RED);
    
    // Hardware-accelerated vertical line
    display.drawFastVLine(0, 0, 320, TFT_GREEN);
    
    // Hardware-accelerated line drawing
    display.startWrite();
    for (int i = 0; i < 100; i++) {
        display.drawLine(
            random(240), random(320),
            random(240), random(320),
            TFT_WHITE
        );
    }
    display.endWrite();
}
```

### Hardware Rectangle Operations

```cpp
void hardwareRectangles() {
    // Hardware-accelerated rectangle drawing
    display.startWrite();
    
    // Filled rectangle with hardware acceleration
    display.fillRect(10, 10, 100, 100, TFT_RED);
    
    // Rectangle outline with hardware acceleration
    display.drawRect(20, 20, 80, 80, TFT_WHITE);
    
    // Rounded rectangle with hardware acceleration
    display.fillRoundRect(30, 30, 60, 60, 10, TFT_BLUE);
    
    display.endWrite();
}
```

## Hardware-Accelerated Blitting

### Basic Blitting

```cpp
void hardwareBlitting() {
    // Create source buffer
    static LGFX_Sprite sprite(&display);
    sprite.createSprite(100, 100);
    
    // Draw something in the sprite
    sprite.fillScreen(TFT_BLACK);
    sprite.drawRect(0, 0, 100, 100, TFT_WHITE);
    
    // Hardware-accelerated blit to display
    display.startWrite();
    sprite.pushSprite(0, 0);
    display.endWrite();
}
```

### Rotated Blitting

```cpp
void rotatedBlitting() {
    static LGFX_Sprite sprite(&display);
    sprite.createSprite(100, 100);
    
    // Draw in sprite
    sprite.fillScreen(TFT_BLACK);
    sprite.drawRect(0, 0, 100, 100, TFT_WHITE);
    
    // Hardware-accelerated rotated blit
    float angle = 45.0;  // 45 degree rotation
    float zoom = 1.0;    // No scaling
    
    display.startWrite();
    sprite.pushRotateZoom(
        120, 160,    // Destination center
        angle,       // Rotation angle
        zoom, zoom,  // X and Y scaling
        TFT_BLACK    // Background color
    );
    display.endWrite();
}
```

### Transparent Blitting

```cpp
void transparentBlitting() {
    static LGFX_Sprite sprite(&display);
    sprite.createSprite(100, 100);
    
    // Draw with transparency
    sprite.fillScreen(TFT_BLACK);  // Black will be transparent
    sprite.fillCircle(50, 50, 40, TFT_RED);
    
    // Hardware-accelerated transparent blit
    display.startWrite();
    sprite.pushSprite(0, 0, TFT_BLACK);  // Black is transparent
    display.endWrite();
}
```

## Hardware Color Operations

### Color Conversion

```cpp
void hardwareColorConversion() {
    // Hardware-accelerated RGB888 to RGB565 conversion
    uint32_t rgb888 = 0x00FF8800;  // Orange in RGB888
    uint16_t rgb565 = display.color565(
        (rgb888 >> 16) & 0xFF,  // Red
        (rgb888 >> 8) & 0xFF,   // Green
        rgb888 & 0xFF           // Blue
    );
    
    // Hardware-accelerated color fill
    display.startWrite();
    display.fillRect(0, 0, 100, 100, rgb565);
    display.endWrite();
}
```

### Color Manipulation

```cpp
void hardwareColorManipulation() {
    // Hardware-accelerated color operations
    display.startWrite();
    
    // Color inversion
    display.invertDisplay(true);
    delay(1000);
    display.invertDisplay(false);
    
    // Brightness control (if supported by hardware)
    for (int i = 0; i <= 255; i++) {
        display.setBrightness(i);
        delay(10);
    }
    
    display.endWrite();
}
```

## Performance Optimization

### Transaction Optimization

```cpp
void optimizeTransactions() {
    // Batch operations for better performance
    display.startWrite();
    
    // Multiple drawing operations
    for (int i = 0; i < 100; i++) {
        int x = random(240);
        int y = random(320);
        int w = random(50) + 10;
        int h = random(50) + 10;
        
        // Hardware-accelerated operations
        display.fillRect(x, y, w, h, random(0xFFFF));
    }
    
    display.endWrite();
}
```

### Memory Management

```cpp
void optimizeMemory() {
    // Use PSRAM for large sprites
    static LGFX_Sprite sprite(&display);
    
    // Create sprite in PSRAM if available
    if (psramFound()) {
        sprite.createSprite(
            240, 320,
            MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT
        );
    } else {
        // Fallback to smaller sprite in RAM
        sprite.createSprite(120, 160);
    }
}
```

## Hardware-Specific Features

### ESP32 Specific

```cpp
#ifdef ESP32
void esp32Features() {
    // Enable hardware-specific optimizations
    auto cfg = display.config();
    
    // Use ESP32's hardware SPI
    cfg.pin_sclk = 18;  // Hardware SPI pins
    cfg.pin_mosi = 23;
    cfg.pin_miso = 19;
    
    // Enable hardware CS control
    cfg.pin_cs = 5;
    cfg.cs_active_level = 0;
    
    // Use hardware rotation
    display.setRotation(0);  // Hardware-accelerated rotation
}
#endif
```

### Performance Monitoring

```cpp
class HardwareMonitor {
    unsigned long start_time;
    unsigned long operation_count;
    
public:
    void startMonitoring() {
        start_time = micros();
        operation_count = 0;
    }
    
    void countOperation() {
        operation_count++;
    }
    
    void printStats() {
        unsigned long duration = micros() - start_time;
        float ops_per_second = (operation_count * 1000000.0f) / duration;
        
        Serial.printf(
            "Operations: %lu\n"
            "Duration: %lu us\n"
            "Speed: %.2f ops/sec\n",
            operation_count,
            duration,
            ops_per_second
        );
    }
};
```

These examples demonstrate various hardware acceleration features in LovyanGFX. Adapt them according to your specific hardware capabilities and requirements. 