# ComJoVR App Sequence Overview V2

This version keeps the app-level view readable by splitting the whole app flow into small sequence slices. It is intended for customers or new developers who need the big picture first, then can open [Netcode_Onboarding.md](Netcode_Onboarding.md) for detailed per-feature diagrams.

## How To Read This Version

- Start with `0. App-Level Map` to understand the major systems.
- Read diagrams `1` to `6` in order for the normal app lifecycle.
- Each sequence diagram keeps only the actors needed for that stage.
- Detailed class-level flows remain in [Netcode_Onboarding.md](Netcode_Onboarding.md).

## 0. App-Level Map

```mermaid
%%{init: {"theme": "base","themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 48px !important; }"}}%%
flowchart LR
    User["Quest User"] --> Client["Client App<br/>Menu, Player, Avatar, Tools"]
    Client <-->|"Netcode RPCs and NetworkVariables"| Server["Netcode Server<br/>Approval, Spawn, Authority"]
    Server --> Shared["Shared Network Objects<br/>Model3DManager, LoadXRObjectHelper"]
    Client <-->|"HTTP upload/download"| Files["HTTP File Server<br/>GLB and Audio"]
    Shared --> Content["Runtime Content<br/>3D Models, Parts, Audio Memos"]
    Content -.-> Detail["Detailed diagrams<br/>Netcode_Onboarding.md"]
```

## 1. Server Boot And Shared Object Setup

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "18px"}, "sequence": {"actorFontSize": 18, "messageFontSize": 18, "noteFontSize": 18}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 18px !important; }"}}%%
sequenceDiagram
    participant ServerApp as Server App
    participant NetcodeServer as Netcode Server
    participant RoomManager as RoomManagerServer
    participant SharedObjects as Shared Network Objects
    participant HttpServer as HTTP File Server

    ServerApp->>NetcodeServer: StartServer()
    NetcodeServer-->>RoomManager: OnServerStarted()
    RoomManager->>SharedObjects: Spawn Model3DManager
    RoomManager->>SharedObjects: Spawn LoadXRObjectHelper
    ServerApp->>HttpServer: Start serving GLB/audio files
    SharedObjects-->>NetcodeServer: Ready for client join and runtime sync
```

## 2. Client Join And Player Spawn

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "18px"}, "sequence": {"actorFontSize": 18, "messageFontSize": 18, "noteFontSize": 18}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 18px !important; }"}}%%
sequenceDiagram
    actor User
    participant StartUI as MenuStart
    participant ClientNetwork as Netcode Client
    participant ServerNetwork as Netcode Server
    participant RoomManager as RoomManagerServer
    participant PlayerObject as Network Player

    User->>StartUI: Enter endpoint, name, avatar
    StartUI->>ClientNetwork: StartClient(connection data)
    ClientNetwork->>ServerNetwork: Connection request
    ServerNetwork->>RoomManager: ApprovalCheck(request)
    RoomManager->>RoomManager: Parse and store user data
    RoomManager-->>ServerNetwork: Approve connection
    ServerNetwork-->>RoomManager: OnClientConnected(clientId)
    RoomManager->>PlayerObject: SpawnAsPlayerObject(clientId)
    ServerNetwork-->>ClientNetwork: Replicate spawned player
    ClientNetwork-->>User: Enter room
```

## 3. Player Presence Synchronization

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "18px"}, "sequence": {"actorFontSize": 18, "messageFontSize": 18, "noteFontSize": 18}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 18px !important; }"}}%%
sequenceDiagram
    actor UserA as User A
    participant ClientA as Client A Player
    participant Server as Netcode Server
    participant ClientB as Client B Player View

    UserA->>ClientA: Move headset/controllers
    ClientA->>Server: Sync player transform RPC
    Server-->>ClientB: Replicate transform
    ClientB-->>ClientB: Update remote player pose

    UserA->>ClientA: Update name/avatar state
    ClientA->>Server: Sync presence data
    Server-->>ClientB: Replicate name/avatar
    ClientB-->>ClientB: Refresh remote player display
```

## 4. Shared 3D Model Workflow

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "18px"}, "sequence": {"actorFontSize": 18, "messageFontSize": 18, "noteFontSize": 18}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 18px !important; }"}}%%
sequenceDiagram
    actor User
    participant ClientModel as Client Model UI/System
    participant Server as Netcode Server
    participant ModelState as Model3DManager/Part State
    participant OtherClients as Other Clients
    participant HttpServer as HTTP Model Server

    User->>ClientModel: Switch model, play animation, or edit part
    ClientModel->>Server: Send model action RPC
    Server->>ModelState: Validate and update shared state

    alt Selected model changes
        ModelState->>HttpServer: Resolve selected GLB path
        ModelState-->>OtherClients: Spawn/load selected model
        OtherClients->>HttpServer: Download GLB if needed
    else Animation or part state changes
        ModelState-->>OtherClients: Replicate animation/part state
    end

    OtherClients-->>OtherClients: Apply same visible model state
```

## 5. Audio Memo Workflow

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "28px"}, "sequence": {"actorFontSize": 28, "messageFontSize": 28, "noteFontSize": 28}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 28px !important; }"}}%%
sequenceDiagram
    actor User
    participant ClientMemo as Client Audio Memo System
    participant HttpServer as HTTP Audio Server
    participant Server as Netcode Server
    participant MemoState as MemoAudio State
    participant OtherClients as Other Clients

    User->>ClientMemo: Record or create memo
    ClientMemo->>HttpServer: Upload audio file
    HttpServer-->>ClientMemo: Return file name/path
    ClientMemo->>Server: Send memo create RPC
    Server->>MemoState: Spawn memo and store metadata
    MemoState-->>OtherClients: Replicate memo object
    OtherClients->>HttpServer: Download audio file when needed

    User->>ClientMemo: Move, play, pause, or delete memo
    ClientMemo->>Server: Send memo state RPC
    Server->>MemoState: Update or despawn memo
    MemoState-->>OtherClients: Replicate memo update
```

## 6. Leave Room And Cleanup

```mermaid
%%{init: {"theme": "base", "themeVariables": {"fontSize": "18px"}, "sequence": {"actorFontSize": 18, "messageFontSize": 18, "noteFontSize": 18}, "themeCSS": "text, tspan, .nodeLabel, .edgeLabel, .actor, .messageText, .labelText, .loopText, .noteText { font-size: 18px !important; }"}}%%
sequenceDiagram
    actor User
    participant ClientApp as Client App
    participant ClientNetwork as Netcode Client
    participant ServerNetwork as Netcode Server
    participant RoomManager as RoomManagerServer
    participant OtherClients as Other Clients

    User->>ClientApp: Logout or disconnect
    ClientApp->>ClientNetwork: Shutdown client
    ClientNetwork->>ServerNetwork: Disconnect
    ServerNetwork-->>RoomManager: OnClientDisconnect(clientId)
    RoomManager->>RoomManager: Remove user/session data
    ServerNetwork-->>OtherClients: Despawn departed player
    OtherClients-->>OtherClients: Remove remote avatar/name/memo ownership view
    ClientApp-->>User: Return to start UI
```

## Detail Map

Use this V2 as the customer-facing app overview, then link to the detailed feature diagrams when needed:

- Server boot and join flow: [Netcode_Onboarding.md](Netcode_Onboarding.md), sections `3.1` to `3.5`.
- Player movement, name, and avatar: [Netcode_Onboarding.md](Netcode_Onboarding.md), sections `4.1` to `4.3`.
- 3D model switching, loading, animation, and parts: [Netcode_Onboarding.md](Netcode_Onboarding.md), sections `5.1` to `5.6`.
- Audio memo create, spawn, playback, transform, and delete: [Netcode_Onboarding.md](Netcode_Onboarding.md), sections `6.1` to `6.4`.

## Recommendation

Use this V2 as the main whole-app diagram document. Keep the original [Netcode_App_Sequence_Overview.md](Netcode_App_Sequence_Overview.md) as an internal scratch/complete lifecycle reference only if someone asks to trace everything in one place.
