# Event System

## Overview

LovyanGFX provides an event handling system for touch input, display updates, and hardware interactions. For implementation examples, see [Touch UI Implementation](../advanced_examples/touch_ui.md).

## Touch Events

The library handles touch input events through the following system. For practical usage, see [Touch Gestures](../tutorials/touch_gestures.md).

```cpp
// Touch event types
enum touch_event_t {
    TOUCH_BEGIN,   // Touch start
    TOUCH_MOVE,    // Touch movement
    TOUCH_END,     // Touch end
    TOUCH_CANCEL   // Touch cancelled
};
```

## Display Events

Events related to display operations. For configuration details, see [Display Configuration](configuration.md#display-configuration).

```cpp
// Display state events
namespace tft_command {
    TFT_DISPOFF = 0x28  // Display OFF event
    TFT_DISPON  = 0x29  // Display ON event
    TFT_SLPIN   = 0x10  // Sleep mode ON event
    TFT_SLPOUT  = 0x11  // Sleep mode OFF event
}
```

## Hardware Events

Events for hardware-specific operations. For platform-specific details, see [Platform-Specific APIs](platform_specific.md).
- DMA transfer completion (see [DMA Operations](../core_concepts/dma_operations.md))
- SPI transaction completion
- Hardware acceleration events (see [Hardware Acceleration](../core_concepts/hardware_acceleration.md))
- Memory management events

## Event Handling

The library provides mechanisms for:
- Event registration
- Callback functions
- Event queuing
- Priority handling

For practical examples, see [Complex UI Development](../tutorials/complex_ui.md). 