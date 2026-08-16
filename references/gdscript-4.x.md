# GDScript 4.x — Operational Reference & Idiomatic Patterns

## Purpose

Write clean, robust, and idiomatic code for **GDScript 2.0 (Godot 4.x)**, prioritizing static typing, modern annotations, decoupled signals, and resource-driven architecture.

---

## 1. Strict Static Typing

Static typing in Godot 4 speeds up execution (bytecode optimizations), prevents errors at edit-time, and provides reliable autocompletion:

```gdscript
class_name PlayerController
extends CharacterBody2D

# Explicit and inferred typing with :=
const DEFAULT_SPEED: float = 300.0
var current_health: int = 100
var is_alive: bool = true

# Typed Arrays and Dictionaries
var inventory: Array[ItemData] = []
var player_stats: Dictionary = {} # Or Dictionary[String, Variant]

# Strictly typed returns and parameters
func apply_damage(amount: int, knockback_vector: Vector2 = Vector2.ZERO) -> bool:
    if not is_alive:
        return false
    
    current_health = maxi(0, current_health - amount)
    velocity += knockback_vector
    
    if current_health == 0:
        _die()
    return true

func _die() -> void:
    is_alive = false
    # Death logic
```

---

## 2. Modern GDScript 4 Annotations

Replace legacy export/onready syntax with `@` annotations:

```gdscript
extends Node2D

# Inspector Property Groups
@export_group("Movement Settings", "move_")
@export var move_speed: float = 250.0
@export_range(100.0, 1000.0, 25.0) var move_acceleration: float = 500.0

@export_group("Combat", "combat_")
@export_range(1, 100, 1) var combat_base_damage: int = 15
@export_enum("Melee", "Ranged", "Magic") var combat_attack_type: String = "Melee"

@export_group("References")
@export var character_stats: CharacterStatsResource # Custom Resource
@export_file("*.tscn") var death_effect_scene: String

# @onready references
@onready var animated_sprite: AnimatedSprite2D = $AnimatedSprite2D
@onready var health_bar: ProgressBar = %HealthBar # Scene Unique Node (%)
```

---

## 3. Signals and Callables (4.x Syntax)

In Godot 4, signals are first-class objects. Never use string names to connect or emit signals:

```gdscript
# Typed signal declaration
signal health_changed(new_health: int, max_health: int)
signal died

func _ready() -> void:
    # Direct connection to Callable method
    health_changed.connect(_on_health_changed)
    
    # Connection with flags (e.g., CONNECT_ONE_SHOT)
    died.connect(_on_died, CONNECT_ONE_SHOT)
    
    # Anonymous function (Lambda)
    $HitBox.area_entered.connect(func(area: Area2D) -> void:
        print("Hit by: ", area.name)
    )

# Modern emission with .emit()
func take_hit(damage: int) -> void:
    current_health = maxi(0, current_health - damage)
    health_changed.emit(current_health, 100)
    
    if current_health <= 0:
        died.emit()

func _on_health_changed(new_val: int, max_val: int) -> void:
    print("Health: ", new_val, "/", max_val)

func _on_died() -> void:
    queue_free()
```

---

## 4. Asynchronous Code with `await` (Godot 4 has no `yield`)

The `yield` keyword was removed in Godot 4. Always use `await`:

```gdscript
func play_attack_sequence() -> void:
    animated_sprite.play("attack_windup")
    
    # Wait for one-shot timer
    await get_tree().create_timer(0.3).timeout
    
    _spawn_hitbox()
    animated_sprite.play("attack_swing")
    
    # Wait for specific signal from a node
    await animated_sprite.animation_finished
    
    animated_sprite.play("idle")
```

---

## 5. Custom Resources (Data-Driven Architecture)

Use `Resource` to store statistics, item configurations, weapons, and dialogues without polluting the `SceneTree`:

```gdscript
# ItemData.gd
class_name ItemData
extends Resource

@export var id: String = ""
@export var display_name: String = ""
@export var icon: Texture2D
@export var stack_limit: int = 99
@export var value: int = 10
```

To use in any script:
```gdscript
@export var starting_item: ItemData
```

---

## 6. Movement with `CharacterBody2D/3D`

- In Godot 4, `velocity` is a built-in property of `CharacterBody2D/3D`.
- **`move_and_slide()` takes no parameters and internally handles delta multiplication**.
- Continuous accelerations (such as gravity) **must** be multiplied by `delta`.

```gdscript
extends CharacterBody2D

@export var speed: float = 200.0
@export var jump_velocity: float = -400.0

# Get project default gravity
var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

func _physics_process(delta: float) -> void:
    # 1. Apply gravity
    if not is_on_floor():
        velocity.y += gravity * delta
    
    # 2. Jump
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = jump_velocity
    
    # 3. Horizontal direction
    var direction := Input.get_axis("move_left", "move_right")
    if direction != 0.0:
        velocity.x = direction * speed
    else:
        velocity.x = move_toward(velocity.x, 0.0, speed)
    
    # 4. Execute physics movement (move_and_slide takes no parameters in Godot 4)
    move_and_slide()
```

---

## 7. Migration Table: Godot 3.x vs Godot 4.x

| Concept | Godot 3.x (Legacy) | Godot 4.x (Modern) |
|---|---|---|
| Character Body | `KinematicBody2D/3D` | `CharacterBody2D/3D` |
| Body Movement | `velocity = move_and_slide(velocity, ...)` | `move_and_slide()` (uses built-in `velocity`) |
| Signal Emission | `emit_signal("health_depleted", amount)` | `health_depleted.emit(amount)` |
| Signal Connection | `connect("pressed", self, "_on_pressed")` | `pressed.connect(_on_pressed)` |
| Async Wait | `yield(get_tree().create_timer(1.0), "timeout")` | `await get_tree().create_timer(1.0).timeout` |
| Annotations | `onready var x = $X`, `export var y` | `@onready var x = $X`, `@export var y` |
| Scene Unique Nodes | `get_node("Path/To/Node")` | `%UniqueNodeName` |
| Input Vector | `Vector2(Input.get_action_strength(...))` | `Input.get_vector("left", "right", "up", "down")` |
| Tween Creation | `var t = Tween.new(); add_child(t)` | `var t = create_tween()` |
