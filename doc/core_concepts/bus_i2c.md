# I2C Bus Interface

The I2C (Inter-Integrated Circuit) bus implementation in LovyanGFX provides communication support for I2C display controllers and touch interfaces. This document details the configuration and usage of the I2C bus across different platforms.

## Architecture

The Bus_I2C class inherits from IBus and implements platform-specific I2C communication:

```cpp
class Bus_I2C : public IBus {
public:
    struct config_t {
        uint32_t freq_write = 400000;  // Write clock frequency (400kHz default)
        uint32_t freq_read  = 400000;  // Read clock frequency
        int16_t pin_scl = 22;          // Clock pin
        int16_t pin_sda = 21;          // Data pin
        uint8_t i2c_port = 0;          // I2C peripheral number
        uint8_t i2c_addr = 0x3C;       // Device address (common for SSD1306)
        uint32_t prefix_cmd = 0x00;     // Command prefix byte
        uint32_t prefix_data = 0x40;    // Data prefix byte
        uint32_t prefix_len = 1;        // Prefix length in bytes
    };
};
```

## Platform-Specific Configuration

### ESP32 Configuration
```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_I2C _bus_instance;

    LGFX(void) {
        auto cfg = _bus_instance.config();
        
        // I2C port selection
        cfg.i2c_port = 0;           // I2C_NUM_0 or I2C_NUM_1
        cfg.i2c_addr = 0x3C;        // Display address
        
        // Pin assignments (ESP32 defaults)
        cfg.pin_sda = 21;           // SDA pin
        cfg.pin_scl = 22;           // SCL pin
        
        // Speed settings
        cfg.freq_write = 400000;    // 400kHz standard mode
        cfg.freq_read  = 400000;    // 400kHz for reads
        
        // Protocol settings
        cfg.prefix_cmd = 0x00;      // Command prefix
        cfg.prefix_data = 0x40;     // Data prefix
        cfg.prefix_len = 1;         // Prefix length
        
        _bus_instance.config(cfg);
    }
};
```

### Raspberry Pi Pico (RP2040) Configuration
```cpp
class LGFX : public lgfx::LGFX_Device {
    lgfx::Bus_I2C _bus_instance;

    LGFX(void) {
        auto cfg = _bus_instance.config();
        
        // I2C configuration
        cfg.i2c_port = 0;           // I2C0 or I2C1
        cfg.pin_sda = 4;            // GPIO4 for SDA
        cfg.pin_scl = 5;            // GPIO5 for SCL
        cfg.i2c_addr = 0x3C;        // Device address
        
        // Speed settings
        cfg.freq_write = 400000;    // Standard 400kHz
        cfg.freq_read  = 400000;    // Standard 400kHz
        
        _bus_instance.config(cfg);
    }
};
```

## Core Operations

### Initialization
```cpp
bool Bus_I2C::init(void) {
    _state = state_t::state_none;
    return lgfx::i2c::init(_cfg.i2c_port, _cfg.pin_sda, _cfg.pin_scl).has_value();
}

void Bus_I2C::release(void) {
    lgfx::i2c::release(_cfg.i2c_port);
    _state = state_t::state_none;
}
```

### Data Transfer
```cpp
// Command transmission
bool writeCommand(uint32_t cmd) {
    uint8_t buf[] = { static_cast<uint8_t>(_cfg.prefix_cmd), static_cast<uint8_t>(cmd) };
    return writeBytes(buf, 2, false, true);
}

// Data transmission
void writeData(uint32_t data) {
    uint8_t buf[] = { static_cast<uint8_t>(_cfg.prefix_data), static_cast<uint8_t>(data) };
    writeBytes(buf, 2, true, true);
}

// Bulk data transfer
void writeBytes(const uint8_t* data, uint32_t length) {
    if (_cfg.prefix_len) {
        uint8_t prefix = _cfg.prefix_data;
        writeBytes(&prefix, 1, false, true);
    }
    writeBytes(data, length, true, true);
}
```

## Performance Optimization

### 1. Clock Speed Selection
Choose appropriate clock speeds based on device capabilities:

```cpp
// Standard mode (100kHz) for basic operation
cfg.freq_write = 100000;
cfg.freq_read  = 100000;

// Fast mode (400kHz) for better performance
cfg.freq_write = 400000;
cfg.freq_read  = 400000;

// Fast mode plus (1MHz) for supported devices
cfg.freq_write = 1000000;
cfg.freq_read  = 1000000;
```

### 2. Transaction Management
Batch operations for better performance:

```cpp
// Efficient multiple operations
_bus_instance.beginTransaction();
writeCommand(CMD_DISPLAY_ON);
writeCommand(CMD_SET_CONTRAST);
writeData(contrast_value);
_bus_instance.endTransaction();
```

### 3. State Management
Handle I2C bus states properly:

```cpp
enum state_t {
    state_none,          // No active transaction
    state_write_none,    // Write transaction started
    state_write_cmd,     // Command write in progress
    state_write_data,    // Data write in progress
    state_read          // Read operation in progress
};
```

## Error Handling

```cpp
bool Bus_I2C::init(void) {
    // Validate pin configuration
    if (_cfg.pin_scl < 0 || _cfg.pin_sda < 0) {
        return false;    // Required pins not configured
    }
    
    // Initialize I2C with error checking
    auto result = lgfx::i2c::init(_cfg.i2c_port, _cfg.pin_sda, _cfg.pin_scl);
    if (!result.has_value()) {
        return false;    // I2C initialization failed
    }
    
    return true;
}
```

## Platform Compatibility

The Bus_I2C implementation supports multiple platforms with specific optimizations:

- ESP32 family (ESP32, ESP32-S2, ESP32-S3)
  - Hardware I2C with multiple ports
  - Standard and Fast mode support
  - Configurable pins through GPIO matrix

- Raspberry Pi Pico (RP2040)
  - Dual I2C controllers
  - Programmable I2C data rate
  - Flexible pin mapping

- Arduino (Generic)
  - Hardware I2C support
  - Software I2C fallback
  - Platform-dependent speed limits

## Best Practices

1. Pin Selection:
   - Use hardware I2C pins when available
   - Keep wire lengths short
   - Include proper pull-up resistors

2. Speed Configuration:
   - Start with standard mode (100kHz)
   - Test stability at higher speeds
   - Consider bus capacitance

3. Address Management:
   - Verify device address
   - Handle address conflicts
   - Support alternative addresses

The I2C bus interface provides reliable communication for displays and touch controllers. Understanding platform-specific features and limitations ensures optimal performance. 