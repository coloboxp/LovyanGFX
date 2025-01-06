# 8-bit Parallel Bus Interface

LovyanGFX's 8-bit parallel bus implementation provides high-speed communication for displays supporting parallel interfaces.

## Basic Configuration

```cpp
namespace lgfx
{
    class Bus_Parallel8 {
    public:
        struct config_t {
            // Data pins D0-D7
            int16_t pin_d0  = -1;
            int16_t pin_d1  = -1;
            int16_t pin_d2  = -1;
            int16_t pin_d3  = -1;
            int16_t pin_d4  = -1;
            int16_t pin_d5  = -1;
            int16_t pin_d6  = -1;
            int16_t pin_d7  = -1;
            
            // Control pins
            int16_t pin_wr  = -1;   // Write strobe
            int16_t pin_rd  = -1;   // Read strobe
            int16_t pin_rs  = -1;   // Register select
            int16_t pin_cs  = -1;   // Chip select
            
            // Timing parameters
            uint32_t freq_write = 20000000;  // Write clock (20MHz default)
            uint32_t freq_read  = 10000000;  // Read clock
            uint8_t write_cycle = 3;         // Write cycle clocks
            uint8_t read_cycle  = 4;         // Read cycle clocks
        };
    };
}
```

## Initialization

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_Parallel8 _bus_instance;

    void init_bus(void) {
        auto cfg = _bus_instance.config();
        
        // Data pins
        cfg.pin_d0 = 0;
        cfg.pin_d1 = 1;
        cfg.pin_d2 = 2;
        cfg.pin_d3 = 3;
        cfg.pin_d4 = 4;
        cfg.pin_d5 = 5;
        cfg.pin_d6 = 6;
        cfg.pin_d7 = 7;
        
        // Control pins
        cfg.pin_wr = 8;
        cfg.pin_rd = 9;
        cfg.pin_rs = 10;
        cfg.pin_cs = 11;
        
        // Timing
        cfg.freq_write = 20000000;
        cfg.write_cycle = 3;
        
        _bus_instance.config(cfg);
    }
};
```

## Data Transfer Operations

```cpp
// Basic write operations
void writeCommand(uint8_t cmd);
void writeData(uint8_t data);
void writeData(uint8_t* data, uint32_t len);

// Read operations
uint8_t readData(void);
void readData(uint8_t* dst, uint32_t len);

// Fast block operations
void writeBlock(uint8_t* data, uint32_t len) {
    startWrite();
    for (uint32_t i = 0; i < len; i++) {
        writeData(data[i]);
    }
    endWrite();
}

// Hardware-optimized writes
void writeBytes(const uint8_t* data, uint32_t len);
void writePixels(const uint16_t* data, uint32_t len);
```

## Performance Optimization

1. Port Manipulation:
```cpp
// Direct port access for maximum speed
#if defined(ESP32)
    #define WRITE_BYTE(b) \
        GPIO.out_w1tc = _mask_lo; \
        GPIO.out_w1ts = _mask_hi & ((b) << _shift);
#endif
```

2. Batch Transfers:
```cpp
// Multiple pixel write
void writePixels(const uint16_t* data, uint32_t len) {
    uint32_t mask_lo = _mask_lo;
    uint32_t mask_hi = _mask_hi;
    do {
        uint16_t pixel = *data++;
        WRITE_BYTE(pixel >> 8);
        WRITE_BYTE(pixel & 0xFF);
    } while (--len);
}
```

3. DMA Support:
```cpp
// DMA configuration
struct dma_config_t {
    int8_t channel;
    void* src_addr;
    size_t size;
    bool circular;
};

void setupDMA(const dma_config_t* cfg);
```

## Hardware-Specific Features

```cpp
#if defined(ESP32)
    // Fast GPIO setup
    void setupGPIO(void) {
        gpio_config_t io_conf = {
            .pin_bit_mask = _pin_mask,
            .mode = GPIO_MODE_OUTPUT,
            .pull_up_en = GPIO_PULLUP_DISABLE,
            .pull_down_en = GPIO_PULLDOWN_DISABLE,
            .intr_type = GPIO_INTR_DISABLE
        };
        gpio_config(&io_conf);
    }
#endif
``` 