# STM32 Platform Guide

This guide covers the STM32 family implementation in LovyanGFX, focusing on the hardware interfaces and DMA capabilities available across different STM32 series.

## Hardware Interfaces

### Display Controllers

1. **FSMC/FMC Controller**
```
Memory Controller Architecture:
┌─────────────────────┐
│    FSMC/FMC Block   │
├─────────┬───────────┤
│NOR/PSRAM│  NAND     │
│Interface│Interface  │
├─────────┼───────────┤
│LCD      │   DMA     │
│Interface│Interface  │
└─────────┴───────────┘
```

- Features:
  - 8/16-bit parallel interface
  - Hardware memory mapping
  - DMA support
  - Up to 100MHz operation

2. **SPI Controller**
```
SPI Architecture:
┌─────────────────────┐
│    SPIx Block       │
├─────────┬───────────┤
│TX FIFO  │ RX FIFO   │
├─────────┼───────────┤
│Clock    │ DMA       │
│Control  │ Interface │
└─────────┴───────────┘
```

- Features:
  - Multiple SPI peripherals
  - DMA support
  - Up to 50MHz operation
  - Hardware CS control

## Bus Configurations

### 1. FSMC/FMC Configuration
```cpp
struct Bus_FSMC::config_t {
    uint32_t freq_write = 40000000;  // Default write frequency: 40MHz
    uint32_t freq_read  = 16000000;  // Default read frequency: 16MHz
    int16_t pin_rd = -1;             // Read strobe pin
    int16_t pin_wr = -1;             // Write strobe pin
    int16_t pin_rs = -1;             // Register/Data select pin
    uint8_t memory_bank = 0;         // FSMC bank number (0-3)
    bool use_dma = true;             // Enable DMA transfers
    DMA_Stream_TypeDef* dma_tx = nullptr;  // DMA stream for TX
    uint8_t dma_channel = 0;         // DMA channel
    uint8_t address_setup = 1;       // Address setup cycles
    uint8_t data_setup = 1;          // Data setup cycles
};
```

### 2. SPI Configuration
```cpp
struct Bus_SPI::config_t {
    uint32_t freq_write = 40000000;  // Default write frequency: 40MHz
    uint32_t freq_read  = 16000000;  // Default read frequency: 16MHz
    int16_t pin_sclk = -1;           // Clock pin
    int16_t pin_miso = -1;           // MISO pin
    int16_t pin_mosi = -1;           // MOSI pin
    int16_t pin_dc   = -1;           // Data/Command pin
    uint8_t spi_mode = 0;            // SPI mode (0-3)
    SPI_TypeDef* spi_host = SPI1;    // SPI peripheral
    bool spi_3wire = true;           // Enable 3-wire SPI mode
    bool use_lock = true;            // Enable SPI bus locking
    DMA_Stream_TypeDef* dma_tx = nullptr;  // DMA stream for TX
    uint8_t dma_channel = 0;         // DMA channel
};
```

### 3. I2C Configuration
```cpp
struct Bus_I2C::config_t {
    uint32_t freq_write = 400000;    // Default write frequency: 400kHz
    int16_t pin_sda = -1;            // SDA pin
    int16_t pin_scl = -1;            // SCL pin
    I2C_TypeDef* i2c_port = I2C1;    // I2C peripheral
    uint8_t i2c_addr = 0x3C;         // I2C device address
    bool use_lock = true;            // Enable I2C bus locking
};
```

## Pin Mapping

### FSMC Pin Options
```
Fixed Pin Assignments for FSMC:
┌─────────┬──────────┬──────────┐
│ Signal  │  Port D  │  Port E  │
├─────────┼──────────┼──────────┤
│ D0-D1   │ PD14-15  │    -     │
│ D2-D3   │ PD0-1    │    -     │
│ D4-D12  │    -     │ PE7-15   │
│ D13-D15 │ PD8-10   │    -     │
│ NOE     │ PD4      │    -     │
│ NWE     │ PD5      │    -     │
│ A0      │ PD11     │    -     │
│ NE1     │ PD7      │    -     │
└─────────┴──────────┴──────────┘
```

### SPI Pin Options

#### SPI1
- SCK:  PA5, PB3
- MISO: PA6, PB4
- MOSI: PA7, PB5
- NSS:  PA4, PA15

#### SPI2
- SCK:  PB13, PB10
- MISO: PB14, PC2
- MOSI: PB15, PC3
- NSS:  PB12, PB9

#### SPI3
- SCK:  PB3, PC10
- MISO: PB4, PC11
- MOSI: PB5, PC12
- NSS:  PA15, PA4

## Performance Features

### DMA Support

The SPI and FSMC implementations include DMA capabilities for efficient data transfer:

```cpp
// DMA configuration for SPI
void initDMA(void) {
    if (_cfg.dma_tx) {
        // Configure DMA stream
        _cfg.dma_tx->CR = 0;
        _cfg.dma_tx->CR = DMA_SxCR_CHSEL_0 * _cfg.dma_channel
                       | DMA_SxCR_MINC       // Memory increment mode
                       | DMA_SxCR_DIR_0      // Memory to peripheral
                       | DMA_SxCR_TCIE;      // Transfer complete interrupt
        
        // Configure DMA request mapping
        _cfg.dma_tx->PAR = (uint32_t)&_cfg.spi_host->DR;
        _cfg.dma_tx->FCR = DMA_SxFCR_DMDIS;  // Disable direct mode
    }
}

// DMA transfer execution
void addDMAQueue(const uint8_t* data, uint32_t length) {
    if (_cfg.dma_tx) {
        _cfg.dma_tx->M0AR = (uint32_t)data;
        _cfg.dma_tx->NDTR = length;
        _cfg.dma_tx->CR |= DMA_SxCR_EN;
    }
}

void execDMAQueue(void) {
    if (_cfg.dma_tx) {
        while (!(_cfg.dma_tx->CR & DMA_SxCR_EN)) {}
        while (!(_cfg.spi_host->SR & SPI_SR_TXE)) {}
    }
}
```

### GPIO Optimization

Direct GPIO manipulation for maximum performance:

```cpp
// Fast GPIO control functions
static inline void gpio_hi(GPIO_TypeDef* port, uint32_t pin) {
    port->BSRR = 1 << pin;
}

static inline void gpio_lo(GPIO_TypeDef* port, uint32_t pin) {
    port->BSRR = 1 << (pin + 16);
}

static inline bool gpio_in(GPIO_TypeDef* port, uint32_t pin) {
    return (port->IDR & (1 << pin)) != 0;
}

// Optimized pin configuration
static void gpio_init(GPIO_TypeDef* port, uint32_t pin, uint32_t mode) {
    uint32_t mask = ~(0xF << (pin * 4));
    port->MODER   = (port->MODER   & mask) | ((mode & 0x3) << (pin * 4));
    port->OSPEEDR = (port->OSPEEDR & mask) | (0x3 << (pin * 4));
    port->OTYPER  &= ~(1 << pin);
    port->PUPDR   = (port->PUPDR   & mask) | (0x1 << (pin * 4));
}
```

## Display Communication

### Write Operations
```cpp
// Write command to display
bool writeCommand(uint32_t data, uint_fast8_t bit_length) {
    dc_control(false);
    write_bytes((uint8_t*)&data, bit_length >> 3);
    return true;
}

// Write data to display
void writeData(uint32_t data, uint_fast8_t bit_length) {
    dc_control(true);
    write_bytes((uint8_t*)&data, bit_length >> 3);
}

// Write repeated data
void writeDataRepeat(uint32_t data, uint_fast8_t bit_length, uint32_t count) {
    dc_control(true);
    if (_cfg.dma_tx) {
        while (count--) {
            write_bytes((uint8_t*)&data, bit_length >> 3);
        }
    } else {
        while (count--) {
            write_bytes((uint8_t*)&data, bit_length >> 3);
        }
    }
}

// Write pixel data
void writePixels(pixelcopy_t* pc, uint32_t length) {
    dc_control(true);
    if (_cfg.dma_tx && pc->no_convert) {
        write_bytes((uint8_t*)pc->src, length * pc->src_bits >> 3);
    } else {
        uint32_t limit = _cfg.dma_tx ? 512 : 32;
        uint8_t buf[512];
        while (length) {
            uint32_t len = length > limit ? limit : length;
            pc->fp_copy(buf, 0, len, pc);
            write_bytes(buf, len * pc->dst_bits >> 3);
            length -= len;
        }
    }
}
```

### Read Operations
```cpp
// Begin read operation
void beginRead(void) {
    dc_control(true);
    if (_cfg.spi_host) {
        _cfg.spi_host->CR1 &= ~SPI_CR1_BIDIOE;
        _cfg.spi_host->CR1 |= SPI_CR1_BIDIMODE | SPI_CR1_RXONLY;
    }
}

// Read data from display
uint32_t readData(uint_fast8_t bit_length) {
    uint32_t res = 0;
    read_bytes((uint8_t*)&res, bit_length >> 3);
    return res;
}

// Read pixel data
void readPixels(void* dst, pixelcopy_t* pc, uint32_t length) {
    uint32_t bytes = length * pc->src_bits >> 3;
    uint8_t buf[512];
    uint32_t limit = 512;
    
    while (bytes) {
        uint32_t len = bytes > limit ? limit : bytes;
        read_bytes(buf, len);
        if (pc->no_convert) {
            memcpy(dst, buf, len);
            dst = (void*)((uint8_t*)dst + len);
        } else {
            pc->fp_copy(dst, buf, len, pc);
        }
        bytes -= len;
    }
}
```

## Platform-Specific Features

### Clock Management
```cpp
// Calculate clock divider for target frequency
uint32_t getPeripheralClock(void) {
    RCC_ClkInitTypeDef clkConfig;
    uint32_t flashLatency;
    HAL_RCC_GetClockConfig(&clkConfig, &flashLatency);
    return HAL_RCC_GetPCLK2Freq();
}

// Set SPI clock frequency
void setClock(uint32_t freq) {
    uint32_t pclk = getPeripheralClock();
    uint32_t div = 0;
    
    // Calculate prescaler
    while (pclk > freq << (div + 1) && div < 7) {
        ++div;
    }
    
    // Configure SPI clock
    _cfg.spi_host->CR1 = (_cfg.spi_host->CR1 & ~SPI_CR1_BR_Msk)
                       | (div << SPI_CR1_BR_Pos);
}
```

### PWM Backlight Control
```cpp
class Light_PWM {
public:
    struct config_t {
        uint8_t pin_bl;              // Backlight pin
        TIM_TypeDef* timer;          // Timer peripheral
        uint8_t channel;             // Timer channel
        uint32_t freq = 12000;       // PWM frequency
        bool invert = false;         // Invert output
    };
    
    bool init(uint8_t brightness) {
        // Configure timer for PWM
        uint32_t period = (HAL_RCC_GetPCLK1Freq() * 2) / _cfg.freq;
        _cfg.timer->PSC = 0;
        _cfg.timer->ARR = period - 1;
        _cfg.timer->CR1 = TIM_CR1_CEN;
        
        // Configure PWM channel
        switch (_cfg.channel) {
            case 1: _cfg.timer->CCMR1 = TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_2; break;
            case 2: _cfg.timer->CCMR1 = TIM_CCMR1_OC2M_1 | TIM_CCMR1_OC2M_2; break;
            case 3: _cfg.timer->CCMR2 = TIM_CCMR2_OC3M_1 | TIM_CCMR2_OC3M_2; break;
            case 4: _cfg.timer->CCMR2 = TIM_CCMR2_OC4M_1 | TIM_CCMR2_OC4M_2; break;
        }
        
        setBrightness(brightness);
        return true;
    }
    
    void setBrightness(uint8_t brightness) {
        uint32_t value = (_cfg.timer->ARR + 1) * brightness / 256;
        if (_cfg.invert) value = _cfg.timer->ARR - value;
        
        switch (_cfg.channel) {
            case 1: _cfg.timer->CCR1 = value; break;
            case 2: _cfg.timer->CCR2 = value; break;
            case 3: _cfg.timer->CCR3 = value; break;
            case 4: _cfg.timer->CCR4 = value; break;
        }
    }
};
```

## Complete Usage Example

Basic setup for STM32 with SPI display:

```cpp
class LGFX : public lgfx::LGFX_Device {
public:
    LGFX(void) {
        // Configure bus
        auto bus = new lgfx::Bus_SPI();
        bus->config().spi_host = SPI1;
        bus->config().spi_mode = 0;
        bus->config().freq_write = 40000000;  // 40MHz
        bus->config().pin_sclk = PA5;
        bus->config().pin_mosi = PA7;
        bus->config().pin_miso = -1;
        bus->config().pin_dc   = PA4;
        
        // Configure DMA
        bus->config().dma_tx = DMA2_Stream3;
        bus->config().dma_channel = 3;
        
        // Configure panel
        auto panel = new lgfx::Panel_ILI9341();
        panel->config().pin_cs = PA3;
        panel->config().pin_rst = PA2;
        panel->config().pin_busy = -1;
        
        // Configure backlight
        auto light = new lgfx::Light_PWM();
        light->config().pin_bl = PA1;
        light->config().timer = TIM2;
        light->config().channel = 2;
        light->config().freq = 12000;
        light->config().invert = false;
        
        // Set bus, panel, and backlight
        setPanel(panel);
        setBus(bus);
        setLight(light);
    }
};

LGFX display;

void setup() {
    display.init();
    display.setRotation(0);
    display.setBrightness(128);
    
    // Basic drawing
    display.fillScreen(TFT_BLACK);
    display.setTextColor(TFT_WHITE);
    display.drawString("Hello STM32!", 10, 10);
}
```

## Best Practices

1. **SPI Configuration**
   - Use appropriate SPI frequency for your display
   - Enable DMA for large transfers
   - Configure correct pin assignments
   - Consider using dedicated SPI peripherals for display

2. **Memory Management**
   - Use DMA for efficient data transfer
   - Implement double buffering for animations
   - Consider available RAM limitations
   - Optimize buffer sizes for performance

3. **Power Management**
   - Implement display sleep mode
   - Use PWM for backlight control
   - Configure appropriate clock frequencies
   - Consider using low-power modes

4. **Error Handling**
   - Check initialization status
   - Validate pin configurations
   - Handle communication errors
   - Monitor DMA transfer completion

## Common Issues and Solutions

1. **Display Initialization**
```cpp
// Handle reset properly
panel->config().pin_rst = PA2;
panel->init(true);  // Use hardware reset

// Verify SPI configuration
if (!bus->config().spi_host) {
    // Handle invalid SPI peripheral
    return false;
}

// Check pin assignments
if (bus->config().pin_sclk < 0 || bus->config().pin_mosi < 0) {
    // Handle invalid pin configuration
    return false;
}
```

2. **SPI Communication**
```cpp
// Reduce speed if experiencing issues
bus->config().freq_write = 20000000;  // Try 20MHz
bus->config().use_lock = true;  // Enable bus locking

// Handle communication errors
if (bus->config().spi_host->SR & SPI_SR_OVR) {
    // Clear overrun flag
    bus->config().spi_host->DR;
    bus->config().spi_host->SR;
}
```

3. **DMA Configuration**
```cpp
// Ensure DMA stream is available
if (bus->config().dma_tx) {
    // Configure DMA priority
    bus->config().dma_tx->CR |= DMA_SxCR_PL_1;  // High priority
    
    // Handle DMA errors
    if (DMA2->LISR & DMA_LISR_TEIF3) {
        // Clear transfer error flag
        DMA2->LIFCR = DMA_LIFCR_CTEIF3;
        // Handle error
        return false;
    }
}
```

4. **Clock Configuration**
```cpp
// Verify peripheral clock
uint32_t pclk = getPeripheralClock();
if (pclk < bus->config().freq_write * 2) {
    // Adjust frequency to match available clock
    bus->config().freq_write = pclk / 2;
}

// Handle clock initialization
RCC_ClkInitTypeDef clkConfig;
uint32_t flashLatency;
if (HAL_RCC_GetClockConfig(&clkConfig, &flashLatency) != HAL_OK) {
    // Handle clock configuration error
    return false;
}
```

## External Resources

### Documentation
- [STM32F4 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0090-stm32f4-series-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [STM32F7 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0385-stm32f7-series-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [FSMC Programming Guide](https://www.st.com/resource/en/application_note/an2784-fsmc-on-stm32f101xx-and-stm32f103xx-microcontrollers-stmicroelectronics.pdf)

### Tools
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
- [STM32 Clock Calculator](https://www.st.com/en/development-tools/clock-configurator.html)

### Example Projects
- [Basic FSMC Example](../examples/FSMC/README.md)
- [SPI Display Example](../examples/SPI/README.md)
- [Performance Tests](../examples/Test/README.md) 