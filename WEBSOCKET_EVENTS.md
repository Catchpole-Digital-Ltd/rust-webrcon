# Rust WebSocket RCON Protocol Documentation

This document details the WebSocket RCON protocol used by Rust game servers, independent of any specific client implementation.

## Protocol Overview

Rust servers support a WebSocket-based RCON (Remote Console) protocol that allows clients to send commands and receive server events in real-time using JSON messages.

```
┌─────────────────┐                           ┌─────────────────┐
│                 │  ──── WebSocket ────►     │                 │
│     Client      │                           │   Rust Server   │
│                 │  ◄─── JSON Messages ───   │                 │
└─────────────────┘                           └─────────────────┘
```

---

## Part 1: WebSocket Connection

### Connection URL Format

```
ws://[host]:[port]/[password]
```

| Component | Description |
|-----------|-------------|
| `host` | Server IP address or hostname |
| `port` | RCON port (typically same as game port, e.g., `28015`) |
| `password` | RCON password configured on the server |

**Examples:**
```
ws://192.168.1.100:28015/myPassword
ws://rust.example.com:28016/secretPass
wss://rust.example.com:28016/secretPass  (secure)
```

### Server Configuration

To enable WebSocket RCON on a Rust server, add these startup parameters:

```bash
+rcon.web 1                    # Enable WebSocket RCON
+rcon.password yourpassword    # Set RCON password
+rcon.port 28016               # Optional: custom port
```

---

## Part 2: Message Format

All messages (both sent and received) use this JSON structure:

```json
{
    "Identifier": <number>,
    "Message": "<string>",
    "Type": "<string>",
    "Stacktrace": "<string>"
}
```

### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `Identifier` | Number | Yes | Correlates requests with responses |
| `Message` | String | Yes | Command text or response content |
| `Type` | String | No | Message type (for incoming messages) |
| `Stacktrace` | String | No | Error stacktrace (usually empty) |

### Outgoing Messages (Client → Server)

When sending commands, include:

```json
{
    "Identifier": 1,
    "Message": "your command here",
    "Name": "WebRcon"
}
```

| Field | Description |
|-------|-------------|
| `Identifier` | Your chosen ID to match responses (use values > 1000 for callbacks) |
| `Message` | The RCON command to execute |
| `Name` | Client identifier (conventionally `"WebRcon"`) |

---

## Part 3: Commands and Responses

### Sending a Command

**Request:**
```json
{
    "Identifier": 1001,
    "Message": "status",
    "Name": "WebRcon"
}
```

**Response:**
```json
{
    "Identifier": 1001,
    "Message": "hostname: My Rust Server\nplayers: 45/100...",
    "Type": "Generic",
    "Stacktrace": ""
}
```

### Common RCON Commands

#### `serverinfo` - Get Server Information

**Request:**
```json
{
    "Identifier": 1001,
    "Message": "serverinfo",
    "Name": "WebRcon"
}
```

**Response:**
```json
{
    "Identifier": 1001,
    "Message": "{\"Hostname\":\"My Server\",\"MaxPlayers\":100,\"Players\":45,\"Queued\":0,\"Joining\":2,\"EntityCount\":54321,\"Framerate\":30,\"Memory\":4096,\"Collections\":0,\"NetworkIn\":12345,\"NetworkOut\":67890,\"Restarting\":false,\"SaveCreatedTime\":\"2024-01-15T10:30:00Z\"}",
    "Type": "Generic",
    "Stacktrace": ""
}
```

**Parsed `serverinfo` Response Structure:**

| Field | Type | Description |
|-------|------|-------------|
| `Hostname` | String | Server name |
| `MaxPlayers` | Number | Maximum player slots |
| `Players` | Number | Current player count |
| `Queued` | Number | Players in queue |
| `Joining` | Number | Players currently joining |
| `EntityCount` | Number | Total entities in world |
| `Framerate` | Number | Server FPS |
| `Memory` | Number | Memory usage |
| `NetworkIn` | Number | Inbound network bytes |
| `NetworkOut` | Number | Outbound network bytes |
| `Restarting` | Boolean | Server restart pending |
| `SaveCreatedTime` | String | Last save timestamp |

---

#### `playerlist` - Get Connected Players

**Request:**
```json
{
    "Identifier": 1002,
    "Message": "playerlist",
    "Name": "WebRcon"
}
```

**Response:**
```json
{
    "Identifier": 1002,
    "Message": "[{\"SteamID\":\"76561198000000001\",\"OwnerSteamID\":\"76561198000000001\",\"DisplayName\":\"Player1\",\"Ping\":45,\"Address\":\"192.168.1.50:12345\",\"ConnectedSeconds\":3600,\"VoiationLevel\":0,\"CurrentLevel\":10,\"UnspentXp\":500,\"Health\":100}]",
    "Type": "Generic",
    "Stacktrace": ""
}
```

**Parsed Player Object Structure:**

| Field | Type | Description |
|-------|------|-------------|
| `SteamID` | String | Player's Steam64 ID |
| `OwnerSteamID` | String | Account owner's Steam64 ID (for family sharing) |
| `DisplayName` | String | Player's display name |
| `Ping` | Number | Network latency in milliseconds |
| `Address` | String | Player's IP:port |
| `ConnectedSeconds` | Number | Time connected in seconds |
| `VoiationLevel` | Number | Anti-cheat violation level |
| `CurrentLevel` | Number | Player's XP level |
| `UnspentXp` | Number | Unspent experience points |
| `Health` | Number | Current health |

---

#### `bans` - Get Ban List

**Request:**
```json
{
    "Identifier": 1003,
    "Message": "bans",
    "Name": "WebRcon"
}
```

**Response:**
```json
{
    "Identifier": 1003,
    "Message": "[{\"SteamID\":\"76561198000000099\",\"Username\":\"BadPlayer\",\"Reason\":\"Cheating\",\"Admin\":\"76561198000000001\",\"Expiry\":-1}]",
    "Type": "Generic",
    "Stacktrace": ""
}
```

---

#### `console.tail` - Get Recent Console Output

**Request:**
```json
{
    "Identifier": 1004,
    "Message": "console.tail 100",
    "Name": "WebRcon"
}
```

Returns the last 100 console messages as a JSON array.

---

#### `say` - Send Server Chat Message

**Request:**
```json
{
    "Identifier": 1005,
    "Message": "say Hello everyone!",
    "Name": "WebRcon"
}
```

---

#### Player Management Commands

| Command | Description |
|---------|-------------|
| `kick <steamid/name> [reason]` | Kick a player |
| `ban <steamid/name> [reason]` | Ban a player |
| `unban <steamid>` | Unban a player |
| `banid <steamid> [reason]` | Ban by Steam ID |

---

## Part 4: Server Events (Incoming Messages)

The server pushes events to connected clients without being requested.

### Message Types

| Type | Description |
|------|-------------|
| `Generic` | Standard console output |
| `Log` | Log messages |
| `Warning` | Warning messages |
| `Error` | Error messages |
| `Chat` | Player chat messages |

### Chat Messages

**Incoming:**
```json
{
    "Identifier": 0,
    "Message": "PlayerName: Hello world!",
    "Type": "Chat",
    "Stacktrace": ""
}
```

Chat messages have:
- `Identifier`: `0` (unsolicited event)
- `Type`: `"Chat"`
- `Message`: Format is `"PlayerName: message content"`

---

### Console Output Events

**Incoming:**
```json
{
    "Identifier": 0,
    "Message": "[event] 76561198000000001/Player1 was killed by Wolf",
    "Type": "Generic",
    "Stacktrace": ""
}
```

Common console event prefixes:
- `[event]` - Game events (kills, deaths, etc.)
- `[EAC]` - EasyAntiCheat notifications
- `[RCON]` - RCON-related messages
- Player connection/disconnection messages

---

### Player Connection Events

**Player Joined:**
```json
{
    "Identifier": 0,
    "Message": "76561198000000001/Player1 joined [192.168.1.50:12345]",
    "Type": "Generic",
    "Stacktrace": ""
}
```

**Player Disconnected:**
```json
{
    "Identifier": 0,
    "Message": "76561198000000001/Player1 disconnecting: disconnect",
    "Type": "Generic",
    "Stacktrace": ""
}
```

---

### Error Events

```json
{
    "Identifier": 0,
    "Message": "NullReferenceException: Object reference not set...",
    "Type": "Error",
    "Stacktrace": "at Server.DoSomething()..."
}
```

---

## Part 5: Identifier System

The `Identifier` field is used to correlate requests with responses.

### Identifier Conventions

| Range | Purpose |
|-------|---------|
| `-1` | Default/untracked commands |
| `0` | Server-initiated events (chat, logs, etc.) |
| `1-1000` | Reserved for simple request types |
| `> 1000` | Callback-based requests (recommended) |

### Request-Response Correlation

```
Client sends:     { "Identifier": 1001, "Message": "status" }
                           │
                           ▼
Server responds:  { "Identifier": 1001, "Message": "..." }
                           │
                           ▼
Client matches Identifier 1001 to original request
```

For callback-based implementations, use an incrementing counter starting above 1000:

```javascript
let nextId = 1001;

function sendCommand(command, callback) {
    const id = nextId++;
    callbacks[id] = callback;
    socket.send(JSON.stringify({
        Identifier: id,
        Message: command,
        Name: "WebRcon"
    }));
}
```

---

## Part 6: WebSocket Lifecycle Events

### Standard WebSocket Events

| Event | Description |
|-------|-------------|
| `onopen` | Connection established successfully |
| `onclose` | Connection closed |
| `onerror` | Connection error occurred |
| `onmessage` | Message received from server |

### Connection Close Codes

| Code | Meaning |
|------|---------|
| `1000` | Normal closure |
| `1001` | Server going away (shutdown/restart) |
| `1006` | Abnormal closure (network issue) |
| `1008` | Policy violation (authentication failed) |

---

## Part 7: Implementation Example (Framework-Agnostic)

### Minimal JavaScript Client

```javascript
class RconClient {
    constructor() {
        this.socket = null;
        this.callbacks = {};
        this.nextId = 1001;
    }

    connect(host, port, password) {
        const url = `ws://${host}:${port}/${password}`;
        this.socket = new WebSocket(url);

        this.socket.onopen = () => {
            console.log('Connected');
        };

        this.socket.onclose = (event) => {
            console.log('Disconnected:', event.code);
        };

        this.socket.onerror = (error) => {
            console.error('Error:', error);
        };

        this.socket.onmessage = (event) => {
            const msg = JSON.parse(event.data);
            this.handleMessage(msg);
        };
    }

    handleMessage(msg) {
        // Check if this is a response to a request
        if (msg.Identifier > 1000 && this.callbacks[msg.Identifier]) {
            this.callbacks[msg.Identifier](msg);
            delete this.callbacks[msg.Identifier];
            return;
        }

        // Handle server-initiated events
        switch (msg.Type) {
            case 'Chat':
                console.log('[Chat]', msg.Message);
                break;
            case 'Warning':
                console.warn('[Warning]', msg.Message);
                break;
            case 'Error':
                console.error('[Error]', msg.Message);
                break;
            default:
                console.log('[Console]', msg.Message);
        }
    }

    command(cmd, callback) {
        const id = callback ? this.nextId++ : -1;

        if (callback) {
            this.callbacks[id] = callback;
        }

        this.socket.send(JSON.stringify({
            Identifier: id,
            Message: cmd,
            Name: 'WebRcon'
        }));
    }

    getServerInfo(callback) {
        this.command('serverinfo', (msg) => {
            callback(JSON.parse(msg.Message));
        });
    }

    getPlayers(callback) {
        this.command('playerlist', (msg) => {
            callback(JSON.parse(msg.Message));
        });
    }

    disconnect() {
        if (this.socket) {
            this.socket.close();
        }
    }
}

// Usage
const client = new RconClient();
client.connect('192.168.1.100', 28015, 'password');

client.getPlayers((players) => {
    console.log('Players online:', players.length);
});
```

---

## Part 8: Security Considerations

| Concern | Details |
|---------|---------|
| **Password in URL** | RCON password is part of the WebSocket URL path |
| **No Encryption** | `ws://` is unencrypted; use `wss://` when available |
| **Server Exposure** | RCON port should be firewalled to trusted IPs only |
| **Command Injection** | Validate/sanitize any user input before sending commands |

---

## Appendix: Quick Reference

### Outgoing Message Template

```json
{
    "Identifier": <number>,
    "Message": "<command>",
    "Name": "WebRcon"
}
```

### Incoming Message Template

```json
{
    "Identifier": <number>,
    "Message": "<content>",
    "Type": "<Generic|Log|Warning|Error|Chat>",
    "Stacktrace": "<string>"
}
```

### Common Commands

| Command | Returns |
|---------|---------|
| `serverinfo` | Server stats (JSON) |
| `playerlist` | Connected players (JSON array) |
| `bans` | Ban list (JSON array) |
| `console.tail N` | Last N console messages |
| `say <message>` | Broadcasts chat message |
| `kick <id> [reason]` | Kicks player |
| `ban <id> [reason]` | Bans player |
| `status` | Server status text |

---

## Sources

- [Facepunch webrcon Repository](https://github.com/Facepunch/webrcon)
- [MrGraversen/rust-rcon Java Client](https://github.com/MrGraversen/rust-rcon)
