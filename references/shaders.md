# Shaders — CanvasItem (2D), Spatial (3D) & Practical Effects

## Purpose

Master GLSL shader development in Godot Engine 4.x, including modern uniform syntax, screen texture sampling (`hint_screen_texture`), and production recipes for gameplay effects and visual feedback.

---

## 1. Shader Types & Pipeline Functions

```glsl
shader_type canvas_item; // Or "spatial" for 3D, "particles", "fog", "sky"

// Executed for every vertex in the mesh/sprite
void vertex() {
    // Example: Wind waving displacement
    // VERTEX.x += sin(TIME * 5.0 + VERTEX.y) * 2.0;
}

// Executed for every rendered pixel/fragment
void fragment() {
    // COLOR = ...
}

// Executed for custom lighting calculations
void light() {
    // LIGHT = ...
}
```

---

## 2. Uniforms and Hints in Godot 4

```glsl
uniform vec4 hit_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float progress : hint_range(0.0, 1.0, 0.01) = 0.0;
uniform sampler2D noise_texture : repeat_enable, filter_linear;

// In Godot 4, reading screen pixels requires an explicit uniform with hint_screen_texture
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;
```

---

## 3. Production Shader Recipes

### A. 2D Hit Flash Shader
Flashes a character white/red upon taking damage:

```glsl
shader_type canvas_item;

uniform vec4 flash_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float flash_modifier : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    // Blend texture color with flash color preserving original alpha
    COLOR.rgb = mix(tex_color.rgb, flash_color.rgb, flash_modifier);
    COLOR.a = tex_color.a;
}
```

Controlled via GDScript:
```gdscript
func take_damage() -> void:
    var mat := sprite.material as ShaderMaterial
    var tween := create_tween()
    tween.tween_property(mat, "shader_parameter/flash_modifier", 1.0, 0.05)
    tween.tween_property(mat, "shader_parameter/flash_modifier", 0.0, 0.1)
```

---

### B. 2D Outline Shader
Highlights interactive objects or selected characters:

```glsl
shader_type canvas_item;

uniform vec4 outline_color : source_color = vec4(1.0, 0.8, 0.0, 1.0);
uniform float outline_width : hint_range(0.0, 10.0) = 1.0;

void fragment() {
    vec2 size = TEXTURE_PIXEL_SIZE * outline_width;
    float alpha = texture(TEXTURE, UV).a;
    
    // Sample 4 neighboring pixels (up, down, left, right)
    float outline = texture(TEXTURE, UV + vec2(-size.x, 0.0)).a;
    outline += texture(TEXTURE, UV + vec2(size.x, 0.0)).a;
    outline += texture(TEXTURE, UV + vec2(0.0, -size.y)).a;
    outline += texture(TEXTURE, UV + vec2(0.0, size.y)).a;
    outline = min(outline, 1.0) - alpha;
    
    vec4 tex_color = texture(TEXTURE, UV);
    COLOR = mix(tex_color, outline_color, outline);
}
```

---

### C. Dissolve / Disintegration Shader
For enemy death or boss spawn transitions:

```glsl
shader_type canvas_item;

uniform sampler2D noise_texture : repeat_enable;
uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.0;
uniform vec4 burn_color : source_color = vec4(1.0, 0.3, 0.0, 1.0);
uniform float burn_size : hint_range(0.0, 0.1) = 0.04;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    float noise_val = texture(noise_texture, UV).r;
    
    // Discard fragment if noise is below threshold
    if (noise_val < dissolve_amount) {
        discard;
    }
    
    // Glowing burn edge along cut line
    if (noise_val < dissolve_amount + burn_size) {
        COLOR = burn_color;
    } else {
        COLOR = tex_color;
    }
}
```

---

### D. Screen Vignette (2D/3D Post-Processing)
Applied to a fullscreen `ColorRect` inside a `CanvasLayer`:

```glsl
shader_type canvas_item;

uniform float vignette_intensity : hint_range(0.0, 2.0) = 0.4;
uniform float vignette_opacity : hint_range(0.0, 1.0) = 0.6;
uniform vec4 vignette_rgb : source_color = vec4(0.0, 0.0, 0.0, 1.0);

void fragment() {
    vec2 uv = UV - 0.5;
    float dist = length(uv);
    float vignette = smoothstep(0.8 - vignette_intensity, 0.8, dist);
    COLOR = vec4(vignette_rgb.rgb, vignette * vignette_opacity);
}
```

---

## 4. Best Practices & Performance

1. **Avoid unnecessary `discard` on mobile:** The `discard` instruction disables early-Z optimizations on older mobile GPUs. For isolated 2D sprites, it is safe.
2. **Reuse materials:** When multiple enemies share the same flash shader, share a single `ShaderMaterial` instance or use `set_instance_shader_parameter()` to tweak individual values without breaking draw-call batching.
