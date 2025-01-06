# Bus Interfaces Overview

LovyanGFX supports multiple bus interfaces for communicating with display panels. Each interface type has its own characteristics, advantages, and use cases.

## Supported Bus Types

### SPI Bus
The Serial Peripheral Interface (SPI) is a synchronous serial communication protocol that provides high-speed data transfer with minimal pin usage.

Key features:
- High-speed data transfer (up to 80MHz on ESP32)
- Minimal pin requirements (MOSI, MISO, SCK, CS)
- Hardware acceleration support
- DMA capability for efficient transfers

[Detailed SPI Bus Documentation](bus_spi.md)

### I2C Bus
The Inter-Integrated Circuit (I2C) bus is a two-wire interface suitable for connecting multiple devices on the same bus.

Key features:
- Simple two-wire interface (SDA, SCL)
- Multiple device support with addressing
- Suitable for small displays and touch controllers
- Lower speed compared to SPI (typically 100kHz-400kHz)

[Detailed I2C Bus Documentation](bus_i2c.md)

### Parallel Bus
Parallel bus interfaces provide high-speed data transfer by sending multiple bits simultaneously.

Key features:
- High bandwidth data transfer
- Available in 8-bit and 16-bit configurations
- Hardware acceleration support
- DMA capability for efficient transfers
- Ideal for large displays requiring fast updates

[Detailed Parallel Bus Documentation](bus_parallel.md)

## Bus Selection Guide

### When to Use SPI
- For most general-purpose displays
- When pin count is limited
- When high-speed communication is needed
- For displays that support hardware acceleration

Example displays:
- ILI9341-based displays
- ST7789-based displays
- Most color TFT displays

### When to Use I2C
- For small monochrome displays
- When connecting multiple devices
- When pin count is extremely limited
- For touch controllers and auxiliary devices

Example displays:
- SSD1306 OLED displays
- Touch controllers (FT6236, GT911)
- Small e-paper displays

### When to Use Parallel
- For large displays requiring fast updates
- When maximum performance is needed
- When hardware supports parallel interface
- For displays with parallel-only interfaces

Example displays:
- HX8357-based displays
- Large TFT displays
- High-resolution panels

## Performance Considerations

### SPI Performance
```cpp
// High-speed SPI configuration
cfg.spi_mode = 0;             // Mode 0 (most common)
cfg.freq_write = 40000000;    // 40MHz write clock
cfg.freq_read = 16000000;     // 16MHz read clock
cfg.spi_3wire = true;         // Enable 3-wire mode if supported
```

### I2C Performance
```cpp
// Optimized I2C configuration
cfg.i2c_port = 0;             // I2C port number
cfg.freq = 400000;            // 400kHz Fast Mode
cfg.pin_sda = 21;             // SDA pin
cfg.pin_scl = 22;             // SCL pin
```

### Parallel Performance
```cpp
// High-performance parallel configuration
cfg.pin_wr = 4;               // Write strobe pin
cfg.pin_rd = 5;               // Read strobe pin
cfg.pin_rs = 6;               // Register select pin
cfg.freq_write = 20000000;    // 20MHz write clock
cfg.bus_shared = false;       // Dedicated bus mode
```

## Platform-Specific Features

### ESP32 Family
- Hardware SPI with DMA support
- Hardware I2C with clock stretching
- 8-bit and 16-bit parallel interfaces
- Multiple bus configurations

### RP2040 (Raspberry Pi Pico)
- PIO-based SPI implementation
- Hardware I2C support
- Flexible pin mapping
- DMA support for efficient transfers

### Arduino SAMD21/SAMD51
- Hardware SPI support
- I2C with clock stretching
- Limited parallel interface support
- DMA capabilities

## Best Practices

1. Bus Configuration:
   - Use appropriate clock speeds for your display
   - Enable DMA when available
   - Configure proper pin assignments
   - Consider voltage levels and logic compatibility

2. Performance Optimization:
   - Use hardware acceleration when available
   - Enable DMA for large transfers
   - Batch commands when possible
   - Choose appropriate bus width

3. Error Handling:
   - Implement proper initialization checks
   - Handle communication timeouts
   - Verify device responses
   - Implement recovery mechanisms

4. Resource Management:
   - Handle bus sharing properly
   - Implement proper locking mechanisms
   - Clean up resources when not needed
   - Monitor bus usage and timing

## Common Issues and Solutions

1. SPI Communication Issues:
   - Verify clock polarity and phase
   - Check CS timing requirements
   - Ensure proper voltage levels
   - Verify cable lengths and signal integrity

2. I2C Communication Issues:
   - Check device addresses
   - Verify pull-up resistors
   - Handle clock stretching properly
   - Monitor bus capacitance

3. Parallel Bus Issues:
   - Verify data setup and hold times
   - Check signal timing requirements
   - Monitor power supply stability
   - Handle bus contention

The bus interface system provides a flexible foundation for supporting various display types while maintaining consistent behavior across different platforms. 