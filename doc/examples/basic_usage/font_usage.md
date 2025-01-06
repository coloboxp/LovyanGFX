# Font Usage Example

This example demonstrates the comprehensive text and font handling capabilities of LovyanGFX, including different font types, text alignment, sizing, and advanced text rendering features.

## Overview

The Font Usage example (`examples/HowToUse/3_fonts/3_fonts.ino`) shows how to:
- Use different font types and styles
- Handle text alignment and positioning
- Control text size and wrapping
- Implement text scrolling and clipping
- Work with multi-language support

## Text Drawing Methods

### Basic Text Functions
```cpp
// Using drawString
lcd.drawString("string!", 10, 10);    // Draw at specific coordinates

// Using drawNumber for integers
lcd.drawNumber(123, 100, 10);         // Draw number at coordinates

// Using drawFloat for decimal numbers
lcd.drawFloat(3.14, 2, 150, 10);      // Draw float with 2 decimal places

// Using print functions
lcd.setCursor(10, 20);                // Set cursor position
lcd.print("print!");                  // Print at cursor position

// Using printf for formatted output
int value = 123;
lcd.printf("test %d", value);

// Using println for automatic line breaks
lcd.println("println");               // Print and move to next line
```

## Font Selection

### Built-in Fonts
```cpp
// Using predefined fonts (TFT_eSPI compatible)
lcd.setFont(&fonts::Font4);           // Using Font4
lcd.setTextFont(2);                   // Alternative method (not recommended)

// Special purpose fonts
lcd.setFont(&fonts::Font6);           // Clock-specific characters
lcd.setFont(&fonts::Font7);           // 7-segment LCD style
lcd.setFont(&fonts::Font8);           // Numbers only
```

### Multi-Language Fonts

#### Japanese Fonts
```cpp
// IPA Fonts (4 types × 9 sizes)
lcd.setFont(&fonts::lgfxJapanMincho_12);      // Mincho 12px fixed-width
lcd.setFont(&fonts::lgfxJapanMinchoP_16);     // Mincho 16px proportional
lcd.setFont(&fonts::lgfxJapanGothic_20);      // Gothic 20px fixed-width
lcd.setFont(&fonts::lgfxJapanGothicP_24);     // Gothic 24px proportional
```

#### Asian Language Support
```cpp
// efont-based fonts (4 types × 5 sizes)
lcd.setFont(&fonts::efontJA_10);              // Japanese 10px
lcd.setFont(&fonts::efontCN_12_b);            // Simplified Chinese 12px bold
lcd.setFont(&fonts::efontTW_14_bi);           // Traditional Chinese 14px bold-italic
lcd.setFont(&fonts::efontKR_16_i);            // Korean 16px italic
```

## Text Styling

### Color Control
```cpp
// Set text and background color
lcd.setTextColor(0x00FFFFU, 0xFF0000U);   // Cyan text on red background

// Text-only color (transparent background)
lcd.setTextColor(0xFFFF00U);              // Yellow text
```

### Text Alignment

```cpp
// Set text alignment datum
lcd.setTextDatum(textdatum_t::top_left);        // Top-left alignment
lcd.setTextDatum(textdatum_t::middle_center);   // Center alignment
lcd.setTextDatum(textdatum_t::bottom_right);    // Bottom-right alignment

// Example of aligned text
lcd.drawString("bottom_right", lcd.width()/2, lcd.height()/2);
```

### Text Size and Scaling
```cpp
// Set text size (uniform scaling)
lcd.setTextSize(2.5);                    // 2.5x scaling both directions

// Set different horizontal and vertical scaling
lcd.setTextSize(2.7, 4);                 // 2.7x horizontal, 4x vertical
```

## Advanced Features

### Text Wrapping
```cpp
// Disable text wrapping
lcd.setTextWrap(false);
lcd.println("Long text without wrapping...");

// Enable horizontal wrapping
lcd.setTextWrap(true);
lcd.println("Text with horizontal wrapping...");

// Enable both horizontal and vertical wrapping
lcd.setTextWrap(true, true);
```

### Text Scrolling
```cpp
// Enable text scrolling
lcd.setTextScroll(true);

// Set scroll region
lcd.setScrollRect(10, 10, width-20, height-20, 0x00001FU);

// Example of scrolling text
for (int i = 0; i < 50; ++i) {
    lcd.printf("scroll test %d \n", i);
}
```

### Clipping and Padding
```cpp
// Set clipping rectangle
lcd.setClipRect(10, 10, width-20, height-20);

// Set text padding
lcd.setTextPadding(100);

// Clear clipping/scroll settings
lcd.clearClipRect();
lcd.clearScrollRect();
```

## Best Practices

1. Font Selection
   - Use built-in fonts for basic Latin text
   - Choose appropriate multi-language fonts for localization
   - Consider memory usage when using large fonts

2. Text Rendering
   - Use appropriate text functions for your needs
   - Set proper text color and background for readability
   - Consider using text padding for consistent layouts

3. Performance
   - Use fixed-width fonts for faster rendering
   - Minimize font changes in performance-critical code
   - Use clipping to optimize rendering in specific areas

## Common Issues

1. Text Alignment
   - Verify datum point settings
   - Check coordinates for aligned text
   - Consider text padding effects

2. Font Display
   - Ensure correct font initialization
   - Verify memory availability for large fonts
   - Check character support for special characters

3. Performance
   - Monitor memory usage with multiple fonts
   - Consider using smaller fonts when possible
   - Use appropriate text wrapping settings

## Additional Resources

- [Font Management Guide](../core_concepts/fonts.md)
- [Text Rendering Guide](../core_concepts/text_rendering.md)
- [Localization Guide](../core_concepts/localization.md)
- [Performance Optimization](../core_concepts/performance.md) 