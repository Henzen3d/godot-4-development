# Bundle Manifest

**Version:** 2.1  
**Updated:** 2026-08-16  
**Target Engine:** Godot Engine 4.x (GDScript 2.0 / .NET)

---

## Estrutura do Pacote

- `SKILL.md` — Roteador principal e manual de operações essenciais.
- `references/gdscript-4.x.md` — Guia completo de GDScript 2.0, tipagem estática, anotações, Callables, lambdas e Resources.
- `references/scenes-and-nodes.md` — Arquitetura de nós, árvores de cenas, regra "Call Down, Signal Up", ciclo de vida e Signal Bus.
- `references/physics-2d-3d.md` — Simulação física 2D/3D (`CharacterBody`), movimento, colisões, raycasts e navegação.
- `references/ui.md` — Sistema de UI do Godot 4, Containers, Anchor Presets, Size Flags, temas e foco por controle/teclado.
- `references/animation.md` — `AnimationPlayer`, `AnimationTree` (StateMachine/BlendTree), SpriteFrames e transições com `Tween`.
- `references/audio.md` — Sistema de áudio, barramentos (Buses), `AudioStreamRandomizer`, pooling e feedback sonoro.
- `references/shaders.md` — Shaders em 2D (`canvas_item`) e 3D (`spatial`), uniforms, texturas de tela e efeitos práticos.
- `references/networking.md` — Multiplayer de alto nível, sincronização de cenas, RPCs anotados e integração HTTP.
- `references/godot-mcp-tools.md` — Manual operacional de integração com servidores Godot MCP (descoberta, headless, bridge e runtime).

---

## Filosofia de Design

1. **Roteamento Ágil:** `SKILL.md` mantém-se enxuto e orientado a decisões, carregando referências detalhadas apenas sob demanda para economia de contexto.
2. **Código de Produção:** Todas as receitas de código usam tipagem estática do GDScript 2.0, anotações modernas e padrões idiomáticos do Godot 4.x.
3. **Desacoplamento e Modularidade:** Incentivo contínuo ao uso de Custom Resources, Signal Buses globais e composição sobre herança.

---

## Proveniência e Compatibilidade MCP

Este pacote foi desenhado para operar de forma resiliente em múltiplos ambientes Godot MCP:
- **`tugcantopaloglu/godot-mcp`**: Conjunto amplo de 157 ferramentas (gerenciamento de projeto, cenas headless, nodes, scripts, UI, áudio, física e ponte de runtime).
- **`IvanMurzak/Godot-MCP` (Godot Asset Library #5245)**: Plugin C# de 42 ferramentas organizadas em 12 famílias funcionais para o editor Godot 4.3+.

**Política de Descoberta:** O agente nunca deve assumir a existência de um nome estático de ferramenta MCP sem antes inspecionar o schema exposto pela sessão ativa do Hermes/Godot MCP.

---

## Política de Segurança e Validação

- **Prevenção de Path Traversal:** Todas as manipulações de arquivo devem sanitizar caminhos garantindo que permaneçam no escopo de `res://` ou do diretório do projeto.
- **Validação Multinível:** Uma alteração só é considerada concluída após:
  1. *Validação Estática:* Ausência de erros de sintaxe ou tipos;
  2. *Validação de Cena:* Integridade dos nós e referências no `.tscn`;
  3. *Validação em Runtime:* Comportamento verificado visualmente ou via logs quando o ambiente permitir.
