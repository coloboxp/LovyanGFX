# Hardware Acceleration Features

This guide explains the hardware acceleration features available in LovyanGFX and how to use them effectively. Hardware acceleration significantly improves performance by offloading graphics operations to dedicated hardware.

## Understanding Hardware Acceleration

### Hardware vs Software Rendering
Graphics operations can be performed in two ways:
1. Software Rendering:
   - CPU performs calculations
   - Slower but more flexible
   - Works on all hardware
   - Higher memory usage

2. Hardware Acceleration:
   - Dedicated hardware performs operations
   - Significantly faster (2-10x)
   - Hardware-specific features
   - Lower CPU usage
   - Reduced memory bandwidth

### Available Hardware Features
Different platforms offer various acceleration features:
- ESP32: DMA transfers, SPI transactions
- RP2040: PIO-based acceleration
- STM32: DMA2D (Chrom-ART)
- Arduino: Platform-specific features

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
            
            cfg.use_psram = true;        // Use PSRAM for large buffers
            cfg.dma_channel = 1;         // DMA channel for memory transfers
            cfg.readable = true;         // Enable hardware-accelerated reading
            cfg.bus_shared = false;      // Dedicated bus for maximum performance
            
            _panel_instance.config(cfg);
        }
        
        // Configure SPI for maximum performance
        {
            auto cfg = _bus_instance.config();
            // 80MHz clock for write operations
            // Most displays support 80MHz in SPI mode 0
            cfg.freq_write = 80000000;   
            
            // 20MHz for read operations
            // Lower frequency needed for reliable reads
            cfg.freq_read  = 20000000;   
            
            // SPI Mode 0: CPOL=0, CPHA=0
            // Clock idles low, data sampled on rising edge
            cfg.spi_mode = 0;            
            
            // Enable multi-threaded protection
            cfg.use_lock = true;         
            
            _bus_instance.config(cfg);
        }
    }
};
```

### Memory Transfer Optimization

#### DMA (Direct Memory Access)
DMA allows direct memory-to-peripheral transfers without CPU intervention:
```cpp
// Configure DMA for optimal transfers
{
    auto cfg = _panel_instance.config();
    
    // DMA burst size optimization
    cfg.dma_channel = 1;           // Dedicated DMA channel
    cfg.dma_burst_len = 32;        // 32-byte bursts for efficiency
    cfg.dma_desc_count = 4;        // Multiple descriptors for chaining
    
    // Memory alignment for DMA
    // Buffers must be 32-bit aligned for optimal DMA
    cfg.memory_width = 32;         // 32-bit aligned transfers
    cfg.dma_buffer_size = 1024;    // 1KB DMA buffer
    
    _panel_instance.config(cfg);
}
```

### Hardware Fill Operations

```cpp
void hardwareFill() {
    // Hardware-accelerated screen fill
    // Uses DMA for large area fills
    display.fillScreen(TFT_BLACK);
    
    // Hardware-accelerated rectangle fill
    // Automatically uses DMA for large areas
    display.fillRect(0, 0, 100, 100, TFT_RED);
    
    // Optimized area fill using window command
    // 1. Sets drawing window
    // 2. Streams color data using DMA
    // 3. Automatically handles memory alignment
    uint16_t color = TFT_BLUE;
    display.startWrite();
    display.setWindow(0, 0, 239, 319);    // Define fill area
    display.writeColor(color, 240 * 320);  // Fill using DMA
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

### Memory Access Patterns

```cpp
void optimizeMemoryAccess() {
    // Align data access to 32-bit boundaries
    // This enables optimal DMA transfers
    uint32_t* buffer = (uint32_t*)heap_caps_aligned_alloc(
        32,                     // 32-byte alignment
        320 * 240 * 2,         // Buffer size (RGB565)
        MALLOC_CAP_DMA         // DMA-capable memory
    );
    
    // Use aligned writes for maximum performance
    display.startWrite();
    display.setWindow(0, 0, 319, 239);
    display.writePixels(buffer, 320 * 240);
    display.endWrite();
    
    heap_caps_free(buffer);
}
```

### Bus Optimization

```cpp
void optimizeBusTransfers() {
    // Configure optimal bus settings
    auto cfg = _bus_instance.config();
    
    // Write frequency calculation:
    // Maximum SPI clock = min(MCU limit, Display limit)
    // ESP32: 80MHz max
    // Most displays: 60-80MHz max
    cfg.freq_write = std::min(80000000, display.getMaxClock());
    
    // Read operations require slower clock
    // Typically 1/4 of write frequency
    cfg.freq_read = cfg.freq_write / 4;
    
    // Use hardware CS for faster transactions
    cfg.spi_cs = SPI_CS_PIN;      // Hardware CS pin
    cfg.use_lock = true;          // Enable transaction locking
    
    _bus_instance.config(cfg);
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