# GDScript 4.x — Referência Operacional e Padrões Idiomáticos

## Propósito

Escrever código limpo, robusto e idiomático para o **GDScript 2.0 (Godot 4.x)**, priorizando tipagem estática, anotações modernas, sinais desacoplados e arquitetura baseada em recursos.

---

## 1. Tipagem Estática Rigorosa

A tipagem estática no Godot 4 acelera a execução (otimizações de bytecode), previne erros em tempo de edição e fornece autocompletion confiável:

```gdscript
class_name PlayerController
extends CharacterBody2D

# Tipagem explícita e inferida com :=
const DEFAULT_SPEED: float = 300.0
var current_health: int = 100
var is_alive: bool = true

# Arrays e Dicionários tipados
var inventory: Array[ItemData] = []
var player_stats: Dictionary = {} # Ou Dictionary[String, Variant]

# Retorno e parâmetros estritamente tipados
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
    # Lógica de morte
```

---

## 2. Anotações Modernas do GDScript 4

Substitua as antigas sintaxes por anotações com `@`:

```gdscript
extends Node2D

# Grupos de propriedades no Inspector
@export_group("Movement Settings", "move_")
@export var move_speed: float = 250.0
@export_range(100.0, 1000.0, 25.0) var move_acceleration: float = 500.0

@export_group("Combat", "combat_")
@export_range(1, 100, 1) var combat_base_damage: int = 15
@export_enum("Melee", "Ranged", "Magic") var combat_attack_type: String = "Melee"

@export_group("References")
@export var character_stats: CharacterStatsResource # Custom Resource
@export_file("*.tscn") var death_effect_scene: String

# Referências @onready
@onready var animated_sprite: AnimatedSprite2D = $AnimatedSprite2D
@onready var health_bar: ProgressBar = %HealthBar # Scene Unique Node (%)
```

---

## 3. Sinais e Callables (Sintaxe 4.x)

No Godot 4, sinais são objetos de primeira classe. Nunca use strings para conectar ou emitir sinais:

```gdscript
# Declaração tipada de sinal
signal health_changed(new_health: int, max_health: int)
signal died

func _ready() -> void:
    # Conexão direta com método Callable
    health_changed.connect(_on_health_changed)
    
    # Conexão com flag (ex.: CONNECT_ONE_SHOT)
    died.connect(_on_died, CONNECT_ONE_SHOT)
    
    # Função anônima (Lambda)
    $HitBox.area_entered.connect(func(area: Area2D) -> void:
        print("Hit por: ", area.name)
    )

# Emissão moderna com .emit()
func take_hit(damage: int) -> void:
    current_health = maxi(0, current_health - damage)
    health_changed.emit(current_health, 100)
    
    if current_health <= 0:
        died.emit()

func _on_health_changed(new_val: int, max_val: int) -> void:
    print("Vida: ", new_val, "/", max_val)

func _on_died() -> void:
    queue_free()
```

---

## 4. Assincronismo com `await` (Godot 4 não possui `yield`)

O comando `yield` foi removido no Godot 4. Utilize `await`:

```gdscript
func play_attack_sequence() -> void:
    animated_sprite.play("attack_windup")
    
    # Aguarda timer descartável
    await get_tree().create_timer(0.3).timeout
    
    _spawn_hitbox()
    animated_sprite.play("attack_swing")
    
    # Aguarda sinal específico de um nó
    await animated_sprite.animation_finished
    
    animated_sprite.play("idle")
```

---

## 5. Custom Resources (Arquitetura Orientada a Dados)

Use `Resource` para armazenar status, configurações de itens, armas e diálogos sem sobrecarregar a `SceneTree`:

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

Para usar em qualquer script:
```gdscript
@export var starting_item: ItemData
```

---

## 6. Movimento com `CharacterBody2D/3D`

- No Godot 4, `velocity` é uma propriedade interna de `CharacterBody2D/3D`.
- **`move_and_slide()` não aceita argumentos e já calcula internamente a multiplicação por `delta`**.
- Acelerações contínuas (como gravidade) **devem** ser multiplicadas por `delta`.

```gdscript
extends CharacterBody2D

@export var speed: float = 200.0
@export var jump_velocity: float = -400.0

# Obtém a gravidade padrão do projeto
var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

func _physics_process(delta: float) -> void:
    # 1. Aplicar gravidade
    if not is_on_floor():
        velocity.y += gravity * delta
    
    # 2. Pulo
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = jump_velocity
    
    # 3. Direção horizontal
    var direction := Input.get_axis("move_left", "move_right")
    if direction != 0.0:
        velocity.x = direction * speed
    else:
        velocity.x = move_toward(velocity.x, 0.0, speed)
    
    # 4. Executa movimento físico (move_and_slide não recebe parâmetros no Godot 4)
    move_and_slide()
```

---

## 7. Tabela de Migração: Godot 3.x vs Godot 4.x

| Conceito | Godot 3.x (Antigo) | Godot 4.x (Atual) |
|---|---|---|
| Corpo de Personagem | `KinematicBody2D/3D` | `CharacterBody2D/3D` |
| Movimento de Corpo | `velocity = move_and_slide(velocity, ...)` | `move_and_slide()` (usa `velocity` interno) |
| Emissão de Sinal | `emit_signal("health_depleted", amount)` | `health_depleted.emit(amount)` |
| Conexão de Sinal | `connect("pressed", self, "_on_pressed")` | `pressed.connect(_on_pressed)` |
| Espera Assíncrona | `yield(get_tree().create_timer(1.0), "timeout")` | `await get_tree().create_timer(1.0).timeout` |
| Anotações | `onready var x = $X`, `export var y` | `@onready var x = $X`, `@export var y` |
| Nós Únicos de Cena | `get_node("Path/To/Node")` | `%UniqueNodeName` |
| Vetor de Input | `Vector2(Input.get_action_strength(...))` | `Input.get_vector("left", "right", "up", "down")` |
| Criação de Tweens | `var t = Tween.new(); add_child(t)` | `var t = create_tween()` |
