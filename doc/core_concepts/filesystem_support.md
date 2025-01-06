# Filesystem Support in LovyanGFX

LovyanGFX provides extensive filesystem support for loading images and fonts. This document explains the filesystem integration capabilities and usage patterns.

## Image Loading Support

The library supports loading various image formats from different sources:

### Supported Image Formats

1. **BMP (Bitmap)**
   ```cpp
   bool drawBmpFile(fs::FS &fs, const char *path, int32_t x = 0, int32_t y = 0);
   bool drawBmp(fs::File *file, int32_t x = 0, int32_t y = 0);
   ```

2. **JPEG**
   ```cpp
   bool drawJpgFile(fs::FS &fs, const char *path, int32_t x = 0, int32_t y = 0);
   bool drawJpg(fs::File *file, int32_t x = 0, int32_t y = 0);
   ```

3. **PNG**
   ```cpp
   bool drawPngFile(fs::FS &fs, const char *path, int32_t x = 0, int32_t y = 0);
   bool drawPng(fs::File *file, int32_t x = 0, int32_t y = 0);
   ```

4. **QOI (Quite OK Image)**
   ```cpp
   bool drawQoiFile(fs::FS &fs, const char *path, int32_t x = 0, int32_t y = 0);
   bool drawQoi(fs::File *file, int32_t x = 0, int32_t y = 0);
   ```

### Common Parameters

All image drawing functions support these parameters:
```cpp
int32_t x = 0;          // X coordinate
int32_t y = 0;          // Y coordinate
int32_t maxWidth = 0;   // Maximum width (0 = no limit)
int32_t maxHeight = 0;  // Maximum height (0 = no limit)
int32_t offX = 0;       // X offset in source image
int32_t offY = 0;       // Y offset in source image
float scale_x = 1.0f;   // X scaling factor
float scale_y = 0.0f;   // Y scaling factor (0 = same as x)
datum_t datum = datum_t::top_left;  // Reference point
```

## Input Sources

### 1. Filesystem (FS)
```cpp
// Using filesystem
LGFX display;
display.drawJpgFile(SD, "/path/to/image.jpg");
```

### 2. File Objects
```cpp
// Using file object
File f = SD.open("/path/to/image.jpg");
display.drawJpg(&f);
```

### 3. Stream Interface
```cpp
// Using stream
Stream* dataStream;  // Any stream source
display.drawPng(dataStream);
```

### 4. HTTP URLs (ESP32)
```cpp
// Loading from URL
display.drawJpgUrl("http://example.com/image.jpg");
```

## Font Support

### Loading Custom Fonts
```cpp
// Load VLW font from filesystem
bool loadFont(const char *path, fs::FS &fs);
```

## Performance Optimization

1. **Memory Usage**
   - Images are loaded and processed in chunks
   - No full image buffer required
   - Streaming support for large files

2. **Scaling Options**
   - Hardware acceleration when available
   - Software scaling for unsupported operations

3. **Best Practices**
   - Use appropriate image format for content type
   - Consider memory constraints when loading large images
   - Use scaling parameters for memory-efficient resizing

## Platform-Specific Features

### Arduino
- Full filesystem support with SD and SPIFFS
- Stream interface support
- HTTP client integration (ESP32)

### ESP32
- Additional HTTP client capabilities
- Integrated WiFi image loading
- SPIFFS and SD card support

### Windows
- Native HTTP client support
- File system integration
- Socket-based networking

## Error Handling

The image loading functions return a boolean indicating success:
```cpp
if (!display.drawJpgFile(SD, "/image.jpg")) {
    // Handle error
}
```

Common error cases:
1. File not found
2. Invalid image format
3. Memory allocation failure
4. Network errors (for HTTP)

## Example Usage

### Basic Image Loading
```cpp
LGFX display;

void setup() {
    display.init();
    SD.begin();
    
    // Load and display image
    display.drawJpgFile(SD, "/images/background.jpg");
    
    // Load with scaling
    display.drawPngFile(SD, "/images/sprite.png", 100, 100, 
                       320, 240,  // max dimensions
                       0, 0,      // source offset
                       2.0f);     // 2x scaling
}
```

### HTTP Image Loading (ESP32)
```cpp
LGFX display;

void setup() {
    display.init();
    WiFi.begin(ssid, password);
    
    // Load image from URL
    display.drawJpgUrl("http://example.com/image.jpg", 
                      0, 0,    // position
                      320, 240 // max dimensions
                     );
} 