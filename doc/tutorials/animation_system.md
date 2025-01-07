# Animation System Development Guide

This guide explains how to create a comprehensive animation system in LovyanGFX.

## Animation Framework

### Animation Base Class

```cpp
class Animation {
protected:
    uint32_t start_time;      // Timestamp when animation started
    uint32_t duration;        // Total duration in milliseconds
    bool is_running;          // Current running state
    bool is_completed;        // Whether animation has finished
    float current_progress;   // Progress from 0.0 to 1.0
    std::function<void()> completion_callback;
    
public:
    // Constructor initializes animation with a specified duration
    Animation(uint32_t duration_ms)
        : start_time(0)
        , duration(duration_ms)
        , is_running(false)
        , is_completed(false)
        , current_progress(0.0f)
    {}
    
    virtual ~Animation() {}
    
    // Starts the animation by recording start time and resetting states
    void start() {
        start_time = millis();
        is_running = true;
        is_completed = false;
        current_progress = 0.0f;
        onStart();
    }
    
    // Updates animation progress based on elapsed time
    // Calculates normalized progress (0.0 to 1.0) and triggers update callback
    void update() {
        if (!is_running || is_completed) return;
        
        uint32_t current_time = millis();
        uint32_t elapsed = current_time - start_time;
        
        // Normalize progress to 0.0-1.0 range, clamped at 1.0
        current_progress = min(1.0f, (float)elapsed / duration);
        
        onUpdate(current_progress);
        
        if (current_progress >= 1.0f) {
            complete();
        }
    }
    
    void stop() {
        is_running = false;
        onStop();
    }
    
    void setCompletionCallback(std::function<void()> callback) {
        completion_callback = callback;
    }
    
    bool isRunning() const { return is_running; }
    bool isCompleted() const { return is_completed; }
    float getProgress() const { return current_progress; }
    
protected:
    // Virtual methods for derived classes to implement animation behavior
    virtual void onStart() {}                     // Called when animation starts
    virtual void onUpdate(float progress) = 0;    // Called each frame with progress (0.0-1.0)
    virtual void onStop() {}                      // Called when animation is stopped
    
    // Handles animation completion and triggers callback
    void complete() {
        is_running = false;
        is_completed = true;
        if (completion_callback) {
            completion_callback();
        }
    }
};
```

### Easing Functions

Easing functions transform linear time (0.0 to 1.0) into non-linear progress to create natural-looking animations.

```cpp
namespace Easing {
    // Linear - no easing, constant speed
    float linear(float t) {
        return t;  // Direct mapping of input to output
    }
    
    // Quadratic easing functions
    float easeInQuad(float t) {
        return t * t;  // Accelerating from zero velocity using t²
    }
    
    float easeOutQuad(float t) {
        // Decelerating to zero velocity using inverted quadratic
        // Formula: -t² + 2t
        return t * (2.0f - t);
    }
    
    float easeInOutQuad(float t) {
        // Acceleration until halfway, then deceleration
        // Uses different formulas for each half:
        // First half:  2t²
        // Second half: -2t² + 4t - 1
        return t < 0.5f ? 
            2.0f * t * t : // First half
            -1.0f + (4.0f - 2.0f * t) * t; // Second half
    }
    
    // Elastic easing - simulates spring/elastic motion
    float easeOutElastic(float t) {
        if (t == 0.0f || t == 1.0f) return t;

        // Constants control the oscillation:
        float p = 0.3f;      // Period - lower = more oscillations
        float s = p / 4.0f;  // Calculate overshoot based on period
        
        // Formula creates dampened sine wave:
        // 2^(-10t) * sin((t-s) * 2π/p) + 1
        // - 2^(-10t): Creates exponential decay
        // - sin(...): Creates oscillation
        // - +1: Brings end point to 1.0
        return pow(2.0f, -10.0f * t) * 
               sin((t - s) * (2.0f * PI) / p) + 1.0f;
    }
    
    // Bounce easing - simulates bouncing motion
    float easeOutBounce(float t) {
        // Divides motion into 4 parts with decreasing bounce heights
        // Each section uses a quadratic curve (n * t²)
        // where n controls the bounce height
        
        if (t < (1.0f/2.75f)) {
            // First bounce - 100% height
            return 7.5625f * t * t;
        } 
        else if (t < (2.0f/2.75f)) {
            // Second bounce - 75% height
            float t1 = t - (1.5f/2.75f);
            return 7.5625f * t1 * t1 + 0.75f;
        } 
        else if (t < (2.5f/2.75f)) {
            // Third bounce - 50% height
            float t1 = t - (2.25f/2.75f);
            return 7.5625f * t1 * t1 + 0.9375f;
        } 
        else {
            // Fourth bounce - 25% height
            float t1 = t - (2.625f/2.75f);
            return 7.5625f * t1 * t1 + 0.984375f;
        }
    }

    // Back easing - slight overshoot
    float easeOutBack(float t) {
        const float c1 = 1.70158f; // Controls overshoot amount
        const float c3 = c1 + 1.0f;
        
        // Formula: 1 + c3 * (t - 1)³ + c1 * (t - 1)²
        // Creates overshoot by going past 1.0 then returning
        float t1 = t - 1.0f;
        return 1.0f + c3 * t1 * t1 * t1 + c1 * t1 * t1;
    }
};
```

### Understanding Easing Functions

1. **Input/Output Range**
   - Input (t) is always 0.0 to 1.0
   - Output should also be 0.0 to 1.0
   - Function must return input value when t = 0 or 1

2. **Common Patterns**
   - **EaseIn**: Start slow, end fast (acceleration)
   - **EaseOut**: Start fast, end slow (deceleration)
   - **EaseInOut**: Combine both for smooth start/end

3. **Physics-Based Easings**
   - **Elastic**: Simulates spring motion using damped sine waves
   - **Bounce**: Simulates bouncing using segmented quadratic curves
   - **Back**: Creates overshoot effect using cubic functions

### Easing Function Visualization

```
Linear:     ─────────
Quad In:    ╭────────
Quad Out:   ────────╯
InOut:      ╭──────╯
Elastic:    ────╮~─╯
Bounce:     ────╰╮╰╮╰
Back:       ────────╮╯
```

## Animation Types

### Property Animation

```cpp
template<typename T>
class PropertyAnimation : public Animation {
    T* target_value;         // Pointer to value being animated
    T start_value;          // Initial value when animation starts
    T end_value;            // Target value to reach
    std::function<float(float)> easing_function;
    
public:
    // Constructor takes pointer to target value, end value, duration and easing function
    PropertyAnimation(T* target, T end, uint32_t duration,
                     std::function<float(float)> easing = Easing::linear)
        : Animation(duration)
        , target_value(target)
        , end_value(end)
        , easing_function(easing)
    {}
    
protected:
    // Store initial value when animation starts
    void onStart() override {
        start_value = *target_value;
    }
    
    // Interpolate between start and end values using easing function
    void onUpdate(float progress) override {
        float eased_progress = easing_function(progress);
        // Linear interpolation: start + (end - start) * progress
        *target_value = start_value + (end_value - start_value) * eased_progress;
    }
};

// Usage example:
float opacity = 0.0f;
auto fade_in = new PropertyAnimation<float>(
    &opacity, 1.0f, 1000, Easing::easeInOutQuad
);
```

### Sprite Animation

```cpp
class SpriteAnimation : public Animation {
    LGFX_Sprite* sprite;
    float start_x, start_y;
    float end_x, end_y;
    float start_scale, end_scale;
    float start_rotation, end_rotation;
    std::function<float(float)> easing_function;
    
public:
    SpriteAnimation(LGFX_Sprite* spr, 
                   float ex, float ey,
                   float end_scale = 1.0f,
                   float end_rotation = 0.0f,
                   uint32_t duration = 1000,
                   std::function<float(float)> easing = Easing::linear)
        : Animation(duration)
        , sprite(spr)
        , end_x(ex), end_y(ey)
        , end_scale(end_scale)
        , end_rotation(end_rotation)
        , easing_function(easing)
    {}
    
protected:
    void onStart() override {
        start_x = sprite->getX();
        start_y = sprite->getY();
        start_scale = sprite->getScale();
        start_rotation = sprite->getRotation();
    }
    
    void onUpdate(float progress) override {
        float eased = easing_function(progress);
        
        float current_x = start_x + (end_x - start_x) * eased;
        float current_y = start_y + (end_y - start_y) * eased;
        float current_scale = start_scale + (end_scale - start_scale) * eased;
        float current_rotation = start_rotation + 
            (end_rotation - start_rotation) * eased;
        
        sprite->setPosition(current_x, current_y);
        sprite->setScale(current_scale);
        sprite->setRotation(current_rotation);
    }
};
```

### Color Animation

```cpp
class ColorAnimation : public Animation {
    uint16_t start_color;    // Initial RGB565 color
    uint16_t end_color;      // Target RGB565 color
    std::function<void(uint16_t)> update_callback;  // Called when color changes
    std::function<float(float)> easing_function;
    
public:
    // Constructor takes start/end colors, callback for updates, duration and easing
    ColorAnimation(uint16_t from, uint16_t to,
                  std::function<void(uint16_t)> callback,
                  uint32_t duration = 1000,
                  std::function<float(float)> easing = Easing::linear)
        : Animation(duration)
        , start_color(from)
        , end_color(to)
        , update_callback(callback)
        , easing_function(easing)
    {}
    
protected:
    void onUpdate(float progress) override {
        float eased = easing_function(progress);
        
        // Extract RGB components from 16-bit RGB565 format
        // RGB565: RRRRRGGGGGGBBBBB (5 bits R, 6 bits G, 5 bits B)
        uint8_t r1 = (start_color >> 11) & 0x1F;  // Get top 5 bits
        uint8_t g1 = (start_color >> 5) & 0x3F;   // Get middle 6 bits
        uint8_t b1 = start_color & 0x1F;          // Get bottom 5 bits
        
        uint8_t r2 = (end_color >> 11) & 0x1F;
        uint8_t g2 = (end_color >> 5) & 0x3F;
        uint8_t b2 = end_color & 0x1F;
        
        // Interpolate each color component separately
        uint8_t r = r1 + (r2 - r1) * eased;
        uint8_t g = g1 + (g2 - g1) * eased;
        uint8_t b = b1 + (b2 - b1) * eased;
        
        // Combine back into RGB565 format
        uint16_t current_color = (r << 11) | (g << 5) | b;
        
        update_callback(current_color);
    }
};
```

### Sequence Animation

```cpp
class SequenceAnimation : public Animation {
    std::vector<Animation*> animations;  // List of animations to play in sequence
    size_t current_index;               // Index of currently playing animation
    
public:
    // Constructor - duration will be sum of all added animations
    SequenceAnimation()
        : Animation(0)
        , current_index(0)
    {}
    
    // Add animation to sequence and update total duration
    void addAnimation(Animation* animation) {
        animations.push_back(animation);
        duration += animation->getDuration();
    }
    
protected:
    // Start first animation in sequence
    void onStart() override {
        current_index = 0;
        if (!animations.empty()) {
            animations[0]->start();
        }
    }
    
    // Update current animation and advance to next when complete
    void onUpdate(float progress) override {
        if (current_index >= animations.size()) return;
        
        Animation* current = animations[current_index];
        current->update();
        
        // When current animation completes, start the next one
        if (current->isCompleted()) {
            current_index++;
            if (current_index < animations.size()) {
                animations[current_index]->start();
            }
        }
    }
    
    // Stop current animation if sequence is interrupted
    void onStop() override {
        if (current_index < animations.size()) {
            animations[current_index]->stop();
        }
    }
};
```

### Parallel Animation

```cpp
class ParallelAnimation : public Animation {
    std::vector<Animation*> animations;  // List of animations to play simultaneously
    
public:
    // Constructor takes total duration for all parallel animations
    ParallelAnimation(uint32_t duration)
        : Animation(duration)
    {}
    
    // Add animation to parallel group
    void addAnimation(Animation* animation) {
        animations.push_back(animation);
    }
    
protected:
    // Start all animations simultaneously
    void onStart() override {
        for (auto anim : animations) {
            anim->start();
        }
    }
    
    // Update all animations and check if all have completed
    void onUpdate(float progress) override {
        bool all_completed = true;
        for (auto anim : animations) {
            anim->update();
            if (!anim->isCompleted()) {
                all_completed = false;
            }
        }
        
        // Complete parallel animation when all children are done
        if (all_completed) {
            complete();
        }
    }
    
    // Stop all animations if interrupted
    void onStop() override {
        for (auto anim : animations) {
            anim->stop();
        }
    }
};
```

## Animation Manager

```cpp
class AnimationManager {
    std::vector<Animation*> active_animations;
    
public:
    void update() {
        for (auto it = active_animations.begin(); 
             it != active_animations.end();) {
            Animation* anim = *it;
            
            if (anim->isRunning()) {
                anim->update();
                ++it;
            } else if (anim->isCompleted()) {
                delete anim;
                it = active_animations.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    void addAnimation(Animation* animation) {
        animation->start();
        active_animations.push_back(animation);
    }
    
    void stopAll() {
        for (auto anim : active_animations) {
            anim->stop();
        }
    }
    
    void clear() {
        for (auto anim : active_animations) {
            delete anim;
        }
        active_animations.clear();
    }
    
    size_t getActiveCount() const {
        return active_animations.size();
    }
};
```

## Usage Examples

### Basic Animation

```cpp
// Create animation manager
AnimationManager animations;

// Create sprite
LGFX_Sprite sprite(&display);
sprite.createSprite(100, 100);
sprite.fillScreen(TFT_RED);

// Create movement animation
auto move = new SpriteAnimation(
    &sprite,
    200, 100,    // End position
    1.5f,        // End scale
    360.0f,      // End rotation
    2000,        // Duration (ms)
    Easing::easeOutElastic
);

// Add to manager
animations.addAnimation(move);

// Update in main loop
void loop() {
    animations.update();
    display.startWrite();
    sprite.pushRotateZoom(
        sprite.getX(), sprite.getY(),
        sprite.getRotation(),
        sprite.getScale(), sprite.getScale()
    );
    display.endWrite();
}
```

### Complex Animation

```cpp
// Create sequence of animations
auto sequence = new SequenceAnimation();

// Fade in
auto fade_in = new PropertyAnimation<uint8_t>(
    &opacity, 255, 500, Easing::easeInQuad
);

// Move and rotate
auto transform = new SpriteAnimation(
    &sprite, 200, 200, 1.0f, 360.0f, 1000, Easing::easeInOutCubic
);

// Color transition
auto color_change = new ColorAnimation(
    TFT_RED, TFT_BLUE,
    [&sprite](uint16_t color) {
        sprite.fillScreen(color);
    },
    800, Easing::linear
);

// Add to sequence
sequence->addAnimation(fade_in);
sequence->addAnimation(transform);
sequence->addAnimation(color_change);

// Add completion callback
sequence->setCompletionCallback([]() {
    Serial.println("Animation sequence completed!");
});

// Start animation
animations.addAnimation(sequence);
```

These examples demonstrate how to create and use animations in LovyanGFX. Adapt them according to your specific needs and requirements. 