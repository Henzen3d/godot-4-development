# UI — Interface Systems, Layouts & Themes

## Purpose

Build responsive, gamepad/keyboard accessible, aesthetically polished user interfaces decoupled from gameplay logic in Godot Engine 4.x.

---

## 1. Containers and Layout Hierarchy

Never position interface elements using manual coordinates (`position`). Always use **Containers**:

```text
CanvasLayer (Layer = 1)
└── MarginContainer (Anchors: Full Rect | Margins: 24px)
    └── VBoxContainer
        ├── HBoxContainer (Top Bar)
        │   ├── TextureRect (Player Avatar)
        │   ├── ProgressBar (HealthBar - Size Flag: Expand Fill)
        │   └── Label (Score)
        ├── Control (Spacer - Size Flag: Expand Fill)
        └── PanelContainer (Dialogue Box / Notification)
```

### Key UI Containers
- **`MarginContainer`:** Adds consistent external padding around its children.
- **`VBoxContainer` / `HBoxContainer`:** Aligns nodes vertically or horizontally with uniform spacing (`theme_override_constants/separation`).
- **`GridContainer`:** Organizes children in fixed columns (ideal for inventory grids and shops).
- **`PanelContainer`:** Draws a background frame with a `StyleBox` around child contents.
- **`CenterContainer`:** Centers menus or popups in the middle of the screen.
- **`ScrollContainer`:** Provides scrollable viewports for long text or item lists.

---

## 2. Size Flags (Expansion Control)

For child nodes inside containers:
- **`Shrink Begin / Center / End`:** Occupies only the minimum required size.
- **`Fill`:** Fills available allotted space without requesting extra room.
- **`Expand + Fill`:** Requests all remaining space from the parent container and expands to fill it.

---

## 3. Mouse Filters and Input Consumption

Click behaviors of `Control` nodes are determined by `mouse_filter`:

| Mode | Effect | Recommended Use |
|---|---|---|
| `STOP` (Default) | Consumes the mouse event; does not pass it down to lower nodes or gameplay. | `Button`, `Slider`, `LineEdit`. |
| `PASS` | Responds to the event, but allows parent container to receive it as well. | Clickable list items. |
| `IGNORE` | Transparent to mouse events; clicks pass through to the game world. | `Label`, background `TextureRect`, fullscreen `MarginContainer`. |

> [!WARNING]
> If game clicks are not registering, check if a fullscreen `MarginContainer` or `Control` is set to `mouse_filter = STOP`. Change it to `IGNORE`.

---

## 4. Focus Navigation for Gamepad & Keyboard

For native controller (Xbox, PlayStation) and keyboard navigation:

```gdscript
extends Control

func _ready() -> void:
    # Ensure first button receives initial focus when menu opens
    %StartButton.grab_focus()

func setup_focus_neighbors() -> void:
    # Explicit focus neighboring (if automatic order needs overriding)
    %StartButton.focus_neighbor_bottom = %OptionsButton.get_path()
    %OptionsButton.focus_neighbor_top = %StartButton.get_path()
    %OptionsButton.focus_neighbor_bottom = %QuitButton.get_path()
    %QuitButton.focus_neighbor_top = %OptionsButton.get_path()
```

---

## 5. Styling with `StyleBoxFlat` and Themes

Create modern UI buttons and panels via code or `.tres` themes:

```gdscript
func apply_custom_button_style(button: Button) -> void:
    var style_normal := StyleBoxFlat.new()
    style_normal.bg_color = Color("#1e293b") # Dark slate
    style_normal.corner_radius_top_left = 8
    style_normal.corner_radius_top_right = 8
    style_normal.corner_radius_bottom_left = 8
    style_normal.corner_radius_bottom_right = 8
    style_normal.content_margin_left = 16.0
    style_normal.content_margin_right = 16.0
    
    var style_hover := style_normal.duplicate() as StyleBoxFlat
    style_hover.bg_color = Color("#3b82f6") # Accent blue
    
    button.add_theme_stylebox_override("normal", style_normal)
    button.add_theme_stylebox_override("hover", style_hover)
    button.add_theme_color_override("font_color", Color.WHITE)
```

---

## 6. Multi-Resolution Display Settings

Under `project.godot > display/window/stretch`:
- **`mode`**:
  - `canvas_items`: Recommended for 2D HD/4K or 3D games (UI and vector text render at native monitor resolution).
  - `viewport`: Recommended for Pixel Art games with fixed low internal resolution (e.g., $320 \times 180$).
- **`aspect`**:
  - `keep`: Preserves aspect ratio with letterboxing (e.g., 16:9 on 21:9 displays).
  - `expand`: Allows viewport or UI to expand and fill available screen width/height.

---

## 7. Layer Stacking with `CanvasLayer`

- **Layer 1:** Main HUD (health bar, ammo, radar).
- **Layer 5:** Dialogue boxes, subtitles, tooltips.
- **Layer 10:** Pause Menu (`process_mode = PROCESS_MODE_ALWAYS`).
- **Layer 100:** Screen transitions (black fade in/out), loading screens, FPS debugger overlay.
