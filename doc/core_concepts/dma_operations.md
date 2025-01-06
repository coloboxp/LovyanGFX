# DMA Operations and Optimization

This guide explains Direct Memory Access (DMA) operations and optimization techniques in LovyanGFX.

## DMA Configuration

### Basic DMA Setup

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;

public:
    LGFX() {
        // Bus configuration with DMA
        {
            auto cfg = _bus_instance.config();
            
            // Enable DMA
            cfg.dma_channel = 1;     // DMA channel (1 or 2)
            cfg.spi_3wire  = false;  // Must be false for DMA
            cfg.use_lock   = true;   // Enable SPI transaction locking
            
            // SPI configuration for DMA
            cfg.spi_host = VSPI_HOST;     // SPI peripheral
            cfg.freq_write = 80000000;    // 80MHz for DMA
            cfg.freq_read  = 16000000;    // 16MHz for read
            cfg.spi_mode = 0;             // SPI mode
            cfg.pin_sclk = 18;            // SCLK pin
            cfg.pin_mosi = 23;            // MOSI pin
            cfg.pin_miso = 19;            // MISO pin
            cfg.pin_dc   = 27;            // D/C pin
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }
        
        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            
            // Enable DMA-compatible settings
            cfg.memory_width     = 240;   // Must match panel width
            cfg.memory_height    = 320;   // Must match panel height
            cfg.dma_channel     = 1;      // Same as bus DMA channel
            cfg.use_psram       = true;   // Enable PSRAM if available
            
            _panel_instance.config(cfg);
        }
        
        setPanel(&_panel_instance);
    }
};
```

### DMA Buffer Management

```cpp
class DMABuffer {
    static const size_t BUFFER_SIZE = 1024;  // DMA buffer size
    uint16_t* buffer;
    bool using_psram;
    
public:
    DMABuffer() {
        // Allocate DMA-capable memory
        #if defined(ESP32)
            if (psramFound()) {
                buffer = (uint16_t*)ps_malloc(BUFFER_SIZE * sizeof(uint16_t));
                using_psram = true;
            } else {
                buffer = (uint16_t*)heap_caps_malloc(
                    BUFFER_SIZE * sizeof(uint16_t),
                    MALLOC_CAP_DMA
                );
                using_psram = false;
            }
        #else
            buffer = (uint16_t*)malloc(BUFFER_SIZE * sizeof(uint16_t));
            using_psram = false;
        #endif
    }
    
    ~DMABuffer() {
        if (buffer) free(buffer);
    }
    
    uint16_t* get() { return buffer; }
    size_t size() { return BUFFER_SIZE; }
    bool isPSRAM() { return using_psram; }
};
```

## DMA Operations

### Pixel Transfer

```cpp
void dmaPixelTransfer() {
    static DMABuffer dma_buffer;
    uint16_t* buffer = dma_buffer.get();
    
    // Fill buffer with pixel data
    for (size_t i = 0; i < dma_buffer.size(); i++) {
        buffer[i] = TFT_RED;  // Example: fill with red pixels
    }
    
    // Start DMA transfer
    display.startWrite();
    display.setWindow(0, 0, 239, 319);  // Set drawing window
    
    // Transfer pixels using DMA
    display.writePixels(buffer, dma_buffer.size());
    
    display.endWrite();
}
```

### Block Transfer

```cpp
void dmaBlockTransfer(uint16_t* src_buffer, int width, int height) {
    const int block_height = 16;  // Height of each block
    
    display.startWrite();
    
    // Transfer image in blocks
    for (int y = 0; y < height; y += block_height) {
        int block_h = min(block_height, height - y);
        display.setWindow(0, y, width - 1, y + block_h - 1);
        
        // Transfer one block using DMA
        display.writePixels(
            &src_buffer[y * width],
            width * block_h
        );
    }
    
    display.endWrite();
}
```

### Sprite DMA

```cpp
class DMASprite : public LGFX_Sprite {
    DMABuffer _dma_buffer;
    
public:
    DMASprite(LGFX* parent) : LGFX_Sprite(parent) {
        // Create sprite in DMA-capable memory
        #if defined(ESP32)
            createSprite(
                240, 320,
                MALLOC_CAP_DMA | (psramFound() ? MALLOC_CAP_SPIRAM : 0)
            );
        #else
            createSprite(240, 320);
        #endif
    }
    
    void pushDMA(int x, int y) {
        // Push sprite to display using DMA
        pushSprite(x, y);
    }
};
```

## Performance Optimization

### Memory Alignment

```cpp
void optimizeMemoryAlignment() {
    // Align buffer to 32-byte boundary for optimal DMA
    const size_t alignment = 32;
    size_t buffer_size = ((width * height * 2 + alignment - 1) 
                         & ~(alignment - 1));
    
    uint8_t* aligned_buffer = (uint8_t*)heap_caps_aligned_alloc(
        alignment,
        buffer_size,
        MALLOC_CAP_DMA
    );
}
```

### Double Buffering

```cpp
class DoubleBuffer {
    DMASprite* front_buffer;
    DMASprite* back_buffer;
    
public:
    DoubleBuffer(LGFX* display) {
        front_buffer = new DMASprite(display);
        back_buffer = new DMASprite(display);
    }
    
    ~DoubleBuffer() {
        delete front_buffer;
        delete back_buffer;
    }
    
    void swap() {
        DMASprite* temp = front_buffer;
        front_buffer = back_buffer;
        back_buffer = temp;
    }
    
    DMASprite* getFront() { return front_buffer; }
    DMASprite* getBack() { return back_buffer; }
};

void useDoubleBuffer() {
    static DoubleBuffer buffers(&display);
    
    // Draw to back buffer
    auto back = buffers.getBack();
    back->fillScreen(TFT_BLACK);
    back->drawRect(0, 0, 100, 100, TFT_RED);
    
    // Swap and display
    buffers.swap();
    buffers.getFront()->pushDMA(0, 0);
}
```

### Parallel DMA Transfers

```cpp
class ParallelDMA {
    static const int NUM_BUFFERS = 4;
    DMABuffer buffers[NUM_BUFFERS];
    int current_buffer;
    
public:
    ParallelDMA() : current_buffer(0) {}
    
    void transferBlock(uint16_t* data, size_t len) {
        // Get next buffer
        uint16_t* buffer = buffers[current_buffer].get();
        
        // Copy data to DMA buffer
        memcpy(buffer, data, len * sizeof(uint16_t));
        
        // Start DMA transfer
        display.startWrite();
        display.writePixels(buffer, len);
        display.endWrite();
        
        // Move to next buffer
        current_buffer = (current_buffer + 1) % NUM_BUFFERS;
    }
};
```

## Error Handling

```cpp
class DMAManager {
    bool dma_enabled;
    int dma_channel;
    
public:
    bool initDMA() {
        #if defined(ESP32)
            // Check if DMA is available
            if (!psramFound() && heap_caps_get_free_size(MALLOC_CAP_DMA) < 1024) {
                dma_enabled = false;
                return false;
            }
            
            // Try to allocate DMA channel
            for (int ch = 1; ch <= 2; ch++) {
                if (spi_bus_initialize(
                        VSPI_HOST,
                        &bus_cfg,
                        ch  // DMA channel
                    ) == ESP_OK) {
                    dma_channel = ch;
                    dma_enabled = true;
                    return true;
                }
            }
        #endif
        
        dma_enabled = false;
        return false;
    }
    
    void handleDMAError() {
        if (!dma_enabled) {
            // Fall back to non-DMA mode
            display.endWrite();
            display.startWrite();
            // Continue with regular transfers
        }
    }
};
```

## Performance Monitoring

```cpp
class DMAMonitor {
    unsigned long transfer_start;
    size_t bytes_transferred;
    
public:
    void startTransfer(size_t bytes) {
        transfer_start = micros();
        bytes_transferred = bytes;
    }
    
    float endTransfer() {
        unsigned long duration = micros() - transfer_start;
        float mb_per_second = (bytes_transferred / 1048576.0f) 
                             / (duration / 1000000.0f);
        return mb_per_second;
    }
    
    void printStats() {
        float speed = endTransfer();
        Serial.printf("DMA Transfer: %.2f MB/s\n", speed);
    }
};
```

These examples demonstrate various aspects of DMA operations in LovyanGFX. Adapt them according to your specific hardware capabilities and performance requirements. 