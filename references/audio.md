# Áudio — Players, Buses, AudioStreamRandomizer e Feedback Sonoro

## Propósito

Garantir um design de áudio imersivo, dinâmico e sem repetições cansativas no Godot 4.x, utilizando barramentos, efeitos, recursos de randomização e pooling de som.

---

## 1. Escolha dos Players de Áudio

| Tipo de Player | Escopo | Uso Recomendado |
|---|---|---|
| `AudioStreamPlayer` | Global / Estéreo | Trilha musical (BGM), cliques de UI, vozes de narrador. |
| `AudioStreamPlayer2D` | Posicional 2D | Passos, explosões, tochas, inimigos (possui atenuação por distância e pan). |
| `AudioStreamPlayer3D` | Espacial 3D | Efeitos sonoros 3D com efeito Doppler, atenuação física e cone de emissão. |

---

## 2. Estrutura de Barramentos (Audio Buses)

Configure os barramentos em `Audio` (aba inferior do editor):

```text
Master (0 dB)
├── Music (Volume BGM)
│   └── [Efeito Opcional: LowPassFilter desativado para menu de pausa]
├── SFX (Volume Efeitos)
│   ├── Combat
│   └── Environment (Efeito: Reverb)
└── UI (Volume Interface)
```

### Controle de Volume por Código (Decibéis vs Linear)
Sliders de UI operam de $0.0$ a $1.0$ (linear). O Godot opera em decibéis ($-\infty$ a $0\text{ dB}$):
```gdscript
func set_bus_volume(bus_name: String, linear_value: float) -> void:
    var bus_idx := AudioServer.get_bus_index(bus_name)
    var db := linear_to_db(linear_value)
    AudioServer.set_bus_volume_db(bus_idx, db)
    AudioServer.set_bus_mute(bus_idx, linear_value <= 0.001)
```

---

## 3. `AudioStreamRandomizer` (Anti-Fadiga Auditiva)

No Godot 4, utilize o recurso nativo `AudioStreamRandomizer` no Inspector para evitar sons repetitivos:
- **Streams:** Adicione 2 a 5 variações do mesmo som (ex.: 4 sons de passos diferentes).
- **Random Pitch:** Configure `random_pitch = 1.15` (varia a afinação em $\pm 15\%$ a cada disparo).
- **Random Volume:** Configure `random_volume_offset_db = 1.5`.

---

## 4. Polifonia Nativa (`max_polyphony`)

No Godot 4, você pode configurar a propriedade `max_polyphony` no próprio `AudioStreamPlayer` (ex.: `max_polyphony = 8`).
- Permite tocar o mesmo player repetidamente sem cortar o som anterior que ainda está ecoando.

---

## 5. Gerenciador de Áudio Transiente (One-Shot SFX Helper)

Para sons que ocorrem na posição de um objeto destruído (ex.: explosão de um inimigo que acabou de chamar `queue_free()`):

```gdscript
# AudioManager.gd (Autoload)
extends Node

func play_sfx_at_position(stream: AudioStream, global_pos: Vector2, bus: String = "SFX") -> void:
    var player := AudioStreamPlayer2D.new()
    player.stream = stream
    player.bus = bus
    player.global_position = global_pos
    player.autoplay = true
    
    # Destrói automaticamente quando o som acabar
    player.finished.connect(player.queue_free)
    
    get_tree().current_scene.add_child(player)
```

---

## 6. Crossfade de Música com `Tween`

Para alternar entre faixas de música de forma cinematográfica:

```gdscript
func crossfade_to_music(new_stream: AudioStream, duration: float = 1.5) -> void:
    var tween := create_tween().set_trans(Tween.TRANS_SINE).set_ease(Tween.EASE_IN_OUT)
    
    # Fade out da faixa atual
    tween.tween_property(%MusicPlayer, "volume_db", -40.0, duration)
    tween.tween_callback(func():
        %MusicPlayer.stream = new_stream
        %MusicPlayer.play()
    )
    # Fade in da nova faixa
    tween.tween_property(%MusicPlayer, "volume_db", 0.0, duration)
```
