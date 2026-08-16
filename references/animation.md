# Animation — AnimationPlayer, AnimationTree, Tweens & Game Feel

## Purpose

Master animation creation and control in Godot 4.x, integrating `AnimationPlayer`, state machines via `AnimationTree`, `AnimatedSprite2D`, and polished procedural animations using `Tween`.

---

## 1. Choosing the Right Animation Tool

```mermaid
graph TD
    A[What is the animation need?] --> B{Simple 2D frame-by-frame sprites?}
    B -- Yes --> C[AnimatedSprite2D / SpriteFrames]
    B -- No --> D{Complex multi-property timeline?}
    D -- Yes --> E[AnimationPlayer]
    E --> F{Smooth state transitions or 2D/3D blending?}
    F -- Yes --> G[AnimationTree + StateMachine / BlendSpace]
    A --> H{Procedural micro-interactions in code?}
    H -- Yes --> I[create_tween()]
```

---

## 2. AnimationPlayer: Track Types

`AnimationPlayer` can animate any property on any node:
- **Property Track:** Changes positions, rotations, colors (`modulate`), shader parameters, or visibility.
- **Call Method Track:** Executes GDScript methods on precise keyframes (e.g., spawn dust particles or enable hitbox at frame 6).
- **Audio Track:** Plays sound effects perfectly synced with footsteps or attack animations.
- **Bezier Track:** Custom easing curves for cinematic motion and smooth camera movements.

---

## 3. AnimationTree: State Machine & BlendSpace

For characters with multiple state transitions (Idle, Run, Jump, Fall, Attack, Hurt):

```gdscript
extends CharacterBody2D

@onready var animation_tree: AnimationTree = %AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback = animation_tree.get("parameters/playback")

func update_animation_state(input_direction: Vector2) -> void:
    # Update BlendSpace2D coordinates (for 4-way or 8-way directional animations)
    if input_direction != Vector2.ZERO:
        animation_tree.set("parameters/Idle/blend_position", input_direction)
        animation_tree.set("parameters/Walk/blend_position", input_direction)
    
    # State transitions
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

## 4. Procedural Animations with `Tween` (Godot 4)

In Godot 4, create tweens directly with `create_tween()`. The tween is automatically freed when finished or if the owning node is deleted:

### A. "Pop" / Squash & Stretch Effect (Coin Pickup, Jump Launch)
```gdscript
func play_squash_and_stretch() -> void:
    var tween := create_tween().set_trans(Tween.TRANS_BACK).set_ease(Tween.EASE_OUT)
    
    # Stretch Y and compress X
    tween.tween_property(self, "scale", Vector2(0.8, 1.25), 0.08)
    # Return to normal scale with elastic bounce
    tween.tween_property(self, "scale", Vector2.ONE, 0.12)
```

### B. UI Transition / Fade In with Completion Callback
```gdscript
func show_game_over_panel(panel: Control) -> void:
    panel.modulate.a = 0.0
    panel.scale = Vector2(0.8, 0.8)
    panel.visible = true
    
    var tween := create_tween().set_parallel(true).set_trans(Tween.TRANS_QUAD).set_ease(Tween.EASE_OUT)
    tween.tween_property(panel, "modulate:a", 1.0, 0.3)
    tween.tween_property(panel, "scale", Vector2.ONE, 0.3)
    
    # Execute callback when tween finishes
    tween.chain().tween_callback(func():
        %RestartButton.grab_focus()
    )
```

### C. Damage Flash Effect
```gdscript
func play_hit_flash(sprite: CanvasItem) -> void:
    var tween := create_tween()
    tween.tween_property(sprite, "modulate", Color(2.0, 2.0, 2.0, 1.0), 0.05) # Overbright glow
    tween.tween_property(sprite, "modulate", Color.WHITE, 0.1)
```

---

## 5. Optimization & Synchronization Tips

- **Animation Process Mode:** For physics-dependent animations, set `AnimationPlayer.playback_process_mode` to `PHYSICS` to prevent jitter.
- **`animation_finished` Signal:** Prefer awaiting the signal over manual timers to detect animation completion:
  ```gdscript
  animation_player.play("attack")
  await animation_player.animation_finished
  can_act = true
  ```
