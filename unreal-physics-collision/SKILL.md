---
name: unreal-physics-collision
description: "Collision/physics/trace in Unreal. Triggers: 'LineTrace','overlap','sweep','collision channel','physics body','Chaos','raytrace','OnHit','OnBeginOverlap'."
author: Sx
version: 1.1.0
keywords:
  - unreal
  - physics
  - collision
---

# Unreal Physics Collision

## Trigger Words

Use this skill when the user mentions:
- "physics"
- "collision"
- "physics collision"

## Always Produce Code

When the user asks for implementation code, you MUST write the complete C++ or Blueprint code in your response. Do not stop after reading context files. If context is missing, use defaults (UE 5.3, Chaos, ECC_GameTraceChannel2 for custom channels) and note assumptions in comments. Always include: header declarations, source implementation, and component setup.

## Step 1: Read Project Context

Read `.agents/unreal-project-context.md` to confirm:
- UE version (Chaos is the default physics backend from UE 5.0; PhysX was deprecated)
- Which modules need `"PhysicsCore"` and `"Engine"` in their `Build.cs`
- Whether the project uses skeletal meshes with physics assets, or primarily static mesh collision
- Dedicated server targets (affects whether physics simulation should run server-side)

## Step 1b: Proceed to Implementation

After reading project context, immediately proceed to implement the requested feature. Do not stop after reading context. If the user asks for code, write the complete implementation with:
- Header declarations (delegates, component pointers)
- Setup code (component creation, binding delegates)
- Filtering logic (channel checks, tag checks)
- Event handler stubs

If context is missing or unclear, make reasonable default assumptions (UE 5.x, Chaos physics) and note them in comments.

---

## Step 2: Identify the Need

Ask which area applies if not stated:
1. **Collision setup** — channels, profiles, responses on components
2. **Trace queries** — line traces, sweeps, overlap queries for gameplay logic
3. **Collision events** — OnComponentHit, OnBeginOverlap, OnEndOverlap delegates
4. **Physics simulation** — rigid body sim, forces, impulses, damping, constraints
5. **Physical materials** — friction, restitution, surface type detection

---

## Collision Channels & Profiles

### ECollisionChannel — built-in channels

```cpp
ECC_WorldStatic, ECC_WorldDynamic, ECC_Pawn, ECC_PhysicsBody,
ECC_Vehicle, ECC_Destructible               // object channels (what an object IS)
ECC_Visibility, ECC_Camera                  // trace channels (used for queries)
// Custom: ECC_GameTraceChannel1..ECC_GameTraceChannel18
```

**Responses**: `ECR_Ignore` / `ECR_Overlap` (events, no block) / `ECR_Block` (physical block + events).

**Built-in profiles**: `BlockAll`, `BlockAllDynamic`, `OverlapAll`, `OverlapAllDynamic`, `Pawn`, `PhysicsActor`, `NoCollision`.

### Setting Collision in C++

```cpp
MyMesh->SetCollisionProfileName(TEXT("BlockAll"));        // preferred — sets all at once
MyMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
// ECollisionEnabled: NoCollision | QueryOnly | PhysicsOnly | QueryAndPhysics
MyMesh->SetCollisionObjectType(ECC_PhysicsBody);
MyMesh->SetCollisionResponseToAllChannels(ECR_Block);
MyMesh->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);
MyMesh->SetCollisionResponseToChannel(ECC_Camera, ECR_Ignore);
```

### Object Type Channels vs Trace Channels

**Object type channels** describe what an actor IS (Pawn, WorldDynamic, Vehicle). Every component has exactly one object type. **Trace channels** are used for queries — they define what a trace is LOOKING FOR (Visibility, Camera, Weapon). This distinction determines which query function to use: `ByObjectType` matches the target's object type channel; `ByChannel` uses the querier's trace channel and checks responses. Most gameplay traces use trace channels (`ECC_Visibility`, custom `Weapon`); overlap queries for "find all pawns" use object type (`ECC_Pawn`).

### Custom Channels — DefaultEngine.ini

```ini
[/Script/Engine.CollisionProfile]
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel1,DefaultResponse=ECR_Block,bTraceType=True,bStaticObject=False,Name="Weapon")
+DefaultChannelResponses=(Channel=ECC_GameTraceChannel2,DefaultResponse=ECR_Block,bTraceType=False,bStaticObject=False,Name="Interactable")
+Profiles=(Name="Interactable",CollisionEnabled=QueryAndPhysics,ObjectTypeName="Interactable",CustomResponses=((Channel="Weapon",Response=ECR_Ignore),(Channel="Visibility",Response=ECR_Block)))
```

`bTraceType=True` = trace channel; `bTraceType=False` = object type channel. They use separate query functions.

See `references/collision-channel-setup.md` for full profile examples.

---

## Collision Event Binding

### C++ Delegate Binding

```cpp
// In header (.h)
UFUNCTION()
void OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, 
                    UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                    bool bFromSweep, const FHitResult& SweepResult);

UFUNCTION()
void OnOverlapEnd(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, 
                  UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

// In source (.cpp) - Setup
SphereComponent = CreateDefaultSubobject<USphereComponent>(TEXT("PickupSphere"));
SphereComponent->InitSphereRadius(200.0f);
SphereComponent->SetCollisionProfileName(TEXT("OverlapAll"));
SphereComponent->SetGenerateOverlapEvents(true);
SphereComponent->OnComponentBeginOverlap.AddDynamic(this, &AYourClass::OnOverlapBegin);
SphereComponent->OnComponentEndOverlap.AddDynamic(this, &AYourClass::OnOverlapEnd);

// Filter by custom channel
SphereComponent->SetCollisionResponseToChannel(ECC_GameTraceChannel2, ECR_Overlap); // Interactable
SphereComponent->SetCollisionResponseToAllChannels(ECR_Ignore);
SphereComponent->SetCollisionResponseToChannel(ECC_GameTraceChannel2, ECR_Overlap);

// Event handler example
void AYourClass::OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, 
                                UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, 
                                bool bFromSweep, const FHitResult& SweepResult)
{
    if (!OtherActor || OtherActor == this) return;
    // Check object type or channel if needed
    UPrimitiveComponent* PrimComp = Cast<UPrimitiveComponent>(OtherComp);
    if (PrimComp && PrimComp->GetCollisionObjectType() == ECC_GameTraceChannel2)
    {
        // Handle pickup overlap
    }
}
```

**Blueprint equivalent**: Bind OnComponentBeginOverlap/OnComponentEndOverlap events on the Sphere component. Use Branch + Get Collision Object Type to filter.

---

## Trace Queries

### FCollisionQueryParams

```cpp
FCollisionQueryParams Params;
Params.TraceTag                = TEXT("WeaponTrace"); // for profiling/debug
Params.bTraceComplex           = false;  // false=simple hull (fast); true=per-poly (expensive)
Params.bReturnPhysicalMaterial = true;   // populates Hit.PhysMaterial
Params.bReturnFaceIndex        = false;  // expensive, only when needed
Params.AddIgnoredActor(this);
Params.AddIgnoredComponent(MyComp);
```

### World-Level Trace Functions (C++) — from `WorldCollision.h` via `UWorld`

```cpp
FHitResult Hit;
// By trace channel
GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params);
GetWorld()->LineTraceMultiByChannel(Hits, Start, End, ECC_Visibility, Params);
// By object type
FCollisionObjectQueryParams ObjParams(ECC_PhysicsBody);
ObjParams.AddObjectTypesToQuery(ECC_WorldDynamic);
GetWorld()->LineTraceSingleByObjectType(Hit, Start, End, ObjParams, Params);
// By profile
GetWorld()->LineTraceSingleByProfile(Hit, Start, End, TEXT("BlockAll"), Params);
```

### Sweep Queries — FCollisionShape (from `CollisionShape.h`)

```cpp
FCollisionShape Sphere  = FCollisionShape::MakeSphere(30.f);
FCollisionShape Box     = FCollisionShape::MakeBox(FVector(50.f, 50.f, 50.f));
FCollisionShape Capsule = FCollisionShape::MakeCapsule(34.f, 88.f); // radius, half-height

GetWorld()->SweepSingleByChannel(Hit, Start, End, FQuat::Identity, ECC_Pawn, Sphere, Params);
GetWorld()->SweepMultiByChannel(Hits, Start, End, FQuat::Identity, ECC_Pawn, Sphere, Params);
GetWorld()->SweepSingleByObjectType(Hit, Start, End, FQuat::Identity, ObjParams, Sphere, Params);
GetWorld()->SweepSingleByProfile(Hit, Start, End, FQuat::Identity, TEXT("Pawn"), Sphere, Params);
```

### Overlap Queries

```cpp
TArray<FOverlapResult> Overlaps;
GetWorld()->OverlapMultiByObjectType(Overlaps, Center, FQuat::Identity,
    FCollisionObjectQueryParams(ECC_Pawn), FCollisionShape::MakeSphere(500.f), Params);
for (const FOverlapResult& R : Overlaps)

---

## Interaction Systems (拾取/生成器/交互状态机)

Layer on top of the collision primitives above: pickups, spawners, overlap/trace-driven interactions, and use-key actors. Covers the interaction state machine, eligibility, server authority, feedback, and lifecycle.

### Interaction Stage Contract

Every interaction feature must define:

- Detection source (overlap/trace/use-key)
- Eligibility checks (distance, tags, inventory/capacity, authority)
- Success and failure result payloads
- Feedback path (VFX/SFX/UI)
- Post-interaction lifecycle policy (destroy/disable/cooldown/respawn)

If any item is missing, the interaction behavior is underspecified.

### Detection Source

- Overlap model: bind `OnComponentBeginOverlap`/`OnComponentEndOverlap` on the collision component.
- Trace model: run line/sphere sweeps on input request and validate the hit actor/component.
- Keep detection deterministic with explicit collision channels/profiles; keep overlap and trace paths functionally equivalent when both are enabled.
- Align collision shape and bounds with the intended interaction distance.

### State Machine: active / cooldown / consumed

- Define the default states (`active`, `cooldown`, `consumed`) and their replication needs up front.
- Keep state stable on failure; mutate state only on confirmed success.

### Eligibility and Structured Failure

- Validate target state, range, actor validity, and gameplay conditions (inventory/capacity) before resolution.
- Return a structured failure reason on rejection so failure feedback can be reason-specific.

### Server Authority

- In multiplayer, validate success on the server before mutating any shared state.
- Replicate the authoritative result state, not raw input spam.
- Use client-side prediction only for cosmetic feedback when acceptable.
- Keep cooldown/timer ownership on the authority side for deterministic multiplayer behavior.

### Feedback Path

- Emit VFX/SFX/UI feedback for both success and failure paths.
- Separate cosmetic-only effects from authoritative gameplay mutations.
- Keep feedback idempotent for repeated client updates.
- Do not destroy actors before broadcasting the required success/failure feedback.

### Lifecycle and Cleanup

- Choose lifecycle explicitly: destroy, hide + disable collision, or reuse via respawn timer.
- If a spawner exists, track spawned instances and clean them up on reset/despawn.
- Disable tick for dormant interaction actors when not needed.

### Tick Gating and Reentrancy Guards

- Prevent re-entrant execution during in-progress resolution: add a state guard before processing and ensure a single delegate binding per component.
- Gate traces by input/distance; reduce polling frequency; disable idle tick to avoid frame spikes near dense interaction areas.

### Interaction Failure Handling

- Overlap never triggers → check collision enabled state, overlap flags, channel responses, component bounds.
- Trace path misses obvious targets → align trace channel/profile; reduce self/owner ignore misuse.
- Interaction triggers multiple times → add a state guard; ensure single delegate binding per component.
- Client sees success but server rejects → move final validation to the server and replicate the authoritative result.
- Consumed pickup remains interactable → disable collision/interaction immediately after confirmed success.
- Spawned interactables leak over time → track spawned instances; destroy/disable stale instances on the respawn cycle.
- Frame spikes near dense interaction areas → reduce polling frequency, gate traces by input/distance, disable idle tick.