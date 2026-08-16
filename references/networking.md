# Networking — Multiplayer, RPCs, MultiplayerSpawner & HTTP

## Purpose

Provide modern patterns for high-level multiplayer and Web/REST API integration in Godot Engine 4.x.

---

## 1. Godot 4 Multiplayer Architecture

Godot 4 introduces declarative node systems for state synchronization and dynamic scene spawning:

```mermaid
graph LR
    Server[Server (Authority: ID 1)] <--> PeerA[Client A (ID: 1024)]
    Server <--> PeerB[Client B (ID: 2048)]
    
    subgraph Declarative Sync
        MultiplayerSpawner[MultiplayerSpawner: Replicates spawning/despawning of entities]
        MultiplayerSynchronizer[MultiplayerSynchronizer: Replicates continuous properties e.g. transforms]
    end
```

---

## 2. Server and Client Setup (ENet)

```gdscript
extends Node

const DEFAULT_PORT: int = 8910
const MAX_CLIENTS: int = 4

func start_server() -> void:
    var peer := ENetMultiplayerPeer.new()
    var error := peer.create_server(DEFAULT_PORT, MAX_CLIENTS)
    if error != OK:
        push_error("Failed to start server: %d" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    multiplayer.peer_connected.connect(_on_peer_connected)
    multiplayer.peer_disconnected.connect(_on_peer_disconnected)
    print("Server started on port ", DEFAULT_PORT)

func join_server(address: String = "127.0.0.1") -> void:
    var peer := ENetMultiplayerPeer.new()
    var error := peer.create_client(address, DEFAULT_PORT)
    if error != OK:
        push_error("Failed to connect: %d" % error)
        return
    
    multiplayer.multiplayer_peer = peer
    multiplayer.connected_to_server.connect(func(): print("Connected successfully!"))
    multiplayer.connection_failed.connect(func(): push_error("Connection failed"))

func _on_peer_connected(id: int) -> void:
    print("Client connected: ", id)

func _on_peer_disconnected(id: int) -> void:
    print("Client disconnected: ", id)
```

---

## 3. RPC Annotations in Godot 4

In Godot 4, use the `@rpc(...)` annotation with explicit permissions:

```gdscript
# Only node authority can call; executed on all peers + local machine reliably
@rpc("authority", "call_local", "reliable")
func update_game_state(new_state: String) -> void:
    print("New game state: ", new_state)

# Any client can invoke on server (e.g., player input/actions)
@rpc("any_peer", "call_remote", "unreliable_ordered")
func send_input_to_server(direction: Vector2) -> void:
    var sender_id := multiplayer.get_remote_sender_id()
    if multiplayer.is_server():
        process_player_input(sender_id, direction)
```

### RPC Modes:
- **Authority:** `"authority"` (default) vs `"any_peer"`.
- **Execution:** `"call_remote"` (remotes only) vs `"call_local"` (remotes + local caller).
- **Transfer Mode:** `"reliable"` (TCP-like, guaranteed) vs `"unreliable"` / `"unreliable_ordered"` (UDP-like, fast, for continuous transforms).

---

## 4. `MultiplayerSpawner` and `MultiplayerSynchronizer`

1. **`MultiplayerSpawner`:**
   - Set `spawn_path` to the container holding dynamic entities (e.g., `../PlayersContainer`).
   - Add `res://scenes/player.tscn` to the `Auto Spawn List`.
   - When the server calls `players_container.add_child(player, true)`, the scene is automatically instantiated and replicated across all connected peers.

2. **`MultiplayerSynchronizer`:**
   - Add as child of `Player`.
   - In Inspector, define the properties to synchronize (e.g., `:position`, `:velocity`, `:rotation`).
   - Configure sync rate (e.g., 20 Hz / 50ms) to preserve bandwidth.

---

## 5. Web & REST API Integration (`HTTPRequest`)

```gdscript
extends Node

@onready var http_request: HTTPRequest = %HTTPRequest

func fetch_leaderboard() -> void:
    http_request.request_completed.connect(_on_request_completed, CONNECT_ONE_SHOT)
    var headers: PackedStringArray = ["Content-Type: application/json"]
    var error := http_request.request("https://api.mygame.com/scores", headers, HTTPClient.METHOD_GET)
    if error != OK:
        push_error("Error sending HTTP request")

func _on_request_completed(_result: int, response_code: int, _headers: PackedStringArray, body: PackedByteArray) -> void:
    if response_code == 200:
        var json_string := body.get_string_from_utf8()
        var json_data: Variant = JSON.parse_string(json_string)
        if json_data is Array:
            print("Leaderboard received: ", json_data)
    else:
        push_error("HTTP Error: %d" % response_code)
```
