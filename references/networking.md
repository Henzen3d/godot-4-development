# Networking — Multiplayer, RPCs, MultiplayerSpawner e HTTP

## Propósito

Fornecer padrões modernos para multiplayer de alto nível e requisições Web/HTTP na Godot Engine 4.x.

---

## 1. Arquitetura Multiplayer do Godot 4

O Godot 4 introduziu um sistema declarativo de nós para replicação de estado e nós dinâmicos:

```mermaid
graph LR
    Server[Servidor (Autoridade: ID 1)] <--> PeerA[Cliente A (ID: 1024)]
    Server <--> PeerB[Cliente B (ID: 2048)]
    
    subgraph Sincronizacao Declarativa
        MultiplayerSpawner[MultiplayerSpawner: Replica criacao/destruicao de entidades]
        MultiplayerSynchronizer[MultiplayerSynchronizer: Replica propriedades continuas ex: posicoes]
    end
```

---

## 2. Configuração de Servidor e Cliente (ENet)

```gdscript
extends Node

const DEFAULT_PORT: int = 8910
const MAX_CLIENTS: int = 4

func start_server() -> void:
    var peer := ENetMultiplayerPeer.new()
    var error := peer.create_server(DEFAULT_PORT, MAX_CLIENTS)
    if error != OK:
        push_error("Falha ao iniciar servidor: %d" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    multiplayer.peer_connected.connect(_on_peer_connected)
    multiplayer.peer_disconnected.connect(_on_peer_disconnected)
    print("Servidor iniciado na porta ", DEFAULT_PORT)

func join_server(address: String = "127.0.0.1") -> void:
    var peer := ENetMultiplayerPeer.new()
    var error := peer.create_client(address, DEFAULT_PORT)
    if error != OK:
        push_error("Falha ao conectar: %d" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    multiplayer.connected_to_server.connect(func(): print("Conectado com sucesso!"))
    multiplayer.connection_failed.connect(func(): push_error("Falha na conexão"))

func _on_peer_connected(id: int) -> void:
    print("Cliente conectado: ", id)

func _on_peer_disconnected(id: int) -> void:
    print("Cliente desconectado: ", id)
```

---

## 3. Anotações RPC no Godot 4

No Godot 4, utilize a anotação `@rpc(...)` com permissões explícitas:

```gdscript
# Apenas a autoridade do nó pode chamar; executado em todos os peers + máquina local de forma confiável
@rpc("authority", "call_local", "reliable")
func update_game_state(new_state: String) -> void:
    print("Novo estado do jogo: ", new_state)

# Qualquer cliente pode chamar no servidor (ex.: enviar input ou ação)
@rpc("any_peer", "call_remote", "unreliable_ordered")
func send_input_to_server(direction: Vector2) -> void:
    var sender_id := multiplayer.get_remote_sender_id()
    if multiplayer.is_server():
        process_player_input(sender_id, direction)
```

### Modos de RPC:
- **Autoridade:** `"authority"` (padrão) vs `"any_peer"`.
- **Execução:** `"call_remote"` (apenas nos outros) vs `"call_local"` (remoto + local).
- **Transferência:** `"reliable"` (TCP-like, garantido) vs `"unreliable"` / `"unreliable_ordered"` (UDP-like, rápido, para posições contínuas).

---

## 4. `MultiplayerSpawner` e `MultiplayerSynchronizer`

1. **`MultiplayerSpawner`:**
   - Aponte a propriedade `spawn_path` para o nó que conterá os jogadores (ex.: `../PlayersContainer`).
   - Adicione a cena `res://scenes/player.tscn` à lista `Auto Spawn List`.
   - Quando o servidor chamar `players_container.add_child(player, true)`, a cena será instanciada e configurada automaticamente em todos os clientes conectados.

2. **`MultiplayerSynchronizer`:**
   - Adicione como filho do `Player`.
   - No Inspector, configure as propriedades a sincronizar (ex.: `:position`, `:velocity`, `:rotation`).
   - Configure a taxa de envio (ex.: 20 Hz / 50ms) para economizar banda.

---

## 5. Integração com APIs Web e REST (`HTTPRequest`)

```gdscript
extends Node

@onready var http_request: HTTPRequest = %HTTPRequest

func fetch_leaderboard() -> void:
    http_request.request_completed.connect(_on_request_completed, CONNECT_ONE_SHOT)
    var headers: PackedStringArray = ["Content-Type: application/json"]
    var error := http_request.request("https://api.meujogo.com/scores", headers, HTTPClient.METHOD_GET)
    if error != OK:
        push_error("Erro ao enviar HTTP request")

func _on_request_completed(_result: int, response_code: int, _headers: PackedStringArray, body: PackedByteArray) -> void:
    if response_code == 200:
        var json_string := body.get_string_from_utf8()
        var json_data: Variant = JSON.parse_string(json_string)
        if json_data is Array:
            print("Leaderboard recebido: ", json_data)
    else:
        push_error("HTTP Error: %d" % response_code)
```
