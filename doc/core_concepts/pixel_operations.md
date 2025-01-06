# Pixel Operations

LovyanGFX provides efficient pixel data transfer and manipulation through the pixelcopy system. This handles color format conversions and optimized memory operations.

## Pixel Copy Operations

```cpp
namespace lgfx {
    // Basic pixel copy with format conversion
    void pixelcopy(void* dst, const void* src,
                  uint32_t length,
                  color_depth_t dst_depth,
                  color_depth_t src_depth);

    // Copy with transparency
    void pixelcopy(void* dst, const void* src,
                  uint32_t length,
                  color_depth_t dst_depth,
                  color_depth_t src_depth,
                  uint32_t transparent_color);
}
```

## Color Format Conversion

```cpp
// Convert between color formats
struct pixel_format_converter {
    // RGB565 to RGB888
    static void rgb565_to_rgb888(void* dst, const void* src, uint32_t length) {
        auto s = static_cast<const uint16_t*>(src);
        auto d = static_cast<uint8_t*>(dst);
        do {
            uint_fast16_t rgb = *s++;
            *d++ = (rgb >> 8) & 0xF8;  // Red
            *d++ = (rgb >> 3) & 0xFC;  // Green
            *d++ = (rgb << 3) & 0xF8;  // Blue
        } while (--length);
    }
    
    // RGB888 to RGB565
    static void rgb888_to_rgb565(void* dst, const void* src, uint32_t length) {
        auto s = static_cast<const uint8_t*>(src);
        auto d = static_cast<uint16_t*>(dst);
        do {
            uint_fast8_t r = *s++;
            uint_fast8_t g = *s++;
            uint_fast8_t b = *s++;
            *d++ = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
        } while (--length);
    }
};
```

## Hardware Acceleration

```cpp
// DMA transfer setup
struct pixelcopy_t {
    union {
        uint32_t positions[4];
        struct {
            uint32_t src_x;
            uint32_t src_y;
            uint32_t src_w;
            uint32_t src_h;
        };
    };
    
    color_depth_t src_depth;
    color_depth_t dst_depth;
    uint32_t src_stride;
    uint32_t dst_stride;
    void* src_data;
    void* dst_data;
    
    // Configure DMA transfer
    void setup_dma_transfer(void) {
        // Hardware-specific DMA setup
    }
};
```

## Memory Management

```cpp
// Optimized memory operations
struct memory_copy {
    // Fast aligned copy
    static void memcpy_aligned(void* dst, const void* src, size_t len) {
        auto d = static_cast<uint32_t*>(dst);
        auto s = static_cast<const uint32_t*>(src);
        len >>= 2;
        do {
            *d++ = *s++;
        } while (--len);
    }
    
    // Unaligned copy
    static void memcpy_unaligned(void* dst, const void* src, size_t len) {
        auto d = static_cast<uint8_t*>(dst);
        auto s = static_cast<const uint8_t*>(src);
        do {
            *d++ = *s++;
        } while (--len);
    }
};
```

## Performance Optimization

1. Use aligned memory access:
```cpp
// Ensure 32-bit alignment
uint32_t* buffer = (uint32_t*)heap_caps_malloc(size, MALLOC_CAP_32BIT);
```

2. Batch operations:
```cpp
// Multiple pixel copies in one transaction
startWrite();
pixelcopy(dst_buf, src_buf, length, dst_depth, src_depth);
endWrite();
```

3. Use DMA when available:
```cpp
// Configure DMA transfer
pixelcopy_t pc;
pc.src_data = src_buffer;
pc.dst_data = dst_buffer;
pc.setup_dma_transfer();
``` 