# Animation System Development Guide

This guide explains how to create a comprehensive animation system in LovyanGFX.

## Animation Framework

### Animation Base Class

```cpp
class Animation {
protected:
    uint32_t start_time;
    uint32_t duration;
    bool is_running;
    bool is_completed;
    float current_progress;
    std::function<void()> completion_callback;
    
public:
    Animation(uint32_t duration_ms)
        : start_time(0)
        , duration(duration_ms)
        , is_running(false)
        , is_completed(false)
        , current_progress(0.0f)
    {}
    
    virtual ~Animation() {}
    
    void start() {
        start_time = millis();
        is_running = true;
        is_completed = false;
        current_progress = 0.0f;
        onStart();
    }
    
    void update() {
        if (!is_running || is_completed) return;
        
        uint32_t current_time = millis();
        uint32_t elapsed = current_time - start_time;
        
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
    virtual void onStart() {}
    virtual void onUpdate(float progress) = 0;
    virtual void onStop() {}
    
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

```cpp
namespace Easing {
    // Linear
    float linear(float t) {
        return t;
    }
    
    // Quadratic
    float easeInQuad(float t) {
        return t * t;
    }
    
    float easeOutQuad(float t) {
        return t * (2.0f - t);
    }
    
    float easeInOutQuad(float t) {
        return t < 0.5f ? 2.0f * t * t : -1.0f + (4.0f - 2.0f * t) * t;
    }
    
    // Cubic
    float easeInCubic(float t) {
        return t * t * t;
    }
    
    float easeOutCubic(float t) {
        float t1 = t - 1.0f;
        return 1.0f + t1 * t1 * t1;
    }
    
    float easeInOutCubic(float t) {
        return t < 0.5f ? 4.0f * t * t * t : 
            (t - 1.0f) * (2.0f * t - 2.0f) * (2.0f * t - 2.0f) + 1.0f;
    }
    
    // Elastic
    float easeInElastic(float t) {
        if (t == 0.0f || t == 1.0f) return t;
        float p = 0.3f;
        float s = p / 4.0f;
        float t1 = t - 1.0f;
        return -(pow(2.0f, 10.0f * t1) * sin((t1 - s) * (2.0f * PI) / p));
    }
    
    float easeOutElastic(float t) {
        if (t == 0.0f || t == 1.0f) return t;
        float p = 0.3f;
        float s = p / 4.0f;
        return pow(2.0f, -10.0f * t) * sin((t - s) * (2.0f * PI) / p) + 1.0f;
    }
    
    // Bounce
    float easeOutBounce(float t) {
        if (t < (1.0f/2.75f)) {
            return 7.5625f * t * t;
        } else if (t < (2.0f/2.75f)) {
            float t1 = t - (1.5f/2.75f);
            return 7.5625f * t1 * t1 + 0.75f;
        } else if (t < (2.5f/2.75f)) {
            float t1 = t - (2.25f/2.75f);
            return 7.5625f * t1 * t1 + 0.9375f;
        } else {
            float t1 = t - (2.625f/2.75f);
            return 7.5625f * t1 * t1 + 0.984375f;
        }
    }
}
```

## Animation Types

### Property Animation

```cpp
template<typename T>
class PropertyAnimation : public Animation {
    T* target_value;
    T start_value;
    T end_value;
    std::function<float(float)> easing_function;
    
public:
    PropertyAnimation(T* target, T end, uint32_t duration,
                     std::function<float(float)> easing = Easing::linear)
        : Animation(duration)
        , target_value(target)
        , end_value(end)
        , easing_function(easing)
    {}
    
protected:
    void onStart() override {
        start_value = *target_value;
    }
    
    void onUpdate(float progress) override {
        float eased_progress = easing_function(progress);
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
    uint16_t start_color;
    uint16_t end_color;
    std::function<void(uint16_t)> update_callback;
    std::function<float(float)> easing_function;
    
public:
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
        
        // Extract RGB components
        uint8_t r1 = (start_color >> 11) & 0x1F;
        uint8_t g1 = (start_color >> 5) & 0x3F;
        uint8_t b1 = start_color & 0x1F;
        
        uint8_t r2 = (end_color >> 11) & 0x1F;
        uint8_t g2 = (end_color >> 5) & 0x3F;
        uint8_t b2 = end_color & 0x1F;
        
        // Interpolate
        uint8_t r = r1 + (r2 - r1) * eased;
        uint8_t g = g1 + (g2 - g1) * eased;
        uint8_t b = b1 + (b2 - b1) * eased;
        
        // Combine
        uint16_t current_color = (r << 11) | (g << 5) | b;
        
        update_callback(current_color);
    }
};
```

### Sequence Animation

```cpp
class SequenceAnimation : public Animation {
    std::vector<Animation*> animations;
    size_t current_index;
    
public:
    SequenceAnimation()
        : Animation(0)  // Duration will be calculated
        , current_index(0)
    {}
    
    void addAnimation(Animation* animation) {
        animations.push_back(animation);
        duration += animation->getDuration();
    }
    
protected:
    void onStart() override {
        current_index = 0;
        if (!animations.empty()) {
            animations[0]->start();
        }
    }
    
    void onUpdate(float progress) override {
        if (current_index >= animations.size()) return;
        
        Animation* current = animations[current_index];
        current->update();
        
        if (current->isCompleted()) {
            current_index++;
            if (current_index < animations.size()) {
                animations[current_index]->start();
            }
        }
    }
    
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
    std::vector<Animation*> animations;
    
public:
    ParallelAnimation(uint32_t duration)
        : Animation(duration)
    {}
    
    void addAnimation(Animation* animation) {
        animations.push_back(animation);
    }
    
protected:
    void onStart() override {
        for (auto anim : animations) {
            anim->start();
        }
    }
    
    void onUpdate(float progress) override {
        bool all_completed = true;
        for (auto anim : animations) {
            anim->update();
            if (!anim->isCompleted()) {
                all_completed = false;
            }
        }
        
        if (all_completed) {
            complete();
        }
    }
    
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