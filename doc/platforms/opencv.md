# OpenCV Platform Support

The OpenCV platform support in LovyanGFX enables advanced image processing and computer vision capabilities by integrating with the OpenCV library. This platform is particularly useful for prototyping, testing, and developing applications that require sophisticated image manipulation or computer vision features.

## Architecture Overview

The OpenCV platform implementation consists of several key components:

- `Panel_OpenCV`: Core display panel implementation using OpenCV's Mat
- `cv::Mat` integration for direct image manipulation
- Mouse-based touch input simulation
- Window management and event handling

## Configuration

### Basic Setup

```cpp
#include <LGFX_OpenCV.hpp>

static LGFX display;

void setup() {
    display.init();
    display.setRotation(0);
    display.setColorDepth(color_depth_t::rgb888_3Byte);  // OpenCV uses 24-bit color
}
```

### Display Configuration

The OpenCV platform provides window and display management:

```cpp
// Window creation and management happens automatically
// Each panel instance creates its own window
Panel_OpenCV::imshowall();  // Update all windows
```

## Image Processing Features

### Direct Mat Access

Access the underlying OpenCV Mat for advanced image processing:

```cpp
cv::Mat& getMat() {
    return _cv_mat;  // Access the internal cv::Mat
}

// Example usage
cv::Mat frame = display.getMat();
cv::GaussianBlur(frame, frame, cv::Size(7, 7), 1.5);
```

### Color Space Handling

The OpenCV platform automatically handles color space conversions:

```cpp
// Internal color handling
void display(uint_fast16_t x, uint_fast16_t y, uint_fast16_t w, uint_fast16_t h) {
    cv::Mat mat;
    cv::cvtColor(_cv_mat, mat, cv::COLOR_BGR2RGB);
    cv::imshow(_window_name, mat);
}
```

## Input Simulation

### Mouse Input

The platform simulates touch input through mouse interactions:

- Left mouse button: Single touch point
- Mouse movement: Touch position tracking
- Event callback system for input handling

```cpp
// Mouse callback handling
void cv_mouse_callback(int event, int x, int y, int flags, void* userdata) {
    auto tp = (touch_point_t*)userdata;
    tp->x = x;
    tp->y = y;
    tp->size = (event == cv::EVENT_LBUTTONDOWN) ? 1 : 0;
}
```

## Performance Features

### Hardware Acceleration

The OpenCV platform leverages OpenCV's optimizations:

- Hardware acceleration through OpenCV
- Efficient memory management
- Multi-threaded operation support

### Memory Management

Efficient memory handling through OpenCV's Mat system:

```cpp
bool initFrameBuffer(size_t width, size_t height) {
    _cv_mat = cv::Mat(height, width, CV_8UC3);
    _img = _cv_mat.data;
    return true;
}
```

## Advanced Usage

### Window Management

Control multiple display windows:

```cpp
// Create and manage windows
cv::namedWindow(_window_name, cv::WINDOW_AUTOSIZE);
cv::setMouseCallback(_window_name, cv_mouse_callback, &_touch_point);

// Update all windows
Panel_OpenCV::imshowall();
```

### Image Operations

Perform advanced image operations:

```cpp
// Direct pixel manipulation
void drawPixelPreclipped(uint_fast16_t x, uint_fast16_t y, uint32_t rawcolor) {
    auto img = &((bgr888_t*)_img)[x + y * _cfg.panel_width];
    *img = rawcolor;
}

// Block operations
void writeBlock(uint32_t rawcolor, uint32_t length);
void writeImage(uint_fast16_t x, uint_fast16_t y, uint_fast16_t w, uint_fast16_t h, pixelcopy_t* param);
```

## Development Tools

### Debug Features

The OpenCV platform provides development aids:

- Real-time window visualization
- Mouse event debugging
- Performance monitoring through OpenCV tools

### Integration with OpenCV Tools

Utilize OpenCV's development tools:

```cpp
// Access to OpenCV's debugging features
cv::setTrackbarPos("Debug", _window_name, value);
cv::imwrite("debug_output.png", _cv_mat);
```

## Limitations and Considerations

1. Performance Characteristics
   - OpenCV overhead for simple operations
   - Memory usage differs from embedded targets
   - Window management system dependencies

2. Platform Specifics
   - Requires OpenCV installation
   - System-dependent window behavior
   - Color space conversions overhead

3. Memory Usage
   - Higher memory requirements
   - System memory limitations apply
   - Mat-based memory management

## Best Practices

1. Development Workflow
   - Use for prototyping complex image operations
   - Test performance-critical code on target hardware
   - Leverage OpenCV's debugging tools

2. Performance Optimization
   - Batch operations when possible
   - Use appropriate color depth
   - Monitor memory usage

3. Cross-Platform Compatibility
   - Abstract OpenCV-specific code
   - Handle platform-specific features separately
   - Test on both OpenCV and target platforms

## Example Applications

### Basic Image Processing

```cpp
void processImage() {
    cv::Mat frame = display.getMat();
    
    // Apply OpenCV operations
    cv::GaussianBlur(frame, frame, cv::Size(5,5), 0);
    cv::Canny(frame, frame, 100, 200);
    
    // Display results
    Panel_OpenCV::imshowall();
}
```

### Computer Vision Integration

```cpp
void detectObjects() {
    cv::Mat frame = display.getMat();
    std::vector<cv::Rect> objects;
    
    // Use OpenCV's detection
    cv::CascadeClassifier classifier;
    classifier.detectMultiScale(frame, objects);
    
    // Draw results
    for (const auto& obj : objects) {
        cv::rectangle(frame, obj, cv::Scalar(0,255,0));
    }
    
    Panel_OpenCV::imshowall();
}
``` 