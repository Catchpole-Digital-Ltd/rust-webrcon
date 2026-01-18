# WebSocket Events Documentation

This document provides a comprehensive guide to how rust-webrcon handles WebSocket communication for game server remote console (RCON) functionality.

## Overview

rust-webrcon is a browser-based RCON client that uses WebSocket connections to communicate with game servers (primarily Rust servers). The application is built with AngularJS and runs entirely in the browser with no server backend required.

## Architecture

```
┌─────────────────┐         WebSocket         ┌─────────────────┐
│                 │  ◄────────────────────►   │                 │
│  Browser Client │                           │   Game Server   │
│   (webrcon)     │   JSON Messages           │   (Rust RCON)   │
│                 │                           │                 │
└─────────────────┘                           └─────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `RconService` | Core WebSocket service managing connection lifecycle |
| `RconController` | Main application orchestrator and event router |
| `ConnectionController` | Handles connection form and server history |
| `ConsoleController` | Manages command input/output display |
| `PlayersController` | Displays and refreshes player list |

## Connection Establishment

### WebSocket URL Format

```
ws://[server_address]:[port]/[password]
```

**Example:**
```
ws://192.168.1.100:28015/mySecurePassword
```

### Connection Code

The connection is established in the `RconService`:

```javascript
s.Connect = function(addr, pass) {
    s.Socket = new WebSocket("ws://" + addr + "/" + pass);

    s.Socket.onmessage = function(e) { ... };
    s.Socket.onopen = s.OnOpen;
    s.Socket.onclose = s.OnClose;
    s.Socket.onerror = s.OnError;
}
```

### Connection Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| Server Address | Yes | IP address and port (e.g., `192.168.1.100:28015`) |
| Password | Yes | RCON password configured on the server |

## WebSocket Lifecycle Events

### 1. `onopen` - Connection Established

Triggered when the WebSocket connection is successfully established.

**Handler:**
```javascript
rconService.OnOpen = function() {
    $scope.Connected = true;
    $scope.$broadcast("OnConnected");
    $scope.$digest();
}
```

**Behavior:**
- Sets the application connection state to `true`
- Broadcasts `OnConnected` event to all controllers
- Triggers automatic player list refresh in `PlayersController`

### 2. `onclose` - Connection Closed

Triggered when the WebSocket connection is closed (either by client, server, or network).

**Handler:**
```javascript
rconService.OnClose = function(ev) {
    $scope.$broadcast("OnDisconnected", ev);
    $scope.$digest();
}
```

**Behavior:**
- Broadcasts `OnDisconnected` event with closure details
- Displays error message: `"Connection was closed - Error [code]"`
- Resets UI to disconnected state

**Common Close Codes:**
| Code | Meaning |
|------|---------|
| 1000 | Normal closure |
| 1001 | Going away (server shutdown) |
| 1006 | Abnormal closure (network issue) |
| 1008 | Policy violation (authentication failure) |

### 3. `onerror` - Connection Error

Triggered when a WebSocket error occurs.

**Handler:**
```javascript
rconService.OnError = function(ev) {
    $scope.$broadcast("OnConnectionError", ev);
    $scope.$digest();
}
```

**Behavior:**
- Broadcasts `OnConnectionError` event
- Typically followed by `onclose` event
- Used for error reporting and UI feedback

### 4. `onmessage` - Message Received

Triggered when a message is received from the server.

**Handler:**
```javascript
s.Socket.onmessage = function(e) {
    if (s.OnMessage != null) {
        s.OnMessage(JSON.parse(e.data));
    }
};
```

**Behavior:**
- Parses incoming JSON data
- Routes message to appropriate controller via callback
- Broadcasts `OnMessage` event for processing

## Message Protocol

### Message Format

All messages exchanged between client and server use a standardized JSON structure:

```json
{
    "Identifier": <number>,
    "Message": "<string>",
    "Name": "<string>"
}
```

### Message Fields

| Field | Type | Description |
|-------|------|-------------|
| `Identifier` | Number | Numeric ID to correlate requests with responses |
| `Message` | String | Command text or response data |
| `Name` | String | Client identifier (always `"WebRcon"` for outgoing) |

### Identifier Values

| Identifier | Purpose |
|------------|---------|
| `-1` | Default/system messages |
| `1` | Console commands and responses |
| `2` | Player list queries |
| `> 1` | Custom command types (filtered from console) |

## Sending Commands

### Command Function

```javascript
s.Command = function(msg, identifier) {
    if (identifier == null)
        identifier = -1;

    var packet = {
        Identifier: identifier,
        Message: msg,
        Name: "WebRcon"
    };

    s.Socket.send(JSON.stringify(packet));
};
```

### Usage Examples

**Send a console command:**
```javascript
rconService.Command("status", 1);
```

**Request player list:**
```javascript
rconService.Command("playerlist", 2);
```

**Send a chat message:**
```javascript
rconService.Command("say Hello everyone!", 1);
```

## Message Flow Examples

### Console Command Flow

```
1. User enters command in console
      │
      ▼
2. ConsoleController.SubmitCommand()
      │
      ▼
3. rconService.Command(text, 1)
      │
      ▼
4. JSON packet sent: {"Identifier": 1, "Message": "command", "Name": "WebRcon"}
      │
      ▼
5. Server processes command
      │
      ▼
6. Server sends response
      │
      ▼
7. Client receives message, parses JSON
      │
      ▼
8. ConsoleController filters by Identifier=1
      │
      ▼
9. Output displayed in console view
```

### Player List Flow

```
1. PlayersController.Refresh() or OnConnected event
      │
      ▼
2. rconService.Command("playerlist", 2)
      │
      ▼
3. Server responds with player data (Identifier=2)
      │
      ▼
4. Response parsed: JSON.parse(msg.Message)
      │
      ▼
5. Player array displayed in UI
```

## Message Types and Examples

### Outgoing: Console Command

```json
{
    "Identifier": 1,
    "Message": "status",
    "Name": "WebRcon"
}
```

### Incoming: Console Response

```json
{
    "Identifier": 1,
    "Message": "Server is running. Players: 15/100",
    "Name": "server"
}
```

### Incoming: Chat Message

```json
{
    "Identifier": 0,
    "Message": "[CHAT] PlayerName: Hello everyone!",
    "Name": "server"
}
```

### Outgoing: Player List Request

```json
{
    "Identifier": 2,
    "Message": "playerlist",
    "Name": "WebRcon"
}
```

### Incoming: Player List Response

```json
{
    "Identifier": 2,
    "Message": "[{\"DisplayName\":\"Player1\",\"SteamID\":\"76561198...\",\"CurrentLevel\":10,\"UnspentXp\":500,\"Ping\":45,\"ConnectedSeconds\":3600}]",
    "Name": "server"
}
```

## Player Data Structure

The player list response contains a JSON array with player objects:

```json
{
    "DisplayName": "PlayerName",
    "SteamID": "76561198000000000",
    "CurrentLevel": 10,
    "UnspentXp": 500,
    "Ping": 45,
    "ConnectedSeconds": 3600
}
```

| Field | Type | Description |
|-------|------|-------------|
| `DisplayName` | String | Player's display name |
| `SteamID` | String | Player's Steam ID |
| `CurrentLevel` | Number | Player's current level |
| `UnspentXp` | Number | Unspent experience points |
| `Ping` | Number | Network latency in milliseconds |
| `ConnectedSeconds` | Number | Time connected in seconds |

## Internal Event Broadcasting

The application uses AngularJS's `$broadcast` mechanism to propagate events:

| Event | Triggered When | Data |
|-------|---------------|------|
| `OnConnected` | WebSocket connection established | None |
| `OnDisconnected` | WebSocket connection closed | Close event object |
| `OnConnectionError` | WebSocket error occurs | Error event object |
| `OnMessage` | Message received from server | Parsed message object |

### Subscribing to Events

Controllers listen for events using `$on`:

```javascript
$scope.$on("OnConnected", function() {
    // Handle connection established
});

$scope.$on("OnMessage", function(event, msg) {
    // Handle incoming message
});
```

## Console Message Styling

Messages are styled based on their content and identifier:

| Condition | Style |
|-----------|-------|
| Contains `[CHAT]` prefix | Green color (chat messages) |
| `Identifier === 1` | Blue color (requested commands) |
| `Identifier > 1` | Filtered/hidden from console |
| Other messages | Default styling |

## Configuration and Storage

### localStorage Usage

Connection history is persisted in browser localStorage:

```javascript
// Retrieve previous connections
if (localStorage.previousConnections != null)
    $scope.PreviousConnects = angular.fromJson(localStorage.previousConnections);

// Save connection
localStorage.previousConnections = angular.toJson($scope.PreviousConnects);
```

### Stored Data

| Key | Content |
|-----|---------|
| `previousConnections` | Array of server addresses and passwords |

## Server Configuration (Rust)

To enable WebSocket RCON on a Rust server:

```bash
# Enable web-based RCON
+rcon.web 1

# Set RCON password
+rcon.password yourpassword

# Optional: Set custom port (default is server port)
+rcon.port 28016
```

## Security Considerations

1. **Password in URL**: The RCON password is embedded in the WebSocket URL path
2. **Local Storage**: Connection credentials are stored in browser localStorage (never sent to remote servers)
3. **Shareable URLs**: Shared URLs contain server address but NOT the password
4. **No Encryption**: Standard `ws://` connections are unencrypted; use `wss://` for secure connections if supported

## Error Handling

### Connection Failures

Common causes and solutions:

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| Connection refused | Server not running or wrong port | Verify server is running and port is correct |
| Connection closed (1008) | Invalid password | Check RCON password |
| Connection timeout | Firewall blocking | Check firewall rules for RCON port |
| Connection closed (1006) | Network interruption | Check network connectivity |

### Reconnection

The application does not automatically reconnect. Users must manually reconnect after a disconnection.
