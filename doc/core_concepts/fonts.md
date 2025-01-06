# Font Handling and Custom Fonts

This guide explains font handling and custom font implementation in LovyanGFX.

## Built-in Fonts

LovyanGFX includes several built-in fonts:

```cpp
// Default font (no need to set)
display.setFont(&fonts::Font0);  // Basic 6x8 font

// Other built-in fonts
display.setFont(&fonts::Font2);    // 12x16 font
display.setFont(&fonts::Font4);    // 26x32 font
display.setFont(&fonts::Font6);    // 32x48 font
display.setFont(&fonts::Font7);    // 48x64 font
display.setFont(&fonts::Font8);    // 75x100 font
```

## Font Configuration

### Basic Font Settings

```cpp
void configureFonts() {
    // Set font
    display.setFont(&fonts::Font2);
    
    // Set text size (scaling)
    display.setTextSize(2);  // Scale text by 2x
    
    // Set text color
    display.setTextColor(TFT_WHITE);  // Single color
    display.setTextColor(TFT_WHITE, TFT_BLACK);  // Text color and background
    
    // Set text position
    display.setCursor(10, 10);
    
    // Set text alignment
    display.setTextDatum(middle_center);  // Center align
    
    // Set line spacing
    display.setTextLineSpacing(6);  // 6 pixel spacing between lines
}
```

### Text Alignment Options

```cpp
// Text alignment constants
enum text_datum_t {
    top_left,
    top_center,
    top_right,
    middle_left,
    middle_center,
    middle_right,
    bottom_left,
    bottom_center,
    bottom_right,
    baseline_left,
    baseline_center,
    baseline_right
};

void textAlignmentExample() {
    int x = display.width() / 2;
    int y = display.height() / 2;
    
    // Center aligned text
    display.setTextDatum(middle_center);
    display.drawString("Center", x, y);
    
    // Right aligned text
    display.setTextDatum(middle_right);
    display.drawString("Right", x, y + 20);
    
    // Left aligned text
    display.setTextDatum(middle_left);
    display.drawString("Left", x, y - 20);
}
```

## Custom Fonts

### Using AdafruitGFX Fonts

```cpp
#include "Fonts/FreeSans9pt7b.h"
#include "Fonts/FreeSans12pt7b.h"
#include "Fonts/FreeSerif12pt7b.h"

void useAdafruitFonts() {
    // Set custom font
    display.setFont(&FreeSans9pt7b);
    display.println("FreeSans 9pt");
    
    display.setFont(&FreeSans12pt7b);
    display.println("FreeSans 12pt");
    
    display.setFont(&FreeSerif12pt7b);
    display.println("FreeSerif 12pt");
    
    // Return to built-in font
    display.setFont();  // or display.setFont(&fonts::Font0)
}
```

### Creating Custom Fonts

Using the `fontconvert` tool:

```bash
# Convert TTF to Adafruit GFX format
./fontconvert yourfont.ttf 12 > YourFont12pt7b.h
```

Example custom font header:

```cpp
const uint8_t YourFont12pt7bBitmaps[] PROGMEM = {
    // Font bitmap data
};

const GFXglyph YourFont12pt7bGlyphs[] PROGMEM = {
    // Glyph definitions
};

const GFXfont YourFont12pt7b PROGMEM = {
    (uint8_t*)YourFont12pt7bBitmaps,
    (GFXglyph*)YourFont12pt7bGlyphs,
    first_char, last_char,
    yAdvance
};
```

### Unicode Font Support

```cpp
// Include Unicode font
#include "lgfx/Fonts/efont/lgfx_efont_cn.h"
#include "lgfx/Fonts/efont/lgfx_efont_ja.h"
#include "lgfx/Fonts/efont/lgfx_efont_kr.h"

void useUnicodeFonts() {
    // Set Unicode font
    display.setFont(&fonts::efont_cn);  // Chinese font
    display.println("你好");  // Hello in Chinese
    
    display.setFont(&fonts::efont_ja);  // Japanese font
    display.println("こんにちは");  // Hello in Japanese
    
    display.setFont(&fonts::efont_kr);  // Korean font
    display.println("안녕하세요");  // Hello in Korean
}
```

## Font Rendering

### Basic Text Drawing

```cpp
void drawText() {
    // Print text at current cursor
    display.print("Hello");
    display.println(" World!");  // With line break
    
    // Draw text at position
    display.drawString("Positioned", 100, 100);
    
    // Draw formatted text
    display.drawFloat(3.14159, 2, 100, 120);  // 2 decimal places
    display.drawNumber(42, 100, 140);
    
    // Printf style formatting
    display.drawprintf(100, 160, "Value: %d", 42);
}
```

### Advanced Text Effects

```cpp
void textEffects() {
    // Shadow text
    display.setTextColor(TFT_DARKGREY);
    display.drawString("Shadow", 101, 101);
    display.setTextColor(TFT_WHITE);
    display.drawString("Shadow", 100, 100);
    
    // Outline text
    display.setTextColor(TFT_BLACK);
    for (int i = 0; i < 8; i++) {
        float angle = i * PI / 4;
        int dx = cos(angle) * 1;
        int dy = sin(angle) * 1;
        display.drawString("Outline", 100 + dx, 100 + dy);
    }
    display.setTextColor(TFT_WHITE);
    display.drawString("Outline", 100, 100);
    
    // Gradient text
    for (int i = 0; i < 100; i++) {
        uint8_t r = map(i, 0, 100, 255, 0);
        uint8_t b = map(i, 0, 100, 0, 255);
        display.setTextColor(display.color565(r, 0, b));
        display.drawChar('A' + i % 26, i * 6, 100);
    }
}
```

## Performance Optimization

### Memory Usage

```cpp
void optimizeFontMemory() {
    // Use built-in fonts for small text
    display.setFont(&fonts::Font0);  // Minimal memory usage
    
    // Load custom fonts into PROGMEM
    static const uint8_t customFont[] PROGMEM = {
        // Font data
    };
    
    // Use appropriate text color depth
    display.setTextColor(display.color565(255, 255, 255));  // 16-bit color
}
```

### Rendering Speed

```cpp
void optimizeTextRendering() {
    // Batch text operations
    display.startWrite();
    
    for (int i = 0; i < 100; i++) {
        display.drawChar('A' + i % 26, i * 6, 100);
    }
    
    display.endWrite();
    
    // Use hardware acceleration when available
    display.setTextSize(1);  // No scaling for better performance
    
    // Cache commonly used strings
    static const char* cached_text PROGMEM = "Frequently used text";
    display.drawString(cached_text, 100, 100);
}
```

### Font Cache

```cpp
class FontCache {
    static const int CACHE_SIZE = 256;
    struct CacheEntry {
        char16_t code;
        uint16_t width;
        uint8_t* bitmap;
    };
    CacheEntry cache[CACHE_SIZE];
    int cache_index;
    
public:
    FontCache() : cache_index(0) {
        for (int i = 0; i < CACHE_SIZE; i++) {
            cache[i].bitmap = nullptr;
        }
    }
    
    void addToCache(char16_t code, uint16_t width, const uint8_t* bitmap) {
        if (cache[cache_index].bitmap) {
            free(cache[cache_index].bitmap);
        }
        
        cache[cache_index].code = code;
        cache[cache_index].width = width;
        cache[cache_index].bitmap = (uint8_t*)malloc(width * 16);  // Assume 16 pixel height
        memcpy(cache[cache_index].bitmap, bitmap, width * 16);
        
        cache_index = (cache_index + 1) % CACHE_SIZE;
    }
    
    const uint8_t* findInCache(char16_t code, uint16_t& width) {
        for (int i = 0; i < CACHE_SIZE; i++) {
            if (cache[i].code == code) {
                width = cache[i].width;
                return cache[i].bitmap;
            }
        }
        return nullptr;
    }
    
    ~FontCache() {
        for (int i = 0; i < CACHE_SIZE; i++) {
            if (cache[i].bitmap) {
                free(cache[i].bitmap);
            }
        }
    }
};
```

These examples demonstrate various aspects of font handling in LovyanGFX. Adapt them according to your specific display requirements and performance needs. 