# 2D/3D Physics — Operational Reference & Movement Patterns

## Purpose

Ensure precise physics control, reliable collisions, optimized raycasting, and responsive character controllers in Godot 4.x.

---

## 1. Physics Body Classification

| Body Type | Primary Use | Critical Properties |
|---|---|---|
| `CharacterBody2D/3D` | Player, NPCs, scripted enemies. | `velocity`, `move_and_slide()`, `is_on_floor()`, `floor_snap_length`. |
| `StaticBody2D/3D` | Floors, walls, non-moving geometry. | `constant_linear_velocity` (for conveyor belts). |
| `AnimatableBody2D/3D` | Moving platforms, elevators, automatic doors. | `sync_to_physics = true` (prevents jitter and crushing). |
| `RigidBody2D/3D` | Pushable crates, debris, pure physics vehicles. | `apply_central_impulse()`, `mass`, `physics_material_override`. |
| `Area2D/3D` | Triggers, checkpoints, hitboxes, gravity zones. | `monitoring`, `monitorable`, `body_entered`, `area_entered`. |

---

## 2. Collision Layers vs Collision Masks

- **Collision Layer:** *"What layer does this object exist on?"*
- **Collision Mask:** *"What layers does this object scan / collide with?"*

### Recommended Layer Matrix
- Layer 1: World / Environment (`World`)
- Layer 2: Player (`Player`)
- Layer 3: Enemies (`Enemies`)
- Layer 4: Player Projectiles (`PlayerBullets`)
- Layer 5: Enemy Projectiles (`EnemyBullets`)
- Layer 6: Pickups & Sensors (`Pickups / Triggers`)

> [!TIP]
> Name your collision layers in `Project Settings > Layer Names > 2D Physics` (or `3D Physics`) to view meaningful names in the Inspector.

---

## 3. Complete 2D Platformer Movement Controller

### Platformer Movement with Coyote Time & Jump Buffer

```gdscript
class_name PlatformerController
extends CharacterBody2D

@export_group("Horizontal Movement")
@export var max_speed: float = 240.0
@export var acceleration: float = 1200.0
@export var friction: float = 1400.0

@export_group("Jump Physics")
@export var jump_velocity: float = -380.0
@export var min_jump_velocity: float = -180.0 # Variable jump height on button release
@export var coyote_time: float = 0.12
@export var jump_buffer_time: float = 0.1

var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")
var coyote_timer: float = 0.0
var jump_buffer_timer: float = 0.0

func _physics_process(delta: float) -> void:
    # 1. Update timers
    if is_on_floor():
        coyote_timer = coyote_time
    else:
        coyote_timer = maxf(0.0, coyote_timer - delta)
        velocity.y += gravity * delta
    
    if Input.is_action_just_pressed("jump"):
        jump_buffer_timer = jump_buffer_time
    else:
        jump_buffer_timer = maxf(0.0, jump_buffer_timer - delta)
    
    # 2. Execute Jump (with Coyote Time and Buffer)
    if jump_buffer_timer > 0.0 and coyote_timer > 0.0:
        velocity.y = jump_velocity
        jump_buffer_timer = 0.0
        coyote_timer = 0.0
    
    # Variable jump height cut
    if Input.is_action_just_released("jump") and velocity.y < min_jump_velocity:
        velocity.y = min_jump_velocity
    
    # 3. Horizontal Movement
    var input_direction := Input.get_axis("move_left", "move_right")
    if input_direction != 0.0:
        velocity.x = move_toward(velocity.x, input_direction * max_speed, acceleration * delta)
    else:
        velocity.x = move_toward(velocity.x, 0.0, friction * delta)
    
    # 4. Slopes & Snapping
    floor_snap_length = 8.0 if is_on_floor() else 0.0
    floor_stop_on_slope = true
    floor_max_angle = deg_to_rad(46.0)
    
    # 5. Physics execution
    move_and_slide()
```

---

## 4. Spatial Queries & Raycasting

### Using `RayCast2D/3D` Node
```gdscript
@onready var ground_ray: RayCast2D = %GroundRay

func check_ledge() -> bool:
    return not ground_ray.is_colliding()
```

### Using Direct Space State (Code-Driven Raycast)
```gdscript
func cast_vision_ray(from_pos: Vector2, target_pos: Vector2) -> Dictionary:
    var space_state := get_world_2d().direct_space_state
    var query := PhysicsRayQueryParameters2D.create(from_pos, target_pos)
    query.collision_mask = 1 # World layer only
    query.exclude = [self]   # Ignore caster
    
    return space_state.intersect_ray(query) # Returns {position, normal, collider, shape, ...} or {}
```

---

## 5. Areas, Hitboxes & Safe Deferral

When disabling collision shapes during physics callbacks, always use `set_deferred`:

```gdscript
func _on_hurtbox_area_entered(hitbox: Area2D) -> void:
    if hitbox.is_in_group("enemy_attack"):
        take_damage(hitbox.damage)
        # Safely disable collision during physics step
        %CollisionShape2D.set_deferred("disabled", true)
        await get_tree().create_timer(1.0).timeout
        %CollisionShape2D.set_deferred("disabled", false)
```

---

## 6. 2D/3D Navigation with `NavigationAgent`

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
