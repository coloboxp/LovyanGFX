# File Operations Guide

This guide covers file operations in LovyanGFX, including image loading, saving, and filesystem support.

## Basic File Operations

### Filesystem Setup

```cpp
#include <LovyanGFX.hpp>
#include <SPIFFS.h>  // For ESP32
// Or
#include <SD.h>      // For SD card support

void setupFilesystem() {
    // Initialize SPIFFS
    if (!SPIFFS.begin(true)) {
        Serial.println("SPIFFS initialization failed!");
        return;
    }
    
    // Or initialize SD card
    if (!SD.begin()) {
        Serial.println("SD card initialization failed!");
        return;
    }
}
```

## Image Loading

### Loading BMP Images

```cpp
void loadBMP(const char* filename) {
    // Load BMP from SPIFFS
    lcd.drawBmpFile(SPIFFS, filename, 0, 0);
    
    // Load BMP with custom position and scale
    lcd.drawBmpFile(SPIFFS, filename, x, y, maxWidth, maxHeight);
    
    // Load BMP with transparency
    lcd.drawBmpFile(SPIFFS, filename, x, y, maxWidth, maxHeight, 0, 0, transparent_color);
}
```

### Loading JPEG Images

```cpp
void loadJPEG(const char* filename) {
    // Load JPEG from SPIFFS
    lcd.drawJpgFile(SPIFFS, filename, 0, 0);
    
    // Load JPEG with custom position and scale
    lcd.drawJpgFile(SPIFFS, filename, x, y, maxWidth, maxHeight);
    
    // Load JPEG from SD card
    lcd.drawJpgFile(SD, "/images/photo.jpg", 0, 0);
}
```

### Loading PNG Images

```cpp
void loadPNG(const char* filename) {
    // Load PNG from SPIFFS
    lcd.drawPngFile(SPIFFS, filename, 0, 0);
    
    // Load PNG with transparency
    lcd.drawPngFile(SPIFFS, filename, x, y, maxWidth, maxHeight, 0, 0, transparent_color);
    
    // Load PNG from SD card with scaling
    lcd.drawPngFile(SD, "/images/icon.png", x, y, maxWidth, maxHeight);
}
```

## Image Saving

### Saving Screenshots

```cpp
void saveScreenshot() {
    // Save entire screen as BMP
    lcd.saveBmp(SD, "/screenshot.bmp");
    
    // Save specific region as BMP
    lcd.saveBmp(SD, "/region.bmp", x, y, width, height);
    
    // Save sprite as BMP
    sprite.saveBmp(SD, "/sprite.bmp");
}
```

## HTTP Image Loading

### Loading Images from Web

```cpp
void loadWebImage(const char* url) {
    // Load image from URL
    lcd.drawJpgUrl(url, x, y);
    
    // Load image with custom buffer
    uint8_t* buf = (uint8_t*)malloc(1024);
    lcd.drawJpgUrl(url, x, y, maxWidth, maxHeight, buf, 1024);
    free(buf);
}
```

## Advanced File Operations

### Custom File Handlers

```cpp
class CustomFileHandler : public lgfx::FileWrapper {
public:
    bool open(const char* path, const char* mode) override {
        // Custom file opening logic
        return true;
    }
    
    int read(uint8_t* buf, uint32_t len) override {
        // Custom read logic
        return len;
    }
    
    int write(const uint8_t* buf, uint32_t len) override {
        // Custom write logic
        return len;
    }
    
    void close(void) override {
        // Custom close logic
    }
};
```

### Memory-Mapped Files

```cpp
void useMemoryMappedFile() {
    // Create memory buffer
    static uint8_t buffer[1024];
    
    // Create memory file wrapper
    lgfx::MemFileWrapper memfile;
    memfile.setBuffer(buffer, sizeof(buffer));
    
    // Use memory file
    lcd.drawBmpFile(&memfile, 0, 0);
}
```

## Resource Management

### File Cache System

```cpp
class ImageCache {
    struct CacheEntry {
        LGFX_Sprite* sprite;
        uint32_t last_access;
        String filename;
    };
    
    std::vector<CacheEntry> cache;
    const size_t max_cache_size = 5;
    
public:
    LGFX_Sprite* getImage(const char* filename) {
        // Check cache
        for (auto& entry : cache) {
            if (entry.filename == filename) {
                entry.last_access = millis();
                return entry.sprite;
            }
        }
        
        // Load new image
        if (cache.size() >= max_cache_size) {
            removeOldestEntry();
        }
        
        LGFX_Sprite* sprite = new LGFX_Sprite(&lcd);
        // Load image into sprite
        // ...
        
        cache.push_back({sprite, millis(), filename});
        return sprite;
    }
    
private:
    void removeOldestEntry() {
        if (cache.empty()) return;
        
        auto oldest = cache.begin();
        for (auto it = cache.begin(); it != cache.end(); ++it) {
            if (it->last_access < oldest->last_access) {
                oldest = it;
            }
        }
        
        delete oldest->sprite;
        cache.erase(oldest);
    }
};
```

## Error Handling

### File Operation Error Handling

```cpp
bool loadImage(const char* filename, const char* filesystem) {
    if (!filesystem) {
        Serial.println("Filesystem not initialized!");
        return false;
    }
    
    if (!lcd.drawJpgFile(filesystem, filename, 0, 0)) {
        Serial.printf("Failed to load image: %s\n", filename);
        return false;
    }
    
    return true;
}

void handleFileErrors() {
    if (!SPIFFS.exists("/image.jpg")) {
        Serial.println("File not found!");
        return;
    }
    
    File file = SPIFFS.open("/image.jpg", "r");
    if (!file) {
        Serial.println("Failed to open file!");
        return;
    }
    
    if (file.size() > availableMemory()) {
        Serial.println("File too large!");
        file.close();
        return;
    }
}
```

## Best Practices

1. **File Access**
   - Check file existence before access
   - Handle file open/close properly
   - Use appropriate buffer sizes
   - Implement error handling

2. **Memory Management**
   - Use appropriate buffer sizes
   - Implement caching when needed
   - Clean up resources properly
   - Monitor memory usage

3. **Performance**
   - Cache frequently used images
   - Use appropriate image formats
   - Implement progressive loading
   - Optimize file access patterns

4. **Error Handling**
   - Check filesystem initialization
   - Validate file operations
   - Handle memory allocation
   - Implement timeout handling

## Common Issues and Solutions

1. **Memory Issues**
   ```cpp
   // Handle large files
   void loadLargeImage(const char* filename) {
       File file = SPIFFS.open(filename, "r");
       if (!file) return;
       
       const size_t bufferSize = 1024;
       uint8_t buffer[bufferSize];
       
       while (file.available()) {
           size_t bytesRead = file.read(buffer, bufferSize);
           // Process buffer
       }
       
       file.close();
   }
   ```

2. **File Access Errors**
   ```cpp
   // Implement retry mechanism
   bool loadImageWithRetry(const char* filename, int maxRetries = 3) {
       for (int i = 0; i < maxRetries; i++) {
           if (loadImage(filename)) {
               return true;
           }
           delay(100);  // Wait before retry
       }
       return false;
   }
   ```

3. **Format Compatibility**
   ```cpp
   // Check file format
   bool isImageSupported(const char* filename) {
       String fname = String(filename);
       fname.toLowerCase();
       
       return fname.endsWith(".jpg") ||
              fname.endsWith(".png") ||
              fname.endsWith(".bmp");
   }
   ``` 