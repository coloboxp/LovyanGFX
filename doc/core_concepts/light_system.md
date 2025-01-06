# Display Backlight Control System

The Light system in LovyanGFX provides an interface for controlling display backlighting. This document explains the core concepts and implementation of the `ILight` interface.

## Overview

The Light system is designed as an abstract interface that allows different display types to implement their own backlight control mechanisms. This abstraction enables consistent backlight control across various hardware platforms while allowing platform-specific optimizations.

## Core Interface

The `ILight` interface defines the basic contract for backlight control:

```cpp
struct ILight {
    virtual ~ILight(void) = default;
    
    // Initialize the backlight with initial brightness
    virtual bool init(uint8_t brightness) = 0;
    
    // Set the backlight brightness level (0-255)
    virtual void setBrightness(uint8_t brightness) = 0;
};
```

## Implementation Guidelines

When implementing the `ILight` interface for a specific display or platform:

1. **Initialization**
   ```cpp
   bool init(uint8_t brightness) {
       // Initialize hardware
       // Set initial brightness
       return true; // Return success/failure
   }
   ```

2. **Brightness Control**
   ```cpp
   void setBrightness(uint8_t brightness) {
       // Convert 8-bit brightness to hardware-specific value
       // Apply brightness to display
   }
   ```

## Example Implementation

Here's a basic example for a PWM-controlled backlight:

```cpp
class PWM_Backlight : public ILight {
private:
    uint8_t _pin;
    uint8_t _channel;

public:
    PWM_Backlight(uint8_t pin, uint8_t channel) 
        : _pin(pin), _channel(channel) {}

    bool init(uint8_t brightness) override {
        // Setup PWM
        pinMode(_pin, OUTPUT);
        // Initialize PWM channel
        setBrightness(brightness);
        return true;
    }

    void setBrightness(uint8_t brightness) override {
        // Apply PWM value
        analogWrite(_pin, brightness);
    }
};
```

## Best Practices

1. **Brightness Range**
   - Input brightness is always 0-255 (8-bit)
   - Implement appropriate scaling for your hardware
   - Handle edge cases (0 = off, 255 = full brightness)

2. **Hardware Considerations**
   - Account for hardware-specific initialization requirements
   - Consider power management implications
   - Implement smooth transitions if hardware supports it

3. **Error Handling**
   - Return meaningful status from init()
   - Gracefully handle hardware failures
   - Validate input parameters

## Performance Considerations

- Minimize operations in setBrightness() as it may be called frequently
- Consider implementing brightness ramping for smooth transitions
- Cache current brightness to avoid unnecessary hardware updates

## Integration Example

```cpp
class MyDisplay : public LGFX {
private:
    PWM_Backlight _backlight;

public:
    MyDisplay() : _backlight(LED_PIN, PWM_CHANNEL) {
        // Display configuration
        panel()->setLight(&_backlight);
    }
};
``` 