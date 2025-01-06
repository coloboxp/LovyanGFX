# Performance Optimization Guide

This guide provides comprehensive strategies and techniques for optimizing LovyanGFX performance in your applications.

## Hardware Acceleration

### DMA Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Bus configuration with DMA
        {
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
        
        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            cfg.use_psram = true;    // Use PSRAM if available
            cfg.readable = true;     // Enable hardware reading
            cfg.bus_shared = false;  // Dedicated bus
            _panel_instance.config(cfg);
        }
    }
};
```

### Memory Optimization

```cpp
// Memory-aligned buffer allocation
void optimizeMemoryAllocation() {
    // Align to 32-byte boundary for optimal DMA
    const size_t alignment = 32;
    size_t buffer_size = ((width * height * 2 + alignment - 1) 
                         & ~(alignment - 1));
    
    // Allocate DMA-capable memory
    uint8_t* buffer = (uint8_t*)heap_caps_aligned_alloc(
        alignment,
        buffer_size,
        MALLOC_CAP_DMA
    );
    
    // Use PSRAM for large allocations
    if (psramFound()) {
        buffer = (uint8_t*)heap_caps_malloc(
            buffer_size,
            MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT
        );
    }
}
```

### Double Buffering

```cpp
class DoubleBuffer {
    LGFX_Sprite* front_buffer;
    LGFX_Sprite* back_buffer;
    LGFX* display;
    
public:
    DoubleBuffer(LGFX* disp, int width, int height) : display(disp) {
        front_buffer = new LGFX_Sprite(display);
        back_buffer = new LGFX_Sprite(display);
        
        // Create buffers with optimal settings
        front_buffer->setColorDepth(16);  // Use 16-bit color for performance
        back_buffer->setColorDepth(16);
        
        front_buffer->createSprite(width, height);
        back_buffer->createSprite(width, height);
    }
    
    ~DoubleBuffer() {
        delete front_buffer;
        delete back_buffer;
    }
    
    void swap() {
        // Push back buffer to display
        back_buffer->pushSprite(0, 0);
        
        // Swap buffers
        LGFX_Sprite* temp = front_buffer;
        front_buffer = back_buffer;
        back_buffer = temp;
    }
    
    LGFX_Sprite* getDrawBuffer() { return back_buffer; }
};
```

## Drawing Optimization

### Batch Operations

```cpp
void optimizedDrawing() {
    display.startWrite();  // Start transaction
    
    // Multiple drawing operations
    for (int i = 0; i < 100; i++) {
        display.drawPixel(x[i], y[i], color[i]);
        display.drawLine(x1[i], y1[i], x2[i], y2[i], color[i]);
        display.fillRect(rx[i], ry[i], rw[i], rh[i], color[i]);
    }
    
    display.endWrite();  // End transaction
}
```

### Color Optimization

```cpp
void optimizeColorHandling() {
    // Pre-calculate frequently used colors
    static constexpr uint16_t CUSTOM_COLOR = color565(255, 128, 0);
    
    // Use direct color values
    display.fillRect(x, y, w, h, CUSTOM_COLOR);
    
    // Optimize color depth for content
    sprite.setColorDepth(16);  // Use 16-bit color for better performance
    
    // Use indexed colors for limited palettes
    if (num_colors <= 256) {
        sprite.setColorDepth(8);  // 8-bit indexed color
        for (int i = 0; i < num_colors; i++) {
            sprite.setPaletteColor(i, palette[i]);
        }
    }
}
```

## Sprite Optimization

### Memory Management

```cpp
class OptimizedSprite {
private:
    LGFX_Sprite* sprite;
    LGFX* display;
    bool use_psram;
    
public:
    OptimizedSprite(LGFX* disp, bool psram = false) 
    : display(disp), use_psram(psram) {
        sprite = new LGFX_Sprite(display);
    }
    
    bool create(int width, int height, int color_depth = 16) {
        // Calculate memory requirements
        size_t required = width * height * (color_depth / 8);
        
        // Check available memory
        if (use_psram && psramFound()) {
            if (heap_caps_get_free_size(MALLOC_CAP_SPIRAM) < required) {
                return false;
            }
            sprite->setPsram(true);
        } else {
            if (heap_caps_get_free_size(MALLOC_CAP_DMA) < required) {
                return false;
            }
            sprite->setPsram(false);
        }
        
        sprite->setColorDepth(color_depth);
        return sprite->createSprite(width, height);
    }
    
    void optimize() {
        // Enable DMA if available
        sprite->enableDMA();
        
        // Use hardware acceleration
        display->startWrite();
        sprite->pushSpriteDMA(0, 0);
        display->endWrite();
    }
};
```

## Performance Monitoring

### Frame Rate Monitoring

```cpp
class PerformanceMonitor {
private:
    unsigned long start_time;
    unsigned long frame_count;
    float current_fps;
    
public:
    void begin() {
        start_time = micros();
        frame_count = 0;
        current_fps = 0;
    }
    
    void countFrame() {
        frame_count++;
        
        // Update FPS every second
        unsigned long now = micros();
        unsigned long elapsed = now - start_time;
        if (elapsed >= 1000000) {  // 1 second
            current_fps = (frame_count * 1000000.0f) / elapsed;
            start_time = now;
            frame_count = 0;
        }
    }
    
    float getFPS() { return current_fps; }
    
    void report() {
        Serial.printf("FPS: %.2f\n", getFPS());
    }
};
```

### Memory Monitoring

```cpp
class MemoryMonitor {
public:
    void report() {
        Serial.printf("Free heap: %d bytes\n", ESP.getFreeHeap());
        if (psramFound()) {
            Serial.printf("Free PSRAM: %d bytes\n", ESP.getFreePsram());
        }
        Serial.printf("Heap fragmentation: %d%%\n", 
                     ESP.getHeapFragmentation());
    }
    
    bool checkMemory(size_t required) {
        size_t free_heap = ESP.getFreeHeap();
        if (psramFound()) {
            free_heap += ESP.getFreePsram();
        }
        return free_heap >= required;
    }
};
```

## Best Practices

1. **Memory Management**
   - Use appropriate color depth for content
   - Enable PSRAM for large sprites
   - Monitor memory usage
   - Handle allocation failures gracefully

2. **Drawing Operations**
   - Batch similar operations
   - Use hardware acceleration
   - Minimize screen updates
   - Implement double buffering

3. **Hardware Configuration**
   - Enable DMA when available
   - Optimize SPI clock speeds
   - Use hardware-specific features
   - Configure appropriate pin assignments

4. **Resource Management**
   - Clean up unused resources
   - Implement proper initialization
   - Handle memory constraints
   - Monitor performance metrics

## Platform-Specific Optimization

### ESP32 Optimization

```cpp
#ifdef ESP32
void optimizeESP32() {
    // Use ESP32's hardware SPI
    auto cfg = _bus_instance.config();
    cfg.spi_host = VSPI_HOST;
    cfg.dma_channel = 1;
    
    // Enable hardware CS control
    cfg.pin_cs = 5;
    cfg.cs_active_level = 0;
    
    // Use hardware rotation
    display.setRotation(0);  // Hardware-accelerated rotation
}
#endif
```

### Memory Constraints

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

## Common Pitfalls

1. **Memory Issues**
   - Not checking available memory
   - Using inappropriate color depth
   - Memory fragmentation
   - Not cleaning up resources

2. **Performance Issues**
   - Not using batch operations
   - Inefficient drawing patterns
   - Unnecessary screen updates
   - Not using hardware acceleration

3. **Configuration Issues**
   - Incorrect DMA settings
   - Suboptimal SPI speeds
   - Inappropriate color formats
   - Inefficient memory allocation

4. **Resource Management**
   - Memory leaks
   - Unclosed transactions
   - Inefficient sprite usage
   - Poor error handling 