# Audio — Players, Buses, AudioStreamRandomizer & Sound Feedback

## Purpose

Ensure immersive, dynamic, non-fatiguing audio design in Godot 4.x using audio buses, sound effects, randomization resources, and SFX pooling.

---

## 1. Audio Player Types

| Player Type | Spatial Scope | Recommended Use |
|---|---|---|
| `AudioStreamPlayer` | Global / Stereo | Background music (BGM), UI clicks, narrator voiceovers. |
| `AudioStreamPlayer2D` | Positional 2D | Footsteps, explosions, torches, enemies (distance attenuation and panning). |
| `AudioStreamPlayer3D` | Positional 3D | 3D sound effects with Doppler effect, physical attenuation, and emission cones. |

---

## 2. Audio Buses Structure

Configure audio buses in the `Audio` bottom panel of the editor:

```text
Master (0 dB)
├── Music (BGM Volume)
│   └── [Optional Effect: LowPassFilter disabled during gameplay, enabled in Pause Menu]
├── SFX (Sound Effects Volume)
│   ├── Combat
│   └── Environment (Effect: Reverb)
└── UI (Interface Volume)
```

### Controlling Volume via Code (Decibels vs Linear)
UI sliders operate on linear scale $0.0$ to $1.0$. Godot handles volume in decibels ($-\infty$ to $0\text{ dB}$):
```gdscript
func set_bus_volume(bus_name: String, linear_value: float) -> void:
    var bus_idx := AudioServer.get_bus_index(bus_name)
    var db := linear_to_db(linear_value)
    AudioServer.set_bus_volume_db(bus_idx, db)
    AudioServer.set_bus_mute(bus_idx, linear_value <= 0.001)
```

---

## 3. `AudioStreamRandomizer` (Anti-Fatigue Audio)

In Godot 4, use the built-in `AudioStreamRandomizer` resource in the Inspector to prevent repetitive auditory fatigue:
- **Streams:** Add 2 to 5 sound variations (e.g., 4 distinct footstep recordings).
- **Random Pitch:** Set `random_pitch = 1.15` (varies pitch by $\pm 15\%$ on each trigger).
- **Random Volume:** Set `random_volume_offset_db = 1.5`.

---

## 4. Native Polyphony (`max_polyphony`)

In Godot 4, you can configure the `max_polyphony` property directly on `AudioStreamPlayer` (e.g., `max_polyphony = 8`).
- Allows playing rapid repeated sounds without prematurely cutting off previous echoing audio instances.

---

## 5. One-Shot Positional SFX Spawner

For sounds played at the position of a destroyed entity (e.g., an enemy explosion immediately calling `queue_free()`):

```gdscript
# AudioManager.gd (Autoload)
extends Node

func play_sfx_at_position(stream: AudioStream, global_pos: Vector2, bus: String = "SFX") -> void:
    var player := AudioStreamPlayer2D.new()
    player.stream = stream
    player.bus = bus
    player.global_position = global_pos
    player.autoplay = true
    
    # Automatically free when audio ends
    player.finished.connect(player.queue_free)
    
    get_tree().current_scene.add_child(player)
```

---

## 6. Music Crossfade with `Tween`

Seamlessly blend between music tracks:

```gdscript
func crossfade_to_music(new_stream: AudioStream, duration: float = 1.5) -> void:
    var tween := create_tween().set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_IN_OUT)
    
    # Fade out current track
    tween.tween_property(%MusicPlayer, "volume_db", -40.0, duration)
    tween.tween_callback(func():
        %MusicPlayer.stream = new_stream
        %MusicPlayer.play()
    )
    # Fade in new track
    tween.tween_property(%MusicPlayer, "volume_db", 0.0, duration)
```
