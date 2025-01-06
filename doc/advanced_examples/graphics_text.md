# Advanced Graphics and Text Rendering

This guide demonstrates advanced graphics and text rendering techniques using LovyanGFX's sprite system and hardware acceleration features.

## Graphics Buffer Management

```cpp
class GraphicsBuffer {
private:
    LGFX* _display;
    LGFX_Sprite* _buffer;
    bool _use_psram;
    uint8_t _rotation;
    uint8_t _color_depth;

public:
    GraphicsBuffer(LGFX* display, bool use_psram = false, uint8_t color_depth = 16)
    : _display(display)
    , _use_psram(use_psram)
    , _rotation(0)
    , _color_depth(color_depth)
    {
        _buffer = new LGFX_Sprite(display);
        _buffer->setPsram(_use_psram);
        _buffer->setColorDepth(_color_depth);
        _buffer->createSprite(display->width(), display->height());
    }

    void setRotation(uint8_t r) {
        _rotation = r % 8;
        _buffer->setRotation(_rotation);
    }

    void clear(uint32_t color = TFT_BLACK) {
        _buffer->fillScreen(color);
    }

    void render() {
        _buffer->pushSprite(0, 0);
    }

    LGFX_Sprite* buffer() { return _buffer; }
};
```

## Advanced Text Rendering

### Text Effects System

```cpp
class TextRenderer {
private:
    LGFX_Sprite* _target;
    float _text_size;
    uint32_t _text_color;
    const lgfx::GFXfont* _font;

public:
    TextRenderer(LGFX_Sprite* target)
    : _target(target)
    , _text_size(1.0f)
    , _text_color(TFT_WHITE)
    , _font(nullptr)
    {
    }

    void setFont(const lgfx::GFXfont* font) {
        _font = font;
        if (_font) {
            _target->setFont(_font);
        }
    }

    void setTextSize(float size) {
        _text_size = size;
        _target->setTextSize(size);
    }

    void setTextColor(uint32_t color) {
        _text_color = color;
        _target->setTextColor(color);
    }

    void drawTextWithShadow(const char* text, int32_t x, int32_t y, 
                           uint32_t shadow_color = TFT_DARKGREY, 
                           int8_t offset_x = 2, 
                           int8_t offset_y = 2) 
    {
        _target->setTextColor(shadow_color);
        _target->drawString(text, x + offset_x, y + offset_y);
        _target->setTextColor(_text_color);
        _target->drawString(text, x, y);
    }

    void drawTextWithOutline(const char* text, int32_t x, int32_t y,
                            uint32_t outline_color = TFT_BLACK,
                            uint8_t thickness = 1)
    {
        for (int8_t i = -thickness; i <= thickness; i++) {
            for (int8_t j = -thickness; j <= thickness; j++) {
                if (i == 0 && j == 0) continue;
                _target->setTextColor(outline_color);
                _target->drawString(text, x + i, y + j);
            }
        }
        _target->setTextColor(_text_color);
        _target->drawString(text, x, y);
    }

    void drawTextWithGradient(const char* text, int32_t x, int32_t y,
                             uint32_t color1, uint32_t color2)
    {
        int16_t w = _target->textWidth(text);
        int16_t h = _target->fontHeight();
        
        LGFX_Sprite temp(_target);
        temp.createSprite(w, h);
        temp.setFont(_font);
        temp.setTextSize(_text_size);
        
        // Create gradient mask
        temp.fillScreen(TFT_BLACK);
        temp.setTextColor(TFT_WHITE);
        temp.drawString(text, 0, 0);
        
        // Apply gradient
        for (int16_t py = 0; py < h; py++) {
            float progress = (float)py / h;
            uint8_t r = lerp(color1 >> 16, color2 >> 16, progress);
            uint8_t g = lerp((color1 >> 8) & 0xFF, (color2 >> 8) & 0xFF, progress);
            uint8_t b = lerp(color1 & 0xFF, color2 & 0xFF, progress);
            uint32_t color = (r << 16) | (g << 8) | b;
            
            for (int16_t px = 0; px < w; px++) {
                if (temp.readPixel(px, py)) {
                    _target->drawPixel(x + px, y + py, color);
                }
            }
        }
    }

private:
    uint8_t lerp(uint8_t a, uint8_t b, float t) {
        return a + (b - a) * t;
    }
};
```

## Advanced Shape Drawing

### Shape Renderer

```cpp
class ShapeRenderer {
private:
    LGFX_Sprite* _target;

public:
    ShapeRenderer(LGFX_Sprite* target) : _target(target) {}

    void drawRoundedRectWithGradient(int32_t x, int32_t y, int32_t w, int32_t h,
                                    int32_t radius, uint32_t color1, uint32_t color2,
                                    bool vertical = true)
    {
        for (int32_t i = 0; i < (vertical ? h : w); i++) {
            float progress = (float)i / (vertical ? h : w);
            uint8_t r = lerp(color1 >> 16, color2 >> 16, progress);
            uint8_t g = lerp((color1 >> 8) & 0xFF, (color2 >> 8) & 0xFF, progress);
            uint8_t b = lerp(color1 & 0xFF, color2 & 0xFF, progress);
            uint32_t color = (r << 16) | (g << 8) | b;
            
            if (vertical) {
                _target->drawFastHLine(x, y + i, w, color);
            } else {
                _target->drawFastVLine(x + i, y, h, color);
            }
        }
        
        // Draw rounded corners
        drawRoundCorners(x, y, w, h, radius);
    }

    void drawCircleWithPattern(int32_t x0, int32_t y0, int32_t r,
                              uint32_t fg_color, uint32_t bg_color,
                              uint8_t pattern = 0xAA)
    {
        int32_t f = 1 - r;
        int32_t ddF_x = 1;
        int32_t ddF_y = -2 * r;
        int32_t x = 0;
        int32_t y = r;
        
        uint8_t pattern_pos = 0;
        
        auto drawPixelPattern = [&](int32_t x, int32_t y) {
            uint32_t color = (pattern & (1 << pattern_pos)) ? fg_color : bg_color;
            _target->drawPixel(x, y, color);
            pattern_pos = (pattern_pos + 1) & 7;
        };
        
        drawPixelPattern(x0, y0 + r);
        drawPixelPattern(x0, y0 - r);
        drawPixelPattern(x0 + r, y0);
        drawPixelPattern(x0 - r, y0);
        
        while (x < y) {
            if (f >= 0) {
                y--;
                ddF_y += 2;
                f += ddF_y;
            }
            x++;
            ddF_x += 2;
            f += ddF_x;
            
            drawPixelPattern(x0 + x, y0 + y);
            drawPixelPattern(x0 - x, y0 + y);
            drawPixelPattern(x0 + x, y0 - y);
            drawPixelPattern(x0 - x, y0 - y);
            drawPixelPattern(x0 + y, y0 + x);
            drawPixelPattern(x0 - y, y0 + x);
            drawPixelPattern(x0 + y, y0 - x);
            drawPixelPattern(x0 - y, y0 - x);
        }
    }

private:
    void drawRoundCorners(int32_t x, int32_t y, int32_t w, int32_t h, int32_t r) {
        // Implementation of rounded corner drawing
        // This is a placeholder for the actual implementation
    }

    uint8_t lerp(uint8_t a, uint8_t b, float t) {
        return a + (b - a) * t;
    }
};
```

## Usage Examples

### Advanced Text Effects

```cpp
LGFX display;
GraphicsBuffer graphics(&display);
TextRenderer text(graphics.buffer());
ShapeRenderer shapes(graphics.buffer());

void setup() {
    display.init();
    display.setRotation(1);
    display.setBrightness(128);
    
    graphics.clear(TFT_BLACK);
    
    // Draw text with shadow
    text.setTextSize(2.0f);
    text.setTextColor(TFT_WHITE);
    text.drawTextWithShadow("Shadow Text", 50, 50);
    
    // Draw text with outline
    text.setTextSize(3.0f);
    text.drawTextWithOutline("Outline", 50, 100, TFT_RED, 2);
    
    // Draw gradient text
    text.setTextSize(4.0f);
    text.drawTextWithGradient("Gradient", 50, 150, TFT_BLUE, TFT_GREEN);
    
    graphics.render();
}
```

### Complex Graphics

```cpp
void drawComplexGraphics() {
    graphics.clear(TFT_BLACK);
    
    // Draw gradient rounded rectangle
    shapes.drawRoundedRectWithGradient(20, 20, 200, 100, 10,
                                     TFT_BLUE, TFT_PURPLE);
    
    // Draw patterned circle
    shapes.drawCircleWithPattern(120, 160, 50,
                               TFT_WHITE, TFT_DARKGREY);
    
    graphics.render();
}
```

## Performance Optimization

1. Buffer Management
   - Use appropriate color depth
   - Enable PSRAM for large buffers
   - Implement double buffering

2. Drawing Optimization
   - Batch similar operations
   - Use hardware acceleration
   - Minimize redraw areas

3. Memory Considerations
   - Monitor sprite memory usage
   - Clean up unused resources
   - Handle memory constraints

4. Error Handling
   - Validate parameters
   - Handle font loading errors
   - Implement graceful degradation 