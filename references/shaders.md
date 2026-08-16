# Shaders — CanvasItem (2D), Spatial (3D) e Efeitos Práticos

## Propósito

Dominar a escrita de shaders em GLSL para a Godot Engine 4.x, incluindo uniforms modernos, leitura de texturas de tela (`hint_screen_texture`) e receitas práticas para efeitos de gameplay e feedback visual.

---

## 1. Tipos de Shaders e Funções de Pipeline

```glsl
shader_type canvas_item; // Ou "spatial" para 3D, "particles", "fog", "sky"

// Executado para cada vértice da malha/sprite
void vertex() {
    // Exemplo: Deformação por onda de vento
    // VERTEX.x += sin(TIME * 5.0 + VERTEX.y) * 2.0;
}

// Executado para cada pixel renderizado
void fragment() {
    // COLOR = ...
}

// Executado para cálculo de iluminação personalizada
void light() {
    // LIGHT = ...
}
```

---

## 2. Uniforms e Hints no Godot 4

```glsl
uniform vec4 hit_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float progress : hint_range(0.0, 1.0, 0.01) = 0.0;
uniform sampler2D noise_texture : repeat_enable, filter_linear;

// No Godot 4, leitura de tela exige uniform explícito com hint_screen_texture
uniform sampler2D screen_texture : hint_screen_texture, filter_linear_mipmap;
```

---

## 3. Receitas de Shaders Prontos para Produção

### A. Flash de Dano (Hit Flash 2D)
Ideal para piscar o sprite de branco/vermelho ao receber dano:

```glsl
shader_type canvas_item;

uniform vec4 flash_color : source_color = vec4(1.0, 1.0, 1.0, 1.0);
uniform float flash_modifier : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    // Mistura a cor da textura com a cor do flash preservando a transparência original
    COLOR.rgb = mix(tex_color.rgb, flash_color.rgb, flash_modifier);
    COLOR.a = tex_color.a;
}
```

Controlado via GDScript:
```gdscript
func take_damage() -> void:
    var mat := sprite.material as ShaderMaterial
    var tween := create_tween()
    tween.tween_property(mat, "shader_parameter/flash_modifier", 1.0, 0.05)
    tween.tween_property(mat, "shader_parameter/flash_modifier", 0.0, 0.1)
```

---

### B. Contorno 2D (Outline Shader)
Destaca itens interativos ou o personagem selecionado:

```glsl
shader_type canvas_item;

uniform vec4 outline_color : source_color = vec4(1.0, 0.8, 0.0, 1.0);
uniform float outline_width : hint_range(0.0, 10.0) = 1.0;

void fragment() {
    vec2 size = TEXTURE_PIXEL_SIZE * outline_width;
    float alpha = texture(TEXTURE, UV).a;
    
    // Amostra os 4 vizinhos (cima, baixo, esquerda, direita)
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

### C. Dissolução / Desintegração (Dissolve Shader)
Para morte de inimigos ou spawn de chefes:

```glsl
shader_type canvas_item;

uniform sampler2D noise_texture : repeat_enable;
uniform float dissolve_amount : hint_range(0.0, 1.0) = 0.0;
uniform vec4 burn_color : source_color = vec4(1.0, 0.3, 0.0, 1.0);
uniform float burn_size : hint_range(0.0, 0.1) = 0.04;

void fragment() {
    vec4 tex_color = texture(TEXTURE, UV);
    float noise_val = texture(noise_texture, UV).r;
    
    // Descarta pixel se o valor de ruído for inferior ao threshold
    if (noise_val < dissolve_amount) {
        discard;
    }
    
    // Adiciona borda incandescente na linha de corte
    if (noise_val < dissolve_amount + burn_size) {
        COLOR = burn_color;
    } else {
        COLOR = tex_color;
    }
}
```

---

### D. Vinheta de Tela (Post-Processing 2D/3D)
Aplicado em um nó `ColorRect` em tela cheia (Full Rect) dentro de um `CanvasLayer`:

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

## 4. Boas Práticas e Depuração

1. **Evite `discard` desnecessário em mobile:** O comando `discard` desativa otimizações de early-Z em GPUs mobile antigas. Em sprites 2D isolados é seguro.
2. **Reaproveite materiais:** Se 50 inimigos usam o mesmo shader de flash, compartilhe o mesmo `ShaderMaterial` ou use `set_instance_shader_parameter()` para controlar valores individuais sem quebrar o batching de draw calls.
