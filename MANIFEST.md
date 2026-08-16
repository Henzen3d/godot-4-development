# Bundle Manifest

**Version:** 2.1  
**Updated:** 2026-08-16  
**Target Engine:** Godot Engine 4.x (GDScript 2.0 / .NET)

---

## Package Structure

- `SKILL.md` — Main router and essential operations manual.
- `references/gdscript-4.x.md` — Complete guide to GDScript 2.0, static typing, annotations, Callables, lambdas, and Custom Resources.
- `references/scenes-and-nodes.md` — Node architecture, scene trees, "Call Down, Signal Up" rule, lifecycle, and Signal Bus.
- `references/physics-2d-3d.md` — 2D/3D physics simulation (`CharacterBody`), movement, collisions, raycasts, and navigation.
- `references/ui.md` — Godot 4 UI system, Containers, Anchor Presets, Size Flags, themes, and gamepad/keyboard focus navigation.
- `references/animation.md` — `AnimationPlayer`, `AnimationTree` (StateMachine/BlendTree), SpriteFrames, and transitions with `Tween`.
- `references/audio.md` — Audio system, audio buses, `AudioStreamRandomizer`, pooling, and sound feedback.
- `references/shaders.md` — 2D (`canvas_item`) and 3D (`spatial`) shaders, uniforms, screen-reading textures, and practical effects.
- `references/networking.md` — High-level multiplayer, scene synchronization, annotated RPCs, and HTTP integration.
- `references/godot-mcp-tools.md` — Operational manual for integrating with Godot MCP servers (discovery, headless, bridge, and runtime).

---

## Design Philosophy

1. **Agile Routing:** `SKILL.md` remains lean and decision-oriented, loading detailed reference files on-demand to optimize token and context usage.
2. **Production-Ready Code:** All code recipes use static typing in GDScript 2.0, modern annotations, and idiomatic Godot 4.x patterns.
3. **Decoupling & Modularity:** Continuous emphasis on Custom Resources, global Signal Buses, and composition over inheritance.

---

## Provenance and MCP Compatibility

This bundle is designed to operate resiliently across multiple Godot MCP environments:
- **`tugcantopaloglu/godot-mcp`**: Comprehensive suite of 157 tools (project management, headless scenes, nodes, scripts, UI, audio, physics, and runtime bridge).
- **`IvanMurzak/Godot-MCP` (Godot Asset Library #5245)**: C# plugin featuring 42 tools organized into 12 functional families for the Godot 4.3+ editor.

**Discovery Policy:** The AI agent should never assume the presence of a hardcoded MCP tool name without first inspecting the schema exposed by the active agent/Godot MCP session.

---

## Security and Validation Policy

- **Path Traversal Prevention:** All file operations must sanitize paths, ensuring they remain strictly within the `res://` scope or project directory.
- **Multi-Level Validation:** A code change is only considered complete after:
  1. *Static Validation:* No syntax or type errors;
  2. *Scene Validation:* Integrity of nodes and resource references in `.tscn`;
  3. *Runtime Validation:* Behavior verified visually or via debugger logs when supported by the environment.
