---
name: unreal-serialization-savegames
description: "USaveGame save/load, player progress persistence, FArchive serialization. NOT for UGameUserSettings, UDeveloperSettings, config/game settings."
author: Sx
version: 1.1.0
keywords:
  - unreal
  - serialization
  - savegames
  - USaveGame
  - gameplay save
  - player progress
  - checkpoint
  - inventory save
  - NOT settings
  - NOT config
  - NOT UGameUserSettings
---

# Unreal Serialization Savegames

## Trigger Words

Use this skill when the user mentions:
- "serialization"
- "savegames"
- "serialization savegames"

## Step 0: Exclusion Check — Is this about game settings?

If the user's request involves:
- `UGameUserSettings` or `UDeveloperSettings` subclasses
- `Config` / `DefaultEngine.ini` / `DefaultGame.ini` modifications
- Graphics, audio, input, or other engine-level settings
- Configuration files or .ini parsing

Then this skill is NOT applicable. Instead, use the **unreal-config-settings** skill (or inform the user that no save-game skill applies).

Only proceed if the request is about **player progress persistence** using `USaveGame`, `UGameplayStatics::SaveGameToSlot`, or custom `FArchive` serialization of gameplay data.

---

## Step 1: Read Project Context

Read `.agents/unreal-project-context.md` before giving any recommendations. You need:
- Engine version (UE 5.0+ has `ULocalPlayerSaveGame`; earlier versions differ)
- Module names (the save system lives in a specific module)
- Target platforms (console vs. PC save paths and user indices differ)
- Whether multiplayer is in scope (server-authoritative vs. client-local saves)

IMPORTANT: You MUST use the `Read` tool to read `.agents/unreal-project-context.md`. 
Use the exact format: `<read><path>.agents/unreal-project-context.md</path></read>`

Example of correct usage:
<read><path>.agents/unreal-project-context.md</path></read>

Do NOT use:
<read>.agents/unreal-project-context.md</read>
or
<read><path>.agents/unreal-project-context.md</path><path>...</path></read>

Note: The `Read` tool requires the path to be inside a `<path>` child element of `<read>`. Using `<read>path</read>` or any other format will fail. Always use `<read><path>your-path-here</path></read>`.

If the `Read` tool call fails or returns an error, do NOT proceed with implementation. Instead:
1. Inform the user that the project context file could not be read.
2. Ask the user to verify the file exists at `.agents/unreal-project-context.md`.
3. Offer to create a placeholder file if the user confirms it's missing.
4. Wait for the user to resolve the issue before continuing.

Do NOT skip this step or proceed without project context.

---

## Step 2: Gather Requirements

Ask before writing code:
1. **Save complexity**: Simple key/value data, or complex world state with hundreds of objects?
2. **Data types**: Primitives, nested structs, asset references (soft vs. hard)?
3. **Versioning needs**: Live game with future patches? Old saves must keep working?
4. **Multiple save slots**: How many? Does each player/user get their own?
5. **Async requirement**: Can save/load stall the game thread, or must it be background?

---

## Step 3: USaveGame Subclass

`USaveGame` is an abstract `UObject` from `GameFramework/SaveGame.h`. Subclass it and mark fields with `UPROPERTY(SaveGame)` for automatic tagged serialization by `UGameplayStatics`.

```cpp
// MyGameSaveGame.h
#pragma once
#include "CoreMinimal.h"
#include "GameFramework/SaveGame.h"
#include "MyGameSaveGame.generated.h"

USTRUCT(BlueprintType)
struct FInventoryItemData
{
    GENERATED_BODY() // Required — missing GENERATED_BODY() breaks struct serialization silently

    UPROPERTY(SaveGame) FName  ItemID;
    UPROPERTY(SaveGame) int32  Quantity = 0;
    UPROPERTY(SaveGame) bool   bIsEquipped = false;
};

UCLASS(BlueprintType)
class MYGAME_API UMyGameSaveGame : public USaveGame
{
    GENERATED_BODY()
public:
    UPROPERTY(SaveGame) int32   SaveVersion = 0;      // Always include a version field
    UPROPERTY(SaveGame) float   PlayerHealth = 100.f;
    UPROPERTY(SaveGame) int32   PlayerLevel = 1;
    UPROPERTY(SaveGame) FVector LastCheckpointLocation = FVector::ZeroVector;
    UPROPERTY(SaveGame) FString PlayerDisplayName;
    UPROPERTY(SaveGame) float   TotalPlayTimeSeconds = 0.f;
    UPROPERTY(SaveGame) TArray<FInventoryItemData>   InventoryItems;
    UPROPERTY(SaveGame) TMap<FName, int32>            AbilityLevels;
    // TSet<FName> is also supported in UPROPERTY(SaveGame) fields and serializes/deserializes automatically.

    // Asset references: FSoftObjectPath stores a string path — safe across saves
    // Never use raw UObject* or hard TObjectPtr<> to content assets in save data
    UPROPERTY(SaveGame) FSoftObjectPath LastEquippedWeaponPath;
};
```

### Saving and Loading

```cpp
#include "Kismet/GameplayStatics.h"

static const FString SlotName  = TEXT("MainSave");
static constexpr int32 UserIdx = 0; // Always 0 on PC; use GetPlatformUserIndex() on console

// Create the object first, populate its fields, then save
UMySaveGame* SaveGame = Cast<UMySaveGame>(UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
SaveGame->PlayerHealth = 75.f;
// Then pass SaveGame to SaveGameToSlot / AsyncSaveGameToSlot below

// Sync save (blocks game thread — avoid in gameplay)
bool bSaved = UGameplayStatics::SaveGameToSlot(SaveGame, SlotName, UserIdx);

// Async save (preferred — runs on background thread)
UGameplayStatics::AsyncSaveGameToSlot(SaveGame, SlotName, UserIdx,
    FAsyncSaveGameToSlotDelegate::CreateLambda([](const FString& Slot, int32 User, bool bSuccess)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Save completed for slot %s"), *Slot);
        }
        else
        {
            UE_LOG(LogTemp, Error, TEXT("Save failed for slot %s"), *Slot);
        }
    }));

// Loading
UMySaveGame* LoadedGame = Cast<UMySaveGame>(UGameplayStatics::LoadGameFromSlot(SlotName, UserIdx));
if (LoadedGame)
{
    // Use LoadedGame data...
}
```

---

## Step 4: ULocalPlayerSa

---

## State Tri-Partitioning (Persist / Replicate / Both)

Before designing any save schema, classify every state field into exactly one of three buckets:

| Classification | Persisted to disk | Replicated over network | Typical examples |
|---|---|---|---|
| **Persistent-only** | Yes | No | settings preferences, cosmetic unlocks, local UI state |
| **Replicated-only** | No | Yes | transient runtime sync, live positions, in-progress combat state |
| **Persistent + replicated (dual)** | Yes | Yes | authoritative world state that must survive restart (inventory, currency, unlocked levels) |

Rules:
- Mark each field explicitly — leaving classification implicit invites drift where a field is added to one pipeline but forgotten in the other.
- Replicated-only fields must NEVER be written into the SaveGame (avoid persisting transient network-only caches).
- Dual fields are the tricky ones: the same logical value has two pipelines, so their write paths must be coordinated (restore on authority first, then replication propagates to clients).
- Partition by ownership: local player, world/shared, match/session. Ephemeral runtime values split from persistent values.

---

## Schema Versioning Contract

Every persisted schema MUST define:
- **Explicit version integer** — a `UPROPERTY(SaveGame) int32 SaveVersion` field, written at save time and checked at load time.
- **Stable identifiers/keys** — actors, components, and inventory-like entries keyed by stable IDs/FNames, never by array index or implicit ordering. Index-based implicit ordering breaks when entries are added/removed between patches.
- **Migration behavior** — a defined path for older versions: either a per-version migration function that upgrades old data in place, or a clean gate that rejects unsupported versions with an explicit reason.

Save pipeline contract:
- Gather runtime state into the save schema in deterministic order.
- Write through sync or async slot APIs based on latency sensitivity.
- Return a structured save result (success, failure reason, version used).

Load pipeline contract:
- Load the save object, validate the version, run migration if needed.
- Apply restore in dependency-safe order (owners before dependents).
- Resolve missing assets/entities with an explicit fallback policy.

---

## Restore Ordering and Idempotent Retry

- Apply restore in staged, dependency-safe order: **owners first, dependents second**. A dependent that references an owner's state must not be restored before its owner exists.
- Keep restore **idempotent**: re-running the restore after a partial failure must be safe. Stage restore as transactions — if a stage fails, roll back or re-run cleanly rather than leaving half-applied state.
- On partial failure, clean up created artifacts (orphaned actors/components) instead of leaving them.
- When a replicated value differs from the saved value after reconnect: restore on the authority first, then let replication propagate to clients — never restore directly into client-side replicated properties.

---

## Async Callback Safety (State Tokens)

Async save/load callbacks race with gameplay transitions (level travel, actor teardown, new session start). Guard every async callback:

- Capture a **state token** (a monotonically increasing counter or a weak/generated identifier captured at call time) in the lambda, and verify it still matches the current state before applying the result.
- Check **world validity** (`GetWorld()` non-null, world context still alive) and object validity (`IsValid()`) inside the callback before dereferencing anything.
- Stale callbacks must no-op cleanly: ignore the result, never write into a torn-down object or a new session's state.