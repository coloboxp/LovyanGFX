# DMA Operations and Optimization

Direct Memory Access (DMA) is a hardware feature that allows peripherals to transfer data directly to/from memory without CPU intervention. This guide explains DMA operations and optimization techniques in LovyanGFX.

## Understanding DMA

### DMA Basics
DMA transfers offer several advantages:
1. CPU Offloading:
   - CPU is free during transfers
   - Can perform other tasks
   - Reduced interrupt overhead

2. Performance Benefits:
   - Higher transfer speeds
   - Lower latency
   - Reduced system bus congestion

3. Memory Considerations:
   - Requires aligned memory
   - Special memory allocation
   - Buffer size restrictions

### Memory Requirements
DMA operations have specific memory requirements:
```cpp
// Memory alignment requirements
#define DMA_ALIGNMENT 32          // 32-byte alignment for ESP32
#define CACHE_LINE_SIZE 32        // Cache line size
#define DMA_BUFFER_SIZE 1024      // Optimal buffer size

// Memory capabilities
#define DMA_MEMORY_FLAGS ( \
    MALLOC_CAP_DMA |           // DMA-capable memory \
    MALLOC_CAP_32BIT |         // 32-bit aligned \
    MALLOC_CAP_INTERNAL        // Internal memory \
)
```

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
            
            // DMA Channel Selection:
            // - Channel 1: Typically for SPI transfers
            // - Channel 2: Available for other peripherals
            cfg.dma_channel = 1;     
            
            // 3-wire SPI mode must be disabled for DMA
            // DMA requires MOSI and MISO to be separate
            cfg.spi_3wire  = false;  
            
            // Enable transaction locking for thread safety
            // Prevents concurrent DMA access
            cfg.use_lock   = true;   
            
            // SPI Configuration for DMA
            cfg.spi_host = VSPI_HOST;     // SPI peripheral (VS or HS)
            cfg.freq_write = 80000000;    // 80MHz max for DMA
            cfg.freq_read  = 16000000;    // Lower for reliable reads
            cfg.spi_mode = 0;             // Mode 0: CPOL=0, CPHA=0
            
            // Pin Configuration
            // Must use DMA-capable pins
            cfg.pin_sclk = 18;            // Hardware SPI CLK
            cfg.pin_mosi = 23;            // Hardware SPI MOSI
            cfg.pin_miso = 19;            // Hardware SPI MISO
            cfg.pin_dc   = 27;            // Data/Command control
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }
        
        // Panel configuration for DMA
        {
            auto cfg = _panel_instance.config();
            
            // Memory dimensions must match panel
            // This ensures proper DMA transfers
            cfg.memory_width     = 240;   // Panel width
            cfg.memory_height    = 320;   // Panel height
            
            // DMA channel must match bus
            cfg.dma_channel     = 1;      
            
            // PSRAM usage for large buffers
            // Falls back to internal memory if unavailable
            cfg.use_psram       = true;   
            
            _panel_instance.config(cfg);
        }
        
        setPanel(&_panel_instance);
    }
};
```

### DMA Buffer Management

```cpp
class DMABuffer {
    // Buffer size calculation:
    // - Multiple of cache line size (32 bytes)
    // - Large enough for efficient transfers
    // - Small enough to fit in memory
    static const size_t BUFFER_SIZE = 1024;  // 1KB optimal for most cases
    
    // Buffer pointer must be aligned
    uint16_t* buffer;
    bool using_psram;
    
public:
    DMABuffer() {
        // Memory allocation strategy:
        #if defined(ESP32)
            if (psramFound()) {
                // PSRAM allocation:
                // - Larger buffers possible
                // - Slightly slower access
                // - Must be cache-aligned
                buffer = (uint16_t*)ps_malloc(
                    BUFFER_SIZE * sizeof(uint16_t)
                );
                using_psram = true;
            } else {
                // DMA-capable memory allocation:
                // - Must be internal memory
                // - 32-bit aligned
                // - Contiguous physical memory
                buffer = (uint16_t*)heap_caps_malloc(
                    BUFFER_SIZE * sizeof(uint16_t),
                    MALLOC_CAP_DMA | MALLOC_CAP_32BIT
                );
                using_psram = false;
            }
        #else
            // Standard allocation for non-ESP platforms
            buffer = (uint16_t*)aligned_alloc(
                32,  // 32-byte alignment
                BUFFER_SIZE * sizeof(uint16_t)
            );
            using_psram = false;
        #endif
    }
    
    ~DMABuffer() {
        if (buffer) {
            // Proper cleanup based on allocation type
            if (using_psram) {
                free(buffer);  // PSRAM allocation
            } else {
                heap_caps_free(buffer);  // DMA memory
            }
        }
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