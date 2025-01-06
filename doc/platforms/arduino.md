# Arduino Platform Guide

This guide covers the Arduino implementation in LovyanGFX, focusing on hardware interfaces and capabilities across different Arduino boards.

## Hardware Interfaces

### Display Controllers

1. **Hardware SPI Controller**
```
SPI Architecture:
┌─────────────────────┐
│    SPI Controller   │
├─────────┬───────────┤
│TX/RX    │ Clock     │
│Buffer   │ Control   │
├─────────┼───────────┤
│DMA      │ Interrupt │
│Support  │ Control   │
└─────────┴───────────┘
```

- Features:
  - Hardware SPI support
  - DMA capabilities (ESP32/STM32)
  - Up to 80MHz operation (board-dependent)
  - Multiple SPI bus support

2. **Parallel Interface**
```
8080-Style Interface:
┌─────────────────────┐
│  Parallel Interface │
├─────────┬───────────┤
│Data Bus │ Control   │
│(8/16bit)│ Signals   │
├─────────┼───────────┤
│GPIO     │ Port      │
│Direct   │ Mapping   │
└─────────┴───────────┘
```

- Features:
  - 8-bit and 16-bit modes
  - Direct port manipulation
  - High-speed operation
  - Flexible pin mapping

## Bus Configurations

### 1. SPI Configuration
```cpp
struct Bus_SPI::config_t {
    uint32_t freq_write = 40000000;  // Default write frequency: 40MHz
    uint32_t freq_read  = 16000000;  // Default read frequency: 16MHz
    int16_t pin_sclk = -1;           // Clock pin
    int16_t pin_miso = -1;           // MISO pin
    int16_t pin_mosi = -1;           // MOSI pin
    int16_t pin_dc   = -1;           // Data/Command pin
    uint8_t spi_mode = 0;            // SPI mode (0-3)
    uint8_t spi_host = 0;            // SPI host (VSPI/HSPI for ESP32)
    bool spi_3wire = true;           // Enable 3-wire SPI mode
    bool use_lock = true;            // Enable SPI bus locking
    #if defined(ESP32)               // ESP32-specific
    uint8_t dma_channel = 1;         // DMA channel
    #endif
};
```

### 2. Parallel Configuration
```cpp
struct Bus_Parallel8::config_t {
    uint32_t freq_write = 20000000;  // Default write frequency: 20MHz
    int16_t pin_wr = -1;             // Write strobe pin
    int16_t pin_rd = -1;             // Read strobe pin
    int16_t pin_rs = -1;             // Register/Data select pin
    uint8_t pin_d0 = -1;             // Data pin 0
    uint8_t pin_d1 = -1;             // Data pin 1
    uint8_t pin_d2 = -1;             // Data pin 2
    uint8_t pin_d3 = -1;             // Data pin 3
    uint8_t pin_d4 = -1;             // Data pin 4
    uint8_t pin_d5 = -1;             // Data pin 5
    uint8_t pin_d6 = -1;             // Data pin 6
    uint8_t pin_d7 = -1;             // Data pin 7
    bool write_bits = 8;             // Data bus width
    bool use_porta = true;           // Use direct port access
};
```

### 3. I2C Configuration
```cpp
struct Bus_I2C::config_t {
    uint32_t freq_write = 400000;    // Default write frequency: 400kHz
    int16_t pin_sda = -1;            // SDA pin
    int16_t pin_scl = -1;            // SCL pin
    uint8_t i2c_port = 0;            // I2C port number
    uint8_t i2c_addr = 0x3C;         // I2C device address
    bool use_lock = true;            // Enable I2C bus locking
};
```

## Pin Mapping

### Default SPI Pins

#### Arduino UNO/Nano
- MOSI: Pin 11
- MISO: Pin 12
- SCK:  Pin 13
- SS:   Pin 10

#### Arduino Mega
- MOSI: Pin 51
- MISO: Pin 50
- SCK:  Pin 52
- SS:   Pin 53

#### ESP32
- VSPI:
  - MOSI: GPIO 23
  - MISO: GPIO 19
  - SCK:  GPIO 18
  - SS:   GPIO 5
- HSPI:
  - MOSI: GPIO 13
  - MISO: GPIO 12
  - SCK:  GPIO 14
  - SS:   GPIO 15

### Default I2C Pins

#### Arduino UNO/Nano
- SDA: A4
- SCL: A5

#### Arduino Mega
- SDA: Pin 20
- SCL: Pin 21

#### ESP32
- SDA: GPIO 21
- SCL: GPIO 22

## Performance Features

### Direct Port Access

Fast GPIO manipulation for parallel interfaces:

```cpp
// Direct port access for 8-bit bus
#if defined(ARDUINO_ARCH_AVR)
static inline void write8(uint8_t value) {
    PORTD = (PORTD & 0x03) | (value & 0xFC);
    PORTB = (PORTB & 0xF8) | (value & 0x03);
}
#endif

// Fast pin control
static inline void gpio_hi(uint32_t pin) {
    *portOutputRegister(digitalPinToPort(pin)) |= digitalPinToBitMask(pin);
}

static inline void gpio_lo(uint32_t pin) {
    *portOutputRegister(digitalPinToPort(pin)) &= ~digitalPinToBitMask(pin);
}
```

### DMA Support (ESP32)

```cpp
// DMA configuration for ESP32
void initDMA(void) {
    if (_cfg.dma_channel) {
        spi_bus_config_t buscfg = {
            .mosi_io_num = _cfg.pin_mosi,
            .miso_io_num = _cfg.pin_miso,
            .sclk_io_num = _cfg.pin_sclk,
            .quadwp_io_num = -1,
            .quadhd_io_num = -1,
            .max_transfer_sz = 4092,
        };
        spi_bus_initialize(_cfg.spi_host, &buscfg, _cfg.dma_channel);
    }
}

// DMA transfer
void writeBytes(const uint8_t* data, size_t len) {
    if (_cfg.dma_channel) {
        spi_transaction_t t;
        memset(&t, 0, sizeof(t));
        t.length = len * 8;
        t.tx_buffer = data;
        spi_device_transmit(_spi, &t);
    } else {
        SPI.writeBytes(data, len);
    }
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
    #if defined(ESP32)
    if (_cfg.dma_channel) {
        static uint32_t dmabuf[16];
        uint32_t len = bit_length >> 3;
        for (int i = 0; i < 16; ++i) dmabuf[i] = data;
        while (count > 16) {
            writeBytes((uint8_t*)dmabuf, 16 * len);
            count -= 16;
        }
        if (count) writeBytes((uint8_t*)dmabuf, count * len);
    } else
    #endif
    {
        while (count--) {
            write_bytes((uint8_t*)&data, bit_length >> 3);
        }
    }
}
```

### Read Operations
```cpp
// Begin read operation
void beginRead(void) {
    dc_control(true);
    #if defined(ESP32)
    if (_cfg.spi_host) {
        SPI.beginTransaction(SPISettings(_cfg.freq_read, MSBFIRST, SPI_MODE0));
    }
    #endif
}

// Read data from display
uint32_t readData(uint_fast8_t bit_length) {
    uint32_t res = 0;
    read_bytes((uint8_t*)&res, bit_length >> 3);
    return res;
}
```

## Platform-Specific Features

### ESP32 Specific
```cpp
// SPI DMA setup
void setupSPI() {
    spi_bus_config_t buscfg = {
        .mosi_io_num = _cfg.pin_mosi,
        .miso_io_num = _cfg.pin_miso,
        .sclk_io_num = _cfg.pin_sclk,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 4092,
    };
    
    spi_device_interface_config_t devcfg = {
        .command_bits = 0,
        .address_bits = 0,
        .dummy_bits = 0,
        .mode = _cfg.spi_mode,
        .duty_cycle_pos = 128,
        .cs_ena_pretrans = 0,
        .cs_ena_posttrans = 0,
        .clock_speed_hz = _cfg.freq_write,
        .spics_io_num = _cfg.pin_cs,
        .flags = SPI_DEVICE_NO_DUMMY,
        .queue_size = 7,
    };
    
    spi_bus_initialize(_cfg.spi_host, &buscfg, _cfg.dma_channel);
    spi_bus_add_device(_cfg.spi_host, &devcfg, &_spi);
}
```

### AVR Specific
```cpp
// Fast port manipulation
void writeData8(uint8_t data) {
    #if defined(__AVR_ATmega2560__)
    PORTA = data;
    #else
    PORTD = (PORTD & 0x03) | (data & 0xFC);
    PORTB = (PORTB & 0xF8) | (data & 0x03);
    #endif
    pulse_low(_cfg.pin_wr);
}
```

### PWM Backlight Control
```cpp
class Light_PWM {
public:
    struct config_t {
        uint8_t pin_bl;              // Backlight pin
        uint8_t pwm_channel;         // PWM channel
        uint32_t freq = 12000;       // PWM frequency
        uint8_t pwm_bits = 8;        // PWM resolution
        bool invert = false;         // Invert output
    };
    
    bool init(uint8_t brightness) {
        #if defined(ESP32)
        ledcSetup(_cfg.pwm_channel, _cfg.freq, _cfg.pwm_bits);
        ledcAttachPin(_cfg.pin_bl, _cfg.pwm_channel);
        #else
        pinMode(_cfg.pin_bl, OUTPUT);
        #endif
        setBrightness(brightness);
        return true;
    }
    
    void setBrightness(uint8_t brightness) {
        #if defined(ESP32)
        ledcWrite(_cfg.pwm_channel, _cfg.invert ? 255 - brightness : brightness);
        #else
        analogWrite(_cfg.pin_bl, _cfg.invert ? 255 - brightness : brightness);
        #endif
    }
};
```

## Complete Usage Example

Basic setup for Arduino with SPI display:

```cpp
class LGFX : public lgfx::LGFX_Device {
public:
    LGFX(void) {
        // Configure bus
        auto bus = new lgfx::Bus_SPI();
        bus->config().spi_host = 0;  // VSPI (ESP32) or default SPI
        bus->config().spi_mode = 0;
        bus->config().freq_write = 40000000;  // 40MHz
        bus->config().pin_sclk = SCK;
        bus->config().pin_mosi = MOSI;
        bus->config().pin_miso = -1;
        bus->config().pin_dc   = 9;
        
        // Configure panel
        auto panel = new lgfx::Panel_ILI9341();
        panel->config().pin_cs = 10;
        panel->config().pin_rst = 8;
        panel->config().pin_busy = -1;
        
        // Configure backlight
        auto light = new lgfx::Light_PWM();
        light->config().pin_bl = 7;
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
    display.drawString("Hello Arduino!", 10, 10);
}

void loop() {
    // Your code here
}
```

## Best Practices

1. **SPI Configuration**
   - Use hardware SPI when possible
   - Enable DMA on supported platforms
   - Configure appropriate clock speeds
   - Use chip select for multiple devices

2. **Memory Management**
   - Consider limited RAM on AVR platforms
   - Use progmem for constant data
   - Implement efficient buffer strategies
   - Optimize sprite usage

3. **Power Management**
   - Implement display sleep mode
   - Use PWM for backlight control
   - Consider battery-powered operation
   - Optimize refresh rates

4. **Error Handling**
   - Check initialization status
   - Validate pin configurations
   - Handle communication timeouts
   - Monitor memory usage

## Common Issues and Solutions

1. **Display Initialization**
```cpp
// Handle reset properly
panel->config().pin_rst = 8;
panel->init(true);  // Use hardware reset

// Verify SPI configuration
if (!bus->config().spi_host) {
    // Handle invalid SPI configuration
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
#if defined(ESP32)
if (!spi_device_polling_end(_spi, portMAX_DELAY)) {
    // Handle timeout
    return false;
}
#endif
```

3. **Memory Issues**
```cpp
// Handle memory allocation
#if defined(__AVR__)
// Use static allocation for AVR
static uint8_t buffer[512];
#else
// Dynamic allocation for other platforms
uint8_t* buffer = (uint8_t*)malloc(1024);
if (!buffer) {
    // Handle allocation failure
    return false;
}
#endif
```

4. **Performance Optimization**
```cpp
// Optimize drawing operations
void optimizedFill(uint16_t color, uint32_t length) {
    #if defined(ESP32)
    // Use DMA for large fills
    if (length > 32) {
        static uint16_t dmabuf[16];
        for (int i = 0; i < 16; ++i) dmabuf[i] = color;
        while (length > 16) {
            writeBytes((uint8_t*)dmabuf, 32);
            length -= 16;
        }
        if (length) writeBytes((uint8_t*)dmabuf, length * 2);
        return;
    }
    #endif
    // Regular fill for small lengths
    while (length--) writeData(color, 16);
}
```

## External Resources

### Documentation
- [Arduino SPI Reference](https://www.arduino.cc/en/Reference/SPI)
- [ESP32 SPI Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_master.html)
- [AVR Port Manipulation](https://www.arduino.cc/en/Reference/PortManipulation)

### Tools
- [Arduino IDE](https://www.arduino.cc/en/software)
- [ESP32 Arduino Core](https://github.com/espressif/arduino-esp32)

### Example Projects
- [Basic SPI Example](../examples/HowToUse/README.md)
- [Parallel Interface Example](../examples/Parallel/README.md)
- [Performance Tests](../examples/Performance/README.md) 