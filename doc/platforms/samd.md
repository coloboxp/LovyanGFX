# Arduino SAMD21/SAMD51 Platform Guide

This guide explains how to configure and optimize LovyanGFX for Arduino SAMD21 and SAMD51 microcontrollers.

## Basic Configuration

### SPI Display Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_SPI _bus_instance;
    lgfx::Light_PWM _light_instance;

public:
    LGFX() {
        // Bus configuration
        {
            auto cfg = _bus_instance.config();
            
            // SAMD specific SPI configuration
            #ifdef ARDUINO_SAMD_MKRVIDOR4000  // SAMD21
                cfg.spi_host = SPI;      // Default SPI
                cfg.freq_write = 24000000;  // Write clock (24MHz max)
                cfg.freq_read = 12000000;   // Read clock
            #else  // SAMD51
                cfg.spi_host = SPI;      // Default SPI
                cfg.freq_write = 48000000;  // Write clock (48MHz max)
                cfg.freq_read = 24000000;   // Read clock
            #endif
            
            cfg.spi_mode = 0;           // SPI mode (0-3)
            cfg.use_lock = true;        // Transaction locking
            
            // SPI Pins (SAMD default SPI)
            cfg.pin_sclk = PIN_SPI_SCK;   // SCK pin
            cfg.pin_mosi = PIN_SPI_MOSI;  // MOSI pin
            cfg.pin_miso = PIN_SPI_MISO;  // MISO pin
            cfg.pin_dc = 9;               // Data/Command pin
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            
            cfg.pin_cs = PIN_SPI_SS;    // CS pin
            cfg.pin_rst = 8;            // Reset pin
            cfg.pin_busy = -1;          // Busy pin (-1 = disable)
            
            // Panel parameters
            cfg.panel_width = 240;      // Display width
            cfg.panel_height = 320;     // Display height
            cfg.offset_x = 0;           // X offset
            cfg.offset_y = 0;           // Y offset
            cfg.offset_rotation = 0;    // Rotation offset
            cfg.dummy_read_pixel = 8;   // Dummy read bits
            cfg.dummy_read_bits = 1;    // Dummy read bits
            
            // Advanced settings
            cfg.readable = true;        // Enable read function
            cfg.invert = false;         // Panel invert
            cfg.rgb_order = false;      // RGB/BGR order
            cfg.dlen_16bit = false;     // 16bit length
            cfg.bus_shared = true;      // Bus shared with SD card
            
            _panel_instance.config(cfg);
        }

        // Backlight configuration
        {
            auto cfg = _light_instance.config();
            
            cfg.pin_bl = 7;            // Backlight pin
            cfg.invert = false;         // Backlight invert
            cfg.freq = 44100;          // PWM frequency
            cfg.pwm_channel = 0;       // PWM channel
            
            _light_instance.config(cfg);
            _panel_instance.setLight(&_light_instance);
        }

        setPanel(&_panel_instance);
    }
};
```

### I2C Display Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_SSD1306 _panel_instance;
    lgfx::Bus_I2C _bus_instance;

public:
    LGFX() {
        // Bus configuration
        {
            auto cfg = _bus_instance.config();
            
            // SAMD specific I2C configuration
            cfg.i2c_port = Wire;        // Default I2C
            cfg.freq_write = 400000;    // Write clock (400kHz)
            cfg.freq_read = 400000;     // Read clock
            cfg.pin_sda = PIN_WIRE_SDA; // SDA pin
            cfg.pin_scl = PIN_WIRE_SCL; // SCL pin
            cfg.i2c_addr = 0x3C;        // I2C address
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            cfg.panel_width = 128;      // Display width
            cfg.panel_height = 64;      // Display height
            cfg.offset_rotation = 0;    // Rotation offset
            
            _panel_instance.config(cfg);
        }

        setPanel(&_panel_instance);
    }
};
```

### Parallel Display Configuration

```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Panel_ILI9341 _panel_instance;
    lgfx::Bus_Parallel8 _bus_instance;

public:
    LGFX() {
        // Bus configuration
        {
            auto cfg = _bus_instance.config();
            
            // SAMD specific parallel configuration
            cfg.freq_write = 20000000;  // Write clock
            cfg.pin_wr = 2;             // WR pin
            cfg.pin_rd = 3;             // RD pin
            cfg.pin_rs = 4;             // RS pin
            
            // Data pins (8-bit)
            cfg.pin_d0 = 5;
            cfg.pin_d1 = 6;
            cfg.pin_d2 = 7;
            cfg.pin_d3 = 8;
            cfg.pin_d4 = 9;
            cfg.pin_d5 = 10;
            cfg.pin_d6 = 11;
            cfg.pin_d7 = 12;
            
            _bus_instance.config(cfg);
            _panel_instance.setBus(&_bus_instance);
        }

        // Panel configuration
        {
            auto cfg = _panel_instance.config();
            cfg.pin_cs = 13;
            cfg.pin_rst = 1;
            cfg.readable = true;
            
            _panel_instance.config(cfg);
        }

        setPanel(&_panel_instance);
    }
};
```

## SAMD Specific Features

### DMA Configuration

```cpp
class DMAManager {
    Adafruit_ZeroDMA dma;
    DmacDescriptor* descriptor;
    
public:
    void initDMA() {
        // Configure DMA for SPI
        dma.setTrigger(SERCOM4_DMAC_ID_TX);  // SPI SERCOM
        dma.setAction(DMA_TRIGGER_ACTON_BEAT);
        
        // Allocate DMA descriptor
        descriptor = dma.addDescriptor(
            NULL,                     // Source address
            (void*)&SERCOM4->SPI.DATA.reg,  // Destination (SPI data register)
            0,                        // Length (set later)
            DMA_BEAT_SIZE_BYTE,       // Beat size
            true,                     // Source increment
            false,                    // Destination increment
            DMA_ADDRESS_INCREMENT_STEP_SIZE_1  // Step size
        );
    }
    
    void transfer(const uint8_t* buffer, size_t len) {
        // Update source address and length
        descriptor->SRCADDR.reg = (uint32_t)buffer + len;
        descriptor->BTCNT.reg = len;
        
        // Start transfer
        dma.startJob();
        while (dma.isBusy()) {}  // Wait for completion
    }
};
```

### Clock Configuration

```cpp
void configureClock() {
    #ifdef ARDUINO_SAMD_MKRVIDOR4000  // SAMD21
        // Configure GCLK0 at 48MHz
        GCLK->GENDIV.reg = GCLK_GENDIV_ID(0) | GCLK_GENDIV_DIV(1);
        GCLK->GENCTRL.reg = GCLK_GENCTRL_ID(0) |
                           GCLK_GENCTRL_SRC_DFLL48M |
                           GCLK_GENCTRL_IDC |
                           GCLK_GENCTRL_GENEN;
        while (GCLK->STATUS.bit.SYNCBUSY) {}
        
    #else  // SAMD51
        // Configure GCLK0 at 120MHz
        GCLK->GENCTRL[0].reg = GCLK_GENCTRL_DIV(1) |
                               GCLK_GENCTRL_SRC_DPLL0 |
                               GCLK_GENCTRL_IDC |
                               GCLK_GENCTRL_GENEN;
        while (GCLK->SYNCBUSY.bit.GENCTRL0) {}
    #endif
}
```

## Performance Optimization

### SPI Optimization

```cpp
void optimizeSPI() {
    // Use maximum SPI clock
    #ifdef ARDUINO_SAMD_MKRVIDOR4000  // SAMD21
        auto cfg = display.config();
        cfg.freq_write = 24000000;  // 24MHz
        display.config(cfg);
        
        // Configure SPI peripheral
        SERCOM4->SPI.CTRLA.bit.DIPO = 0;  // MISO on PAD[0]
        SERCOM4->SPI.CTRLA.bit.DOPO = 1;  // MOSI on PAD[2], SCK on PAD[3]
        
    #else  // SAMD51
        auto cfg = display.config();
        cfg.freq_write = 48000000;  // 48MHz
        display.config(cfg);
        
        // Configure SPI peripheral
        SERCOM4->SPI.CTRLA.bit.DIPO = 0;
        SERCOM4->SPI.CTRLA.bit.DOPO = 1;
    #endif
}
```

### Memory Management

```cpp
void optimizeMemory() {
    // Align buffers for DMA
    static uint8_t __attribute__((aligned(16))) 
        buffer[240 * 320 * 2];
    
    // Double buffering
    static uint8_t __attribute__((aligned(16))) 
        front_buffer[240 * 320 * 2],
        back_buffer[240 * 320 * 2];
}
```

### Cache Management

```cpp
#ifdef __SAMD51__  // SAMD51 only
void optimizeCache() {
    // Enable data cache
    SCB->CCSIDR = 0;
    SCB->CSSELR = 0;
    SCB->CCR |= SCB_CCR_DC_Msk;
    SCB->CCSIDR;
    __DSB();
    
    // Enable instruction cache
    SCB->ICIALLU = 0;
    SCB->CCR |= SCB_CCR_IC_Msk;
    __DSB();
    __ISB();
}
#endif
```

## Error Handling

```cpp
class SAMDErrorHandler {
public:
    static void handleSPIError() {
        // Reset SPI peripheral
        SERCOM4->SPI.CTRLA.bit.ENABLE = 0;
        while (SERCOM4->SPI.SYNCBUSY.bit.ENABLE) {}
        
        SERCOM4->SPI.CTRLA.bit.SWRST = 1;
        while (SERCOM4->SPI.SYNCBUSY.bit.SWRST) {}
        
        // Reconfigure SPI
        configureSPI();
    }
    
    static void handleDMAError() {
        // Reset DMA
        DMAC->CTRL.bit.DMAENABLE = 0;
        while (DMAC->CTRL.bit.DMAENABLE) {}
        
        DMAC->CTRL.bit.SWRST = 1;
        while (DMAC->CTRL.bit.SWRST) {}
        
        // Reconfigure DMA
        initDMA();
    }
    
    static void handleClockError() {
        // Reset clock system
        GCLK->CTRLA.bit.SWRST = 1;
        while (GCLK->CTRLA.bit.SWRST) {}
        
        // Reconfigure clocks
        configureClock();
    }
};
```

These examples demonstrate SAMD21/SAMD51-specific features and optimizations in LovyanGFX. Adapt them according to your specific board and requirements. 