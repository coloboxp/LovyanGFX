# Event Handling

This document details the event handling system in LovyanGFX.

## Touch Events

### Touch Event Structure

```cpp
struct touch_point_t {
    int32_t x;          // X coordinate
    int32_t y;          // Y coordinate
    int32_t id;         // Touch point ID
    int32_t size;       // Touch point size
    int32_t pressure;   // Touch pressure
};
```

### Touch Event Methods

```cpp
bool getTouchRaw(touch_point_t* tp, uint_fast8_t count = 1)
```
Gets raw touch data for specified number of touch points.
- Parameters:
  - tp: Pointer to touch_point_t array
  - count: Number of touch points to read
- Returns: true if touch detected
- Throws: None

```cpp
void setTouchCalibrate(int32_t x_min, int32_t x_max, int32_t y_min, int32_t y_max)
```
Sets touch calibration parameters.
- Parameters:
  - x_min: Minimum X coordinate
  - x_max: Maximum X coordinate
  - y_min: Minimum Y coordinate
  - y_max: Maximum Y coordinate
- Returns: void
- Throws: None

## Button Events

### Button Event Structure

```cpp
struct button_state_t {
    bool pressed;       // Current press state
    bool clicked;       // Click event occurred
    bool held;          // Button held
    bool released;      // Button released
    uint32_t duration;  // Press duration
};
```

### Button Event Methods

```cpp
bool isPressed()
```
Checks if button is currently pressed.
- Returns: true if pressed
- Throws: None

```cpp
bool justPressed()
```
Checks if button was just pressed.
- Returns: true if just pressed
- Throws: None

```cpp
bool justReleased()
```
Checks if button was just released.
- Returns: true if just released
- Throws: None

## Display Events

### Display Event Structure

```cpp
struct display_event_t {
    uint8_t type;       // Event type
    uint16_t param1;    // First parameter
    uint16_t param2;    // Second parameter
    void* data;         // Additional data
};
```

### Display Event Methods

```cpp
void setEventCallback(void (*callback)(display_event_t*))
```
Sets callback function for display events.
- Parameters:
  - callback: Pointer to callback function
- Returns: void
- Throws: None

```cpp
void dispatchEvent(display_event_t* event)
```
Dispatches a display event.
- Parameters:
  - event: Pointer to event structure
- Returns: void
- Throws: None

## Input Events

### Input Event Structure

```cpp
struct input_event_t {
    uint8_t type;       // Event type
    uint16_t code;      // Event code
    int32_t value;      // Event value
    uint32_t timestamp; // Event timestamp
};
```

### Input Event Methods

```cpp
bool pollEvent(input_event_t* event)
```
Polls for pending input events.
- Parameters:
  - event: Pointer to event structure
- Returns: true if event available
- Throws: None

```cpp
void flushEvents()
```
Flushes all pending events.
- Returns: void
- Throws: None

## Event Handling Examples

### Touch Event Handling

```cpp
void handleTouchEvents() {
    touch_point_t tp;
    if (display.getTouchRaw(&tp)) {
        // Handle touch event
        handleTouch(tp.x, tp.y);
    }
}

void handleMultiTouch() {
    static constexpr size_t MAX_TOUCHES = 5;
    touch_point_t tp[MAX_TOUCHES];
    
    size_t count = display.getTouchRaw(tp, MAX_TOUCHES);
    for (size_t i = 0; i < count; ++i) {
        handleTouch(tp[i].x, tp[i].y);
    }
}
```

### Button Event Handling

```cpp
void handleButtonEvents() {
    if (button.justPressed()) {
        // Handle button press
        handlePress();
    }
    
    if (button.justReleased()) {
        // Handle button release
        handleRelease();
    }
    
    if (button.isPressed()) {
        // Handle continuous press
        handleHold();
    }
}
```

### Display Event Handling

```cpp
void displayEventCallback(display_event_t* event) {
    switch (event->type) {
        case DISPLAY_EVENT_VSYNC:
            handleVSync();
            break;
            
        case DISPLAY_EVENT_REFRESH:
            handleRefresh();
            break;
            
        case DISPLAY_EVENT_ERROR:
            handleError(event->param1);
            break;
    }
}

void setupDisplayEvents() {
    display.setEventCallback(displayEventCallback);
}
```

### Input Event Handling

```cpp
void handleInputEvents() {
    input_event_t event;
    while (display.pollEvent(&event)) {
        switch (event.type) {
            case INPUT_TYPE_TOUCH:
                handleTouchInput(event);
                break;
                
            case INPUT_TYPE_BUTTON:
                handleButtonInput(event);
                break;
                
            case INPUT_TYPE_GESTURE:
                handleGestureInput(event);
                break;
        }
    }
}
```

## Event Management

### Event Queue

```cpp
class EventQueue {
    static constexpr size_t QUEUE_SIZE = 32;
    display_event_t events[QUEUE_SIZE];
    size_t head;
    size_t tail;
    
public:
    void push(const display_event_t& event) {
        events[tail] = event;
        tail = (tail + 1) % QUEUE_SIZE;
    }
    
    bool pop(display_event_t* event) {
        if (head == tail) return false;
        *event = events[head];
        head = (head + 1) % QUEUE_SIZE;
        return true;
    }
};
```

### Event Dispatcher

```cpp
class EventDispatcher {
    std::vector<EventHandler*> handlers;
    
public:
    void addHandler(EventHandler* handler) {
        handlers.push_back(handler);
    }
    
    void dispatch(const display_event_t& event) {
        for (auto handler : handlers) {
            handler->handleEvent(event);
        }
    }
};
```

## Best Practices

1. **Event Processing**
   - Process events in a timely manner
   - Handle events in priority order
   - Avoid blocking in event handlers
   - Keep handlers lightweight

2. **Touch Handling**
   - Implement debouncing
   - Handle multi-touch properly
   - Calibrate touch input
   - Validate coordinates

3. **Button Handling**
   - Implement debouncing
   - Handle long press events
   - Track press duration
   - Validate state changes

4. **Error Handling**
   - Handle all event types
   - Validate event data
   - Implement timeouts
   - Log error events

## Common Issues and Solutions

1. **Touch Issues**
   ```cpp
   // Implement touch debouncing
   class TouchDebouncer {
       touch_point_t last_point;
       uint32_t last_time;
       static constexpr uint32_t DEBOUNCE_MS = 50;
       
   public:
       bool process(touch_point_t* point) {
           uint32_t now = millis();
           if (now - last_time < DEBOUNCE_MS) {
               return false;
           }
           last_point = *point;
           last_time = now;
           return true;
       }
   };
   ```

2. **Button Issues**
   ```cpp
   // Handle button debouncing
   class ButtonDebouncer {
       bool last_state;
       uint32_t last_time;
       static constexpr uint32_t DEBOUNCE_MS = 50;
       
   public:
       bool process(bool current_state) {
           uint32_t now = millis();
           if (now - last_time < DEBOUNCE_MS) {
               return last_state;
           }
           last_state = current_state;
           last_time = now;
           return current_state;
       }
   };
   ```

3. **Event Queue Issues**
   ```cpp
   // Handle queue overflow
   class SafeEventQueue {
       EventQueue queue;
       uint32_t overflow_count;
       
   public:
       void push(const display_event_t& event) {
           if (!queue.push(event)) {
               overflow_count++;
               handleOverflow();
           }
       }
       
       void handleOverflow() {
           // Log overflow
           // Take corrective action
       }
   };
   ```

4. **Event Processing Issues**
   ```cpp
   // Implement event timeout
   class EventProcessor {
       uint32_t last_process_time;
       static constexpr uint32_t TIMEOUT_MS = 1000;
       
   public:
       void process() {
           uint32_t start = millis();
           display_event_t event;
           
           while (display.pollEvent(&event)) {
               if (millis() - start > TIMEOUT_MS) {
                   // Timeout occurred
                   break;
               }
               handleEvent(event);
           }
       }
   };
   ``` 