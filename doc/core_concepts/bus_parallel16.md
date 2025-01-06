# 16-bit Parallel Bus Interface

LovyanGFX's 16-bit parallel bus implementation enables high-bandwidth communication with displays supporting 16-bit parallel interfaces.

## Basic Configuration

```cpp
namespace lgfx
{
    class Bus_Parallel16 {
    public:
        struct config_t {
            // Data pins D0-D15
            int16_t pin_d0  = -1;
            int16_t pin_d1  = -1;
            int16_t pin_d2  = -1;
            int16_t pin_d3  = -1;
            int16_t pin_d4  = -1;
            int16_t pin_d5  = -1;
            int16_t pin_d6  = -1;
            int16_t pin_d7  = -1;
            int16_t pin_d8  = -1;
            int16_t pin_d9  = -1;
            int16_t pin_d10 = -1;
            int16_t pin_d11 = -1;
            int16_t pin_d12 = -1;
            int16_t pin_d13 = -1;
            int16_t pin_d14 = -1;
            int16_t pin_d15 = -1;
            
            // Control pins
            int16_t pin_wr  = -1;   // Write strobe
            int16_t pin_rd  = -1;   // Read strobe
            int16_t pin_rs  = -1;   // Register select
            int16_t pin_cs  = -1;   // Chip select
            
            // Timing parameters
            uint32_t freq_write = 16000000;  // Write clock (16MHz default)
            uint32_t freq_read  =  8000000;  // Read clock
            uint8_t write_cycle = 2;         // Write cycle clocks
            uint8_t read_cycle  = 4;         // Read cycle clocks
        };
    };
}
```

## Initialization

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_Parallel16 _bus_instance;

    void init_bus(void) {
        auto cfg = _bus_instance.config();
        
        // Lower byte pins (D0-D7)
        cfg.pin_d0  = 0;
        cfg.pin_d1  = 1;
        cfg.pin_d2  = 2;
        cfg.pin_d3  = 3;
        cfg.pin_d4  = 4;
        cfg.pin_d5  = 5;
        cfg.pin_d6  = 6;
        cfg.pin_d7  = 7;
        
        // Upper byte pins (D8-D15)
        cfg.pin_d8  = 8;
        cfg.pin_d9  = 9;
        cfg.pin_d10 = 10;
        cfg.pin_d11 = 11;
        cfg.pin_d12 = 12;
        cfg.pin_d13 = 13;
        cfg.pin_d14 = 14;
        cfg.pin_d15 = 15;
        
        // Control signals
        cfg.pin_wr = 16;
        cfg.pin_rd = 17;
        cfg.pin_rs = 18;
        cfg.pin_cs = 19;
        
        _bus_instance.config(cfg);
    }
};
```

## Data Transfer Operations

```cpp
// Direct 16-bit operations
void writeCommand16(uint16_t cmd);
void writeData16(uint16_t data);
void writeData16(uint16_t* data, uint32_t length);

// Read operations
uint16_t readData16(void);
void readData16(uint16_t* dst, uint32_t length);

// Block transfers
void writeBlock(const uint16_t* data, uint32_t length) {
    startWrite();
    for (uint32_t i = 0; i < length; i++) {
        writeData16(data[i]);
    }
    endWrite();
}
```

## Performance Optimization

1. Direct Port Access:
```cpp
// Fast 16-bit write
#if defined(ESP32)
    #define WRITE_WORD(w) \
        GPIO.out_w1tc = _mask_lo; \
        GPIO.out_w1ts = _mask_hi & ((uint32_t)(w));
#endif
```

2. DMA Operations:
```cpp
// DMA transfer setup
void setupDMA(void) {
    gdma_channel_alloc_config_t dma_conf = {
        .direction = GDMA_CHANNEL_DIRECTION_TX,
        .flags = {
            .reserve_sibling = true,
        },
    };
    gdma_new_channel(&dma_conf, &_dma_chan);
}

// DMA transfer execution
void writePixelsDMA(const uint16_t* data, uint32_t len);
```

3. Burst Transfers:
```cpp
// Optimized burst write
void writeBurst(const uint16_t* data, uint32_t len) {
    uint32_t mask_lo = _mask_lo;
    uint32_t mask_hi = _mask_hi;
    do {
        uint32_t word = *data++;
        WRITE_WORD(word);
        pulse_low(_pin_wr);
    } while (--len);
}
```

## Hardware-Specific Features

```cpp
#if defined(ESP32)
    // Fast parallel bus setup
    void setupParallelBus(void) {
        // Configure all data pins as outputs
        for (int i = 0; i < 16; ++i) {
            gpio_set_direction((gpio_num_t)_cfg.pin_data[i], GPIO_MODE_OUTPUT);
            gpio_set_drive_capability((gpio_num_t)_cfg.pin_data[i], GPIO_DRIVE_CAP_3);
        }
        
        // Configure control pins
        gpio_set_direction((gpio_num_t)_cfg.pin_wr, GPIO_MODE_OUTPUT);
        gpio_set_direction((gpio_num_t)_cfg.pin_rd, GPIO_MODE_OUTPUT);
        gpio_set_direction((gpio_num_t)_cfg.pin_rs, GPIO_MODE_OUTPUT);
        gpio_set_direction((gpio_num_t)_cfg.pin_cs, GPIO_MODE_OUTPUT);
    }
#endif
``` 