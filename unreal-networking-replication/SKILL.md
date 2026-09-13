---
name: unreal-networking-replication
description: "Multiplayer networking, replication, RPC calls, net role logic, server/client authority, prediction, synchronizing game state."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - networking
  - replication
---

# Unreal Networking Replication

## Trigger Words

Use this skill when the user mentions:
- "networking"
- "replication"
- "networking replication"
- "replication graph"
- "spatial culling"

## Context Check

Read `.agents/unreal-project-context.md` for this project's multiplayer configuration.
Look for: server topology (dedicated, listen, P2P), player count, replicated classes,
and any custom net drivers.

If the context file is absent, ask:
1. Server topology? (dedicated, listen, P2P)
2. Maximum player count per session?
3. Which actors or components need to replicate data?
4. Are you using Gameplay Ability System (GAS)?
5. Are you using a custom Replication Graph? (for 30+ players)

---

## Net Roles and Authority

UE uses a server-authoritative model: the server is the source of truth for game state.
Clients predict locally and reconcile with server corrections.

Every actor on every machine has a local role and a remote role (`ENetRole`).

```
ROLE_Authority       — owns and can modify this actor (server for replicated actors)
ROLE_AutonomousProxy — client copy of the locally controlled pawn
ROLE_SimulatedProxy  — client copy of another player's actor; engine interpolates state
ROLE_None            — not replicated
```

From `Actor.h`:
```cpp
ENetRole GetLocalRole() const { return Role; }   // role on current machine
ENetRole GetRemoteRole() const;                   // role the other end sees
bool HasAuthority() const { return (GetLocalRole() == ROLE_Authority); }
```

Net modes: `NM_Standalone`, `NM_DedicatedServer`, `NM_ListenServer`, `NM_Client`.

**Role matrix for a replicated Pawn:**

| Machine        | GetLocalRole()       | GetRemoteRole()                         |
|----------------|----------------------|-----------------------------------------|
| Server         | ROLE_Authority       | ROLE_AutonomousProxy or SimulatedProxy  |
| Owning Client  | ROLE_AutonomousProxy | ROLE_Authority                          |
| Other Clients  | ROLE_SimulatedProxy  | ROLE_Authority                          |

**Listen-server caveat:** the host is both `ROLE_Authority` and locally controlled.
Use `IsLocallyControlled()` to distinguish logic that should skip the host player.

---

## Common Task: Setting Up Actor Replication

When asked to set up actor replication, follow this sequence:

1. Read the target actor/character header file to check existing declarations
2. Read the corresponding .cpp file to see current implementation
3. Edit files to add:
   - `bReplicates = true` in constructor
   - `SetReplicateMovement(true)` if needed
   - `SetNetUpdateFrequency(10.f)` and `SetMinNetUpdateFrequency(2.f)`
   - `NetPriority = 1.0f`
   - `Replicated` or `ReplicatedUsing` specifiers on UPROPERTY declarations
   - `GetLifetimeReplicatedProps` override in .cpp
4. Verify changes compile

Required tools: `Read`, `Edit`, `Write` (in that order)

---

## UNetDriver

`UNetDriver` is the core transport class responsible for managing all network connections and
packet delivery for a world. It owns the list of `UNetConnection` objects and drives the
replication tick. Access it via `UWorld::GetNetDriver()`.

```cpp
UNetDriver* Driver = GetWorld()->GetNetDriver();
// Driver->ClientConnections  — all connected clients (server-side)
// Driver->ServerConnection   — connection to server (client-side)
```

For most gameplay code you never interact with `UNetDriver` directly; it is relevant when
writing custom net drivers, profiling connection state, or debugging packet loss.

---

## Property Replication

### Actor Setup

```cpp
AMyActor::AMyActor()
{
    bReplicates = true;             // AActor::SetReplicates() also available at runtime
    SetReplicateMovement(true);     // replicates FRepMovement (location/rotation/velocity)
    SetNetUpdateFrequency(10.f);    // checks per second
    SetMinNetUpdateFrequency(2.f);  // floor when nothing changes
    NetPriority = 1.0f;            // higher = preferred when bandwidth is saturated
}
```

From `Actor.h`: `SetReplicates`, `SetReplicateMovement`, `SetNetUpdateFrequency`,
`SetMinNetUpdateFrequency`, and `SetNetCullDistanceSquared` are all `ENGINE_API`.

**FRepMovement:** when `bReplicateMovement = true`, the engine automatically replicates
location, rotation, linear velocity, and angular velocity via `FRepMovement`. You can
customize which components are replicated with `SetReplicatedMovement(RepMovement)`.

### Declaring Properties

```cpp
UPROPERTY(Replicated)
int32 Health;

UPROPERTY(ReplicatedUsing = OnRep_State)
EMyState State;

UFUNCTION()
void OnRep_State(EMyState PreviousState); // old value passed as optional parameter
```

### GetLifetimeReplicatedProps

```cpp
// MyActor.cpp
#include "Net/UnrealNetwork.h"

void AMyActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); // NEVER omit this
    DOREPLIFETIME(AMyActor, Health);
    DOREPLIFETIME_CONDITION(AMyActor, State,        COND_OwnerOnly);
    DOREPLIFETIME_CONDITION(AMyActor, SimData,      COND_SimulatedOnly);
    DOREPLIFETIME_CONDITION(AMyActor, InitData,     COND_InitialOnly);
    DOREPLIFETIME_CONDITION(AMyActor, PublicData,   COND_SkipOwner);
}
```

**Conditions:** `COND_None` (all), `COND_OwnerOnly`, `COND_SkipOwner`,
`COND_SimulatedOnly`, `COND_AutonomousOnly`, `COND_InitialOnly`, `COND_Custom`.

Use `COND_OwnerOnly` for private player data (inventory, currency).
Use `COND_InitialOnly` for immutable spawn data (team, character class).

**Initial replication burst**: When a client first joins or an actor first becomes relevant, ALL replicated properties send at once regardless of conditions (`COND_InitialOnly` fires exactly once here). This burst can saturate the actor channel — keep initial state compact and use `COND_InitialOnly` for spawn-time-only data to reduce ongoing bandwidth.

### Example: Full Property Replication Implementation

```cpp
// In header file (.h)
UPROPERTY(Replicated)
float Health;

UPROPERTY(ReplicatedUsing = OnRep_State)
ECharacterState State;

// ReplicatedUsing requires a UFUNCTION for the callback
UFUNCTION()
void OnRep_State();
```

```cpp
// Example implementation in .cpp file
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Replicate to all clients
    DOREPLIFETIME(AMyCharacter, Health);
    DOREPLIFETIME(AMyCharacter, State);

    // For conditional replication:
    // DOREPLIFETIME_CONDITION(AMyCharacter, Ammo, COND_OwnerOnly);
}
```

### FRepLayout (Internal)

`FRepLayout` is an internal engine struct that describes which properties of a class are
replicated and how.

---

## Replication Graphs (Scalable Networking)

For games with 30+ players

---

## RPC vs Replication vs SaveGame Boundaries

> 边界原则：RPC 只做意图传输、replication 做权威状态扇出、SaveGame 做磁盘持久化；客户端本地存档不作多人权威输入。

Three mechanisms exist for moving game state; each has exactly one job:

| Mechanism | Role | Anti-pattern |
|---|---|---|
| **RPC** | Intent transfer only — "the client wants to do X" | Using RPC as persistent state storage, or passing full state payloads instead of intent |
| **Replication** | Authoritative state fan-out — server state pushed to clients via `DOREPLIFETIME` / `ReplicatedUsing` | Replicating data that is also redundantly re-sent through RPCs |
| **SaveGame** | Disk persistence — durable state across sessions | Persisting transient network-only caches |

Guidelines:
- Use `ReplicatedUsing` when client-side side effects are required (UI refresh, cosmetic reconstruction) — but `OnRep_*` reconstructs/refreshes client view, it never performs authority writes.
- Use RPC only for intent transfer; let the server mutate state and replication broadcast the result.
- **Never trust client-local save data as authoritative multiplayer input.**

---

## Server-Authoritative State Flow

Canonical flow for any state-changing action:

1. Client → server RPC: send the *intent* (what the player wants to do), not the resulting state.
2. Server validates the payload (ownership, cooldowns, legality of the target, anti-cheat checks) before mutating anything.
3. Server applies the mutation to authoritative replicated state.
4. Result propagates via replication/NetMulticast to all clients (broadcast only when needed).

Failure symptoms and fixes:
- Client can trigger unauthorized state changes → RPC validation/authority checks missing. Fix: enforce server validation and reject invalid client payloads.
- Replicated value differs from saved value after reconnect → authority write path vs restore timing mismatch. Fix: restore on authority first, then let replication propagate to clients.

---

## Client-Local Saves Are Not Authoritative

- A client-local SaveGame (loaded and applied entirely on the client) must never be treated as authoritative input in multiplayer.
- Client-saved data can be tampered with; validate any client-reported persisted state on the server before it affects shared gameplay state.
- Restore client-local data only as a *request* for state, routed through the same validated server RPC entry points as any other client action.

## Tools

This skill primarily uses:
- `Read` - to examine existing code and project context
- `Edit` - to modify existing files (preferred over Write for targeted changes)
- `Write` - to create new files when needed
- `Bash` - to run Unreal Build Tool or check compilation

Always read the target file before editing it.