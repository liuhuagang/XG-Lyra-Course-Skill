# GameInstance and Login Flow

## ULyraGameInstance

[ULyraGameInstance](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraGameInstance.h) inherits from `UCommonGameInstance` and is a cross-map persistent global state manager.

### Responsibilities

1. **User Login Management** — Handle external account login, platform identity authentication
2. **Experience Lifecycle** — Coordinate with `ULyraExperienceManagerComponent`
3. **Session Management** — Create/destroy game sessions
4. **Cross-map Persistent Data** — Player settings, debug console variables

## User Login Flow

```
ULyraGameInstance::BeginLoginAttempt()
  → Check if online subsystem authentication is needed
  → If not needed, directly complete local user creation
  → If needed, call OnlineSubsystem for login
  → After login completes, callback OnUserLoginComplete()
  → Create/bind ULyraLocalPlayer
  → Prepare to enter game world
```

## Token Validation Flow

Lyra uses tokens for user identity validation, with the following flow:

1. **Client requests token** — Client requests an identity token from the online service
2. **Server validates token** — Server validates the token with the online service
3. **Validation passed** — Server allows the client to join the game session
4. **Validation failed** — Server rejects the connection

Token types:
- **Identity Token** — Credential identifying the user
- **Session Token** — Credential identifying the game session
- **Validation Token** — Credential for server-to-server validation

## DTLS Concept

DTLS (Datagram Transport Layer Security) is a TLS encryption protocol over UDP, used by Lyra to protect network transmission.

### Usage in Lyra

- Implemented in the `DTLSHandlerComponent` module
- Used for encrypted communication in multiplayer games
- Prevents man-in-the-middle attacks and data tampering

### DTLS vs TCP/UDP

| Feature | TLS (TCP) | DTLS (UDP) |
|------|-----------|-------------|
| Transport Layer | TCP (reliable) | UDP (unreliable) |
| Encryption | Supported | Supported |
| Packet Loss Handling | Automatic retransmission | Handled by upper layer |
| Latency | Higher (1-RTT handshake) | Lower (0-RTT optional) |

## ULyraGameInstance Debug Tips

View the current GameInstance via console command:

```
showdebug GameInstance
```

Common debug commands:

```
// Toggle Experience reload
ce Experience.Unload
ce Experience.Load {ExperienceName}
```
