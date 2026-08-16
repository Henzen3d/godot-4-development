# Animation — AnimationPlayer, AnimationTree, Tweens e Game Feel

## Propósito

Dominar a criação e controle de animações no Godot 4.x, integrando `AnimationPlayer`, máquinas de estados via `AnimationTree`, `AnimatedSprite2D` e animações procedurais polidas via `Tween`.

---

## 1. Escolha da Ferramenta de Animação

```mermaid
graph TD
    A[Qual é a necessidade?] --> B{Apenas sprites frame a frame 2D?}
    B -- Sim --> C[AnimatedSprite2D / SpriteFrames]
    B -- Não --> D{Timeline complexa de propriedades?}
    D -- Sim --> E[AnimationPlayer]
    E --> F{Transições suaves entre estados ou blending 2D/3D?}
    F -- Sim --> G[AnimationTree + StateMachine / BlendSpace]
    A --> H{Micro-interações procedurais via código?}
    H -- Sim --> I[create_tween()]
```

---

## 2. AnimationPlayer: Tipos de Trilhas (Tracks)

O `AnimationPlayer` pode animar qualquer propriedade de qualquer nó:
- **Property Track:** Altera posições, rotações, cores (`modulate`), propriedades de shaders ou visibilidade.
- **Call Method Track:** Executa métodos do GDScript exatamente no frame desejado (ex.: emitir partículas de poeira ou instanciar hitbox no frame 6).
- **Audio Track:** Toca efeitos sonoros sincronizados perfeitamente com os passos ou ataques do personagem.
- **Bezier Track:** Curvas de aceleração personalizadas para movimentos cinematográficos.

---

## 3. AnimationTree: Máquina de Estados e BlendSpace

Para personagens com muitas transições (Idle, Run, Jump, Fall, Attack, Hurt):

```gdscript
extends CharacterBody2D

@onready var animation_tree: AnimationTree = %AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback = animation_tree.get("parameters/playback")

func update_animation_state(input_direction: Vector2) -> void:
    # Atualiza posição no BlendSpace2D (para animações de 4 ou 8 direções)
    if input_direction != Vector2.ZERO:
        animation_tree.set("parameters/Idle/blend_position", input_direction)
        animation_tree.set("parameters/Walk/blend_position", input_direction)
    
    # Transição de estados
    if not is_on_floor():
        if velocity.y < 0:
            state_machine.travel("Jump")
        else:
            state_machine.travel("Fall")
    elif velocity.x != 0.0 or velocity.y != 0.0:
        state_machine.travel("Walk")
    else:
        state_machine.travel("Idle")
```

---

## 4. Animações Procedurais com `Tween` (Godot 4)

No Godot 4, instancie tweens diretamente com `create_tween()`. O tween é destruído automaticamente após a execução ou se o nó for liberado:

### A. Efeito "Pop" / Squash & Stretch ao Coletar Moeda ou Pular
```gdscript
func play_squash_and_stretch() -> void:
    var tween := create_tween().set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_OUT)
    
    # Estica no eixo Y e comprime no X
    tween.tween_property(self, "scale", Vector2(0.8, 1.25), 0.08)
    # Retorna ao tamanho normal com amortecimento
    tween.tween_property(self, "scale", Vector2.ONE, 0.12)
```

### B. Transição de UI / Fade In com Callback
```gdscript
func show_game_over_panel(panel: Control) -> void:
    panel.modulate.a = 0.0
    panel.scale = Vector2(0.8, 0.8)
    panel.visible = true
    
    var tween := create_tween().set_parallel(true).set_trans(Tween.TRANS_QUAD).set_ease(Tween.EASE_OUT)
    tween.tween_property(panel, "modulate:a", 1.0, 0.3)
    tween.tween_property(panel, "scale", Vector2.ONE, 0.3)
    
    # Executa função ao finalizar
    tween.chain().tween_callback(func():
        %RestartButton.grab_focus()
    )
```

### C. Flash de Dano (Piscar Vermelho/Branco)
```gdscript
func play_hit_flash(sprite: CanvasItem) -> void:
    var tween := create_tween()
    tween.tween_property(sprite, "modulate", Color(2.0, 2.0, 2.0, 1.0), 0.05) # Brilho intenso
    tween.tween_property(sprite, "modulate", Color.WHITE, 0.1)
```

---

## 5. Dicas de Otimização e Sincronização

- **Interpolação de Animação:** Para animações suaves mesmo com taxas de quadros variáveis, configure `AnimationPlayer.playback_process_mode` como `PHYSICS` se o nó depender de física.
- **Sinal `animation_finished`:** Prefira conectar sinais a timers manuais para saber quando um golpe ou animação terminou:
  ```gdscript
  animation_player.play("attack")
  await animation_player.animation_finished
  can_act = true
  ```
