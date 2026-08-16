# UI — Sistema de Interface, Layouts e Temas

## Propósito

Construir interfaces responsivas, acessíveis por teclado/gamepad, esteticamente polidas e desacopladas da lógica de gameplay na Godot Engine 4.x.

---

## 1. Contêineres e Hierarquia de Layout

Nunca posicione elementos de interface com coordenadas manuais (`position`). Use **Containers**:

```text
CanvasLayer (Layer = 1)
└── MarginContainer (Anchors: Full Rect | Margens: 24px)
    └── VBoxContainer
        ├── HBoxContainer (Barra Superior)
        │   ├── TextureRect (Ícone do Jogador)
        │   ├── ProgressBar (HealthBar - Size Flag: Expand Fill)
        │   └── Label (Pontuação)
        ├── Control (Espaçador - Size Flag: Expand Fill)
        └── PanelContainer (Caixa de Diálogo / Notificação)
```

### Principais Contêineres
- **`MarginContainer`:** Adiciona margens externas consistentes (padding).
- **`VBoxContainer` / `HBoxContainer`:** Alinha nós vertical ou horizontalmente com espaçamento uniforme (`theme_override_constants/separation`).
- **`GridContainer`:** Organiza itens em colunas fixas (ideal para inventários e lojas).
- **`PanelContainer`:** Desenha uma moldura/fundo com `StyleBox` ao redor do conteúdo interno.
- **`CenterContainer`:** Centraliza menus ou popups no centro da tela.
- **`ScrollContainer`:** Cria áreas de rolagem para textos longos ou listas de itens.

---

## 2. Size Flags (Controle de Expansão)

Nos nós filhos dentro de contêineres:
- **`Shrink Begin / Center / End`:** Ocupa apenas o tamanho mínimo necessário.
- **`Fill`:** Preenche o espaço disponível sem solicitar espaço extra.
- **`Expand + Fill`:** Requisita todo o espaço restante do contêiner pai e se estende nele.

---

## 3. Mouse Filters e Consumo de Input

O comportamento de clique de um `Control` é definido por `mouse_filter`:

| Modo | Efeito | Uso Recomendado |
|---|---|---|
| `STOP` (Padrão) | Consome o evento do mouse; não passa para nós inferiores nem para o jogo. | `Button`, `Slider`, `LineEdit`. |
| `PASS` | Reage ao evento, mas permite que o contêiner pai também receba o evento. | Itens de lista clicáveis. |
| `IGNORE` | Transparente ao mouse; os cliques passam diretamente para o mundo do jogo. | `Label`, `TextureRect` de fundo, `MarginContainer`. |

> [!WARNING]
> Se cliques no jogo não estiverem funcionando, verifique se um `MarginContainer` ou `Control` em tela cheia está configurado com `mouse_filter = STOP`. Mude para `IGNORE`.

---

## 4. Foco e Navegação por Teclado/Gamepad

Para suporte nativo a controles (Xbox, PlayStation) e teclado sem mouse:

```gdscript
extends Control

func _ready() -> void:
    # Garante que o primeiro botão recebe o foco inicial ao abrir o menu
    %StartButton.grab_focus()

func setup_focus_neighbors() -> void:
    # Vinculação explícita de navegação (se a ordem automática falhar)
    %StartButton.focus_neighbor_bottom = %OptionsButton.get_path()
    %OptionsButton.focus_neighbor_top = %StartButton.get_path()
    %OptionsButton.focus_neighbor_bottom = %QuitButton.get_path()
    %QuitButton.focus_neighbor_top = %OptionsButton.get_path()
```

---

## 5. Estilização com `StyleBoxFlat` e Temas

Crie botões e painéis com visual moderno via código ou `.tres`:

```gdscript
func apply_custom_button_style(button: Button) -> void:
    var style_normal := StyleBoxFlat.new()
    style_normal.bg_color = Color("#1e293b") # Slate escuro
    style_normal.corner_radius_top_left = 8
    style_normal.corner_radius_top_right = 8
    style_normal.corner_radius_bottom_left = 8
    style_normal.corner_radius_bottom_right = 8
    style_normal.content_margin_left = 16.0
    style_normal.content_margin_right = 16.0
    
    var style_hover := style_normal.duplicate() as StyleBoxFlat
    style_hover.bg_color = Color("#3b82f6") # Azul destaque
    
    button.add_theme_stylebox_override("normal", style_normal)
    button.add_theme_stylebox_override("hover", style_hover)
    button.add_theme_color_override("font_color", Color.WHITE)
```

---

## 6. Configurações de Multi-Resolução

Em `project.godot > display/window/stretch`:
- **`mode`**:
  - `canvas_items`: Ideal para jogos 2D HD/4K ou 3D (a UI e os vetores são renderizados na resolução nativa do monitor).
  - `viewport`: Ideal para jogos Pixel Art com resolução interna baixa e fixa (ex.: $320 \times 180$).
- **`aspect`**:
  - `keep`: Mantém barras pretas se a proporção de aspecto mudar (ex.: 16:9 em tela 21:9).
  - `expand`: Permite que a viewport ou UI se expanda para preencher a tela inteira.

---

## 7. Estrutura de Camadas com `CanvasLayer`

- **Layer 1:** HUD principal (vida, munição, radar).
- **Layer 5:** Caixas de diálogo e legendas.
- **Layer 10:** Menu de Pausa (`process_mode = PROCESS_MODE_ALWAYS`).
- **Layer 100:** Transições de tela (fade in/out preto), tela de carregamento e FPS debugger.
