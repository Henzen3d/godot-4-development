# Física 2D/3D — Referência Operacional e Padrões de Movimento

## Propósito

Garantir controle preciso de física, colisões confiáveis, raycasting otimizado e movimentação fluida de personagens no Godot 4.x.

---

## 1. Classificação dos Corpos Físicos

| Tipo de Corpo | Uso Principal | Propriedades Críticas |
|---|---|---|
| `CharacterBody2D/3D` | Jogador, NPCs, inimigos controlados por código. | `velocity`, `move_and_slide()`, `is_on_floor()`, `floor_snap_length`. |
| `StaticBody2D/3D` | Chão, paredes, estruturas imóveis. | `constant_linear_velocity` (para esteiras rolantes). |
| `AnimatableBody2D/3D` | Plataformas móveis, elevadores, portas automáticas. | `sync_to_physics = true` (evita esmagamento/jitter). |
| `RigidBody2D/3D` | Caixas empurradas, detritos, veículos com física pura. | `apply_central_impulse()`, `mass`, `physics_material_override`. |
| `Area2D/3D` | Triggers, checkpoints, hitboxes, zonas de gravidade. | `monitoring`, `monitorable`, `body_entered`, `area_entered`. |

---

## 2. Camadas e Máscaras de Colisão (Layers vs Masks)

- **Collision Layer:** *"Em qual camada este objeto existe?"*
- **Collision Mask:** *"Quais camadas este objeto deve escanear / colidir?"*

### Matriz Recomendada
- Layer 1: Mundo / Terreno (`World`)
- Layer 2: Jogador (`Player`)
- Layer 3: Inimigos (`Enemies`)
- Layer 4: Projéteis do Jogador (`PlayerBullets`)
- Layer 5: Projéteis dos Inimigos (`EnemyBullets`)
- Layer 6: Sensores / Coletáveis (`Pickups / Triggers`)

> [!TIP]
> Configure os nomes das camadas em `Project Settings > Layer Names > 2D Physics` para visualizá-las pelo nome no editor.

---

## 3. Template de Movimento 2D Completo (Plataforma / Top-Down)

### Movimentação de Plataforma com Coyote Time e Jump Buffer

```gdscript
class_name PlatformerController
extends CharacterBody2D

@export_group("Horizontal Movement")
@export var max_speed: float = 240.0
@export var acceleration: float = 1200.0
@export var friction: float = 1400.0

@export_group("Jump Physics")
@export var jump_velocity: float = -380.0
@export var min_jump_velocity: float = -180.0 # Pulo curto ao soltar tecla
@export var coyote_time: float = 0.12
@export var jump_buffer_time: float = 0.1

var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")
var coyote_timer: float = 0.0
var jump_buffer_timer: float = 0.0

func _physics_process(delta: float) -> void:
    # 1. Atualizar timers
    if is_on_floor():
        coyote_timer = coyote_time
    else:
        coyote_timer = maxf(0.0, coyote_timer - delta)
        velocity.y += gravity * delta
    
    if Input.is_action_just_pressed("jump"):
        jump_buffer_timer = jump_buffer_time
    else:
        jump_buffer_timer = maxf(0.0, jump_buffer_timer - delta)
    
    # 2. Executar Pulo (com Coyote e Buffer)
    if jump_buffer_timer > 0.0 and coyote_timer > 0.0:
        velocity.y = jump_velocity
        jump_buffer_timer = 0.0
        coyote_timer = 0.0
    
    # Pulo com altura variável ao soltar o botão
    if Input.is_action_just_released("jump") and velocity.y < min_jump_velocity:
        velocity.y = min_jump_velocity
    
    # 3. Movimento Horizontal
    var input_direction := Input.get_axis("move_left", "move_right")
    if input_direction != 0.0:
        velocity.x = move_toward(velocity.x, input_direction * max_speed, acceleration * delta)
    else:
        velocity.x = move_toward(velocity.x, 0.0, friction * delta)
    
    # 4. Ajuste de Rampas e Snap
    floor_snap_length = 8.0 if is_on_floor() else 0.0
    floor_stop_on_slope = true
    floor_max_angle = deg_to_rad(46.0)
    
    # 5. Execução física
    move_and_slide()
```

---

## 4. Consultas Espaciais e Raycasting

### Via Nó `RayCast2D/3D`
```gdscript
@onready var ground_ray: RayCast2D = %GroundRay

func check_ledge() -> bool:
    return not ground_ray.is_colliding()
```

### Via Direct Space State (Raycast Direto por Código)
```gdscript
func cast_vision_ray(from_pos: Vector2, target_pos: Vector2) -> Dictionary:
    var space_state := get_world_2d().direct_space_state
    var query := PhysicsRayQueryParameters2D.create(from_pos, target_pos)
    query.collision_mask = 1 # Apenas camada World
    query.exclude = [self]   # Ignora o próprio emissor
    
    return space_state.intersect_ray(query) # Retorna {position, normal, collider, shape, ...} ou {}
```

---

## 5. Áreas, Hitboxes e Desativação Segura

Ao desativar colisões durante o processamento de física, sempre utilize `set_deferred`:

```gdscript
func _on_hurtbox_area_entered(hitbox: Area2D) -> void:
    if hitbox.is_in_group("enemy_attack"):
        take_damage(hitbox.damage)
        # Desativa colisão temporariamente com segurança
        %CollisionShape2D.set_deferred("disabled", true)
        await get_tree().create_timer(1.0).timeout
        %CollisionShape2D.set_deferred("disabled", false)
```

---

## 6. Navegação 2D/3D com `NavigationAgent`

```gdscript
extends CharacterBody2D

@export var movement_speed: float = 150.0
@onready var nav_agent: NavigationAgent2D = %NavigationAgent2D

func set_destination(target_global_pos: Vector2) -> void:
    nav_agent.target_position = target_global_pos

func _physics_process(_delta: float) -> void:
    if nav_agent.is_navigation_finished():
        velocity = Vector2.ZERO
        move_and_slide()
        return
    
    var next_path_pos := nav_agent.get_next_path_position()
    var move_dir := global_position.direction_to(next_path_pos)
    velocity = move_dir * movement_speed
    move_and_slide()
```
