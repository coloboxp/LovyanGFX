# SPI Bus Interface

The SPI (Serial Peripheral Interface) bus implementation in LovyanGFX provides high-speed communication with display controllers. This document covers the configuration and usage of the SPI bus across different platforms.

## Architecture

The Bus_SPI class inherits from IBus and implements platform-specific SPI communication:

```cpp
class Bus_SPI : public IBus {
public:
    struct config_t {
        uint32_t freq_write = 16000000;  // Write clock frequency (16MHz default)
        uint32_t freq_read  =  8000000;  // Read clock frequency (8MHz default)
        int16_t pin_sclk = -1;   // Clock pin
        int16_t pin_mosi = -1;   // Master Out Slave In pin
        int16_t pin_miso = -1;   // Master In Slave Out pin (optional)
        int16_t pin_dc   = -1;   // Data/Command pin
        uint8_t spi_mode = 0;    // SPI mode (0-3)
        bool spi_3wire = true;   // Enable 3-wire SPI mode
        uint8_t dma_channel = 1; // DMA channel (platform specific)
    };
};
```

## Platform-Specific Configuration

### ESP32 Configuration
```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_SPI _bus_instance;

    LGFX(void) {
        auto cfg = _bus_instance.config();
        
        // SPI host selection
        cfg.spi_host = VSPI_HOST;  // or SPI2_HOST, HSPI_HOST
        
        // Pin assignments (ESP32 defaults)
        cfg.pin_sclk = 18;     // VSPI SCLK
        cfg.pin_mosi = 23;     // VSPI MOSI
        cfg.pin_miso = 19;     // VSPI MISO (optional)
        cfg.pin_dc   = 16;     // Data/Command control
        
        // Performance settings
        cfg.freq_write = 40000000;  // 40MHz
        cfg.freq_read  = 16000000;  // 16MHz
        cfg.spi_mode = 0;           // SPI mode (0-3)
        cfg.spi_3wire = true;       // Enable 3-wire mode
        cfg.dma_channel = 1;        // DMA channel
        
        _bus_instance.config(cfg);
    }
};
```

### Raspberry Pi Pico (RP2040) Configuration
```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_SPI _bus_instance;

    LGFX(void) {
        auto cfg = _bus_instance.config();
        
        // SPI configuration
        cfg.spi_host = 0;          // SPI port number (0 or 1)
        cfg.pin_sclk = 2;          // SCK pin
        cfg.pin_mosi = 3;          // TX/MOSI pin
        cfg.pin_miso = -1;         // RX/MISO pin (set -1 if unused)
        cfg.pin_dc   = 4;          // Data/Command pin
        
        // Speed settings
        cfg.freq_write = 62500000;  // 62.5MHz max for RP2040
        cfg.freq_read  = 8000000;   // 8MHz for reliable reads
        
        _bus_instance.config(cfg);
    }
};
```

## Core Operations

### Initialization
```cpp
bool Bus_SPI::init(void) {
    // Initialize pins
    if (_cfg.pin_dc >= 0) {
        pinMode(_cfg.pin_dc, pin_mode_t::output);
    }
    
    // Configure SPI
    beginTransaction();
    
    return true;
}
```

### Data Transfer
```cpp
// Command transmission
bool writeCommand(uint32_t cmd) {
    dc_l();                    // Set D/C line low for command
    write(cmd);               // Send command byte
    dc_h();                    // Return D/C line high
    return true;
}

// Data transmission
void writeData(uint32_t data) {
    dc_h();                    // Set D/C line high for data
    write(data);              // Send data byte
}

// Bulk data transfer
void writeBytes(const uint8_t* data, uint32_t length) {
    dc_h();                    // Set D/C line high for data
    write(data, length);      // Send multiple bytes
}
```

## Performance Optimization

### 1. Clock Speed Selection
Choose appropriate clock speeds based on your display and platform capabilities:

```cpp
// High-performance configuration
auto cfg = _bus_instance.config();
cfg.freq_write = 80000000;     // 80MHz for fast displays
cfg.freq_read  = 20000000;     // 20MHz for reliable reads
```

### 2. DMA Transfers
Enable DMA for efficient data transfer:

```cpp
// DMA configuration
cfg.dma_channel = 1;           // Select DMA channel
writeBytes(data, length, true); // Use DMA for large transfers
```

### 3. Transaction Management
Batch operations for better performance:

```cpp
// Efficient multiple operations
_bus_instance.beginTransaction();
writeCommand(CMD_CASET);      // Column address set
writeData(x1);               // Start column
writeData(x2);               // End column
writeCommand(CMD_RASET);      // Row address set
writeData(y1);               // Start row
writeData(y2);               // End row
_bus_instance.endTransaction();
```

## Error Handling

```cpp
bool Bus_SPI::init(void) {
    // Validate pin configuration
    if (_cfg.pin_sclk < 0 || _cfg.pin_mosi < 0) {
        return false;    // Required pins not configured
    }
    
    // Validate frequency settings
    if (_cfg.freq_write > 80000000) {
        _cfg.freq_write = 80000000;  // Limit maximum frequency
    }
    
    return true;
}
```

## Platform Compatibility

The Bus_SPI implementation supports multiple platforms with specific optimizations:

- ESP32 family (ESP32, ESP32-S2, ESP32-S3)
  - Hardware SPI with DMA support
  - Multiple SPI peripherals
  - High-speed operation up to 80MHz

- Raspberry Pi Pico (RP2040)
  - Hardware SPI with DMA
  - Dual SPI peripherals
  - Operation up to 62.5MHz

- Arduino (Generic)
  - Hardware SPI support
  - Software fallback for unsupported pins
  - Platform-dependent maximum speeds

## Best Practices

1. Pin Selection:
   - Use hardware SPI pins when possible
   - Keep wires short for high-speed operation
   - Include proper level shifting if needed

2. Speed Optimization:
   - Start with conservative speeds
   - Increase gradually while testing stability
   - Consider signal integrity at high speeds

3. DMA Usage:
   - Use DMA for large transfers
   - Ensure proper buffer alignment
   - Handle DMA limitations per platform

The SPI bus interface provides a flexible and efficient way to communicate with display controllers. Understanding platform-specific features and limitations is key to optimal performance. 