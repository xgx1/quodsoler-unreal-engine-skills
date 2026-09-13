---
name: unreal-procedural-generation
description: "Unreal procedural generation: PCG framework, ProceduralMesh, instanced mesh, HISM, spline, runtime mesh, noise, terrain, dungeon generation."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - procedural
  - generation
---

# Unreal Procedural Generation

## Trigger Words

Use this skill when the user mentions:
- "procedural"
- "generation"
- "procedural generation"

## Context Check

Before advising, read `.agents/unreal-project-context.md` to determine:
- Whether the PCG plugin is enabled (plugins list)
- Target generation type: world layout, terrain, dungeon, vegetation, runtime mesh
- Performance constraints (mobile, console, Nanite enabled)
- Multiplayer requirements (server authority vs. deterministic seeding)

## Information Gathering

Ask for clarification on:
1. **Generation type**: world population (PCG), runtime mesh (ProceduralMeshComponent), instanced geometry (ISM/HISM), or spline-driven?
2. **Timing**: editor-time baked result or runtime dynamic generation?
3. **Instance count**: hundreds (ISM) or tens of thousands (HISM)?
4. **Collision**: does generated geometry need physics collision?
5. **Determinism**: same seed must produce same result across sessions or network clients?

---

## 1. PCG Framework (UE 5.2+)

Node-based rule-driven world generation. Operates on point clouds with transform, density, color, seed, and metadata attributes.

### Plugin Setup

```csharp
// Build.cs
PublicDependencyModuleNames.Add("PCG");
```
```json
// .uproject Plugins array
{ "Name": "PCG", "Enabled": true }
```

### Core Classes

| Class | Header | Purpose |
|---|---|---|
| `UPCGComponent` | `PCGComponent.h` | Actor component driving generation |
| `UPCGGraph` | `PCGGraph.h` | Asset: nodes + edges |
| `UPCGGraphInstance` | `PCGGraph.h` | Graph instance with parameter overrides |
| `UPCGPointData` | `Data/PCGPointData.h` | Point cloud between nodes |
| `UPCGSettings` | `PCGSettings.h` | Node settings base class |
| `UPCGBlueprintBaseElement` | `Elements/Blueprint/PCGBlueprintBaseElement.h` | Custom Blueprint node base |

### UPCGComponent Key API (from `PCGComponent.h`)

```cpp
// Assign graph (NetMulticast)
void SetGraph(UPCGGraphInterface* InGraph);

// Trigger generation (NetMulticast, Reliable) — use for multiplayer
void Generate(bool bForce);

// Local non-replicated generation
void GenerateLocal(bool bForce);

// Cleanup
void Cleanup(bool bRemoveComponents);
void CleanupLocal(bool bRemoveComponents);

// Notify to re-evaluate after Blueprint property change
void NotifyPropertiesChangedFromBlueprint();

// Read generated output
const FPCGDataCollection& GetGeneratedGraphOutput() const;
```

Generation triggers (`EPCGComponentGenerationTrigger`):
- `GenerateOnLoad` — one-shot on BeginPlay
- `GenerateOnDemand` — explicit `Generate()` call only
- `GenerateAtRuntime` — budget-scheduled by `UPCGSubsystem`

### UPCGGraph Node API (from `PCGGraph.h`)

```cpp
// Add node by settings class
UPCGNode* AddNodeOfType(TSubclassOf<UPCGSettings> InSettingsClass, UPCGSettings*& DefaultNodeSettings);

// Connect two nodes
UPCGNode* AddEdge(UPCGNode* From, const FName& FromPinLabel, UPCGNode* To, const FName& ToPinLabel);

// Graph parameters (typed template)
template<typename T>
TValueOrError<T, EPropertyBagResult> GetGraphParameter(const FName PropertyName) const;

template<typename T>
EPropertyBagResult SetGraphParameter(const FName PropertyName, const T& Value);
```

### Custom Blueprint PCG Node

Derive from `UPCGBlueprintBaseElement`:

```cpp
UCLASS(BlueprintType, Blueprintable)
class UMyPCGNode : public UPCGBlueprintBaseElement
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "PCG|Execution")
    void Execute(const FPCGDataCollection& Input, FPCGDataCollection& Output);
};

// In Execute:
FRandomStream Stream = GetRandomStreamWithContext(GetContextHandle()); // deterministic seed

for (const FPCGTaggedData& In : Input.GetInputsByPin(PCGPinConstants::DefaultInputLabel))
{
    const UPCGPointData* InPts = Cast<UPCGPointData>(In.Data);
    if (!InPts) continue;

    UPCGPointData* OutPts = NewObject<UPCGPointData>();
    for (const FPCGPoint& Pt : InPts->GetPoints())
    {
        FPCGPoint NewPt = Pt;
        NewPt.Density = Stream.FRandRange(0.5f, 1.0f);
        OutPts->GetMutablePoints().Add(NewPt);
    }
    Output.TaggedData.Emplace_GetRef().Data = OutPts;
}
```

Key `UPCGBlueprintBaseElement` properties:
- `bIsCacheable = false` — when node spawns actors or components
- `bRequiresGameThread = true` — for actor spawn, component add
- `CustomInputPins` / `CustomOutputPins` — extra typed pins

### PCG Determinism

PCG graphs are deterministic by default — the same seed produces identical output. Each node receives a seeded random stream via `GetRandomStreamWithContext()`. To vary output across instances, set the PCG component's `Seed` property. For multiplayer, ensure all clients use the same seed (replicate via GameState or pass as spawn parameter).

```cpp
// Set PCG seed at runtime for deterministic variation
UPCGComponent* PCG = FindComponentByClass<UPCGComponent>();
PCG->Seed = MyDeterministicSeedValue;
PCG->Generate(); // Regenerate with new seed
```

### PCG Data Types

| Type | Contains | Use for |
|---|---|---|
| `FPCGPoint` / Point Data | Position, rotation, scale, density, color | Scatter placement, foliage, instance positioning |
| `UPCGSplineData` | Spline points + tangents | Roads, rivers, paths, boundary definitions |
| `UPCGLandscapeData` | Height + layer weight sampling | Terrain-aware placement, biome queries |
| `UPCGVolumeData` | 3D bounds | Volume-based filtering and generation |

Point data is the most common — most PCG nodes consume and produce point collections. Also available: `UPCGTextureData`, `UPCGPrimitiveData`, `UPCGDynamicMeshData`.

See `references/pcg-node-reference.md` for all node types, settings fields, and pin labels.

---

## 2. ProceduralMeshComponent

```csharp
// Build.cs
PublicDependencyModuleNames.Add("ProceduralMeshComponent");
```

### Core API

```cpp
// Create section: vertices, triangles (CCW = front), normals, UVs, colors, tangents
void CreateMeshSection(int32 SectionIndex,
    const TArray<FVector>& Vertices, const TArray<int32>& Triangles,
    const TArray<FVector>& Normals,  const TArray<FVector2D>& UV0,
    const TArray<FColor>& VertexColors, const TArray<FProcMeshTangent>& Tangents,
    bool bCreateCollision);

// Updates vertex positions (incl. collision if enabled). Cannot change topology.
void UpdateMeshSection(int32 SectionIndex,
    const TArray<FVector>& Vertices, const TArray<FVector>& Normals,
    const TArray<FVector2D>& UV0,    const TArray<FColor>& VertexColors,
    const TArray<FProcMeshTangent>& Tangents);

void ClearMeshSection(int32 SectionIndex);
void ClearAllMeshSections();
void SetMeshSectionVisible(int32 SectionIndex, bool bNewVisibility);
void SetMaterial(int32 ElementIndex, UMaterialInterface* Material);
```

### Terrain Grid Example

```cpp
void ATerrainActor::Build(int32 Grid, float Cell)
{
    TArray<FVector> Verts; TArray<int32> Tris; TArray<FVector> Norms;
    TArray<FVector2D> UVs; TArray<FColor> Colors; TArray<FProcMeshTangent> Tangs;

    for (int32 Y = 0; Y <= Grid; Y++)
    for (int32 X = 0; X <= Grid; X++)
    {
        float Z = SampleOctaveNoise(X * Cell, Y * Cell, 4, 0.5f, 2.f, 80.f);
        Verts.Add(FVector(X * Cell, Y * Cell, Z));
        Norms.Add(FVector::UpVector);
        UVs.Add(FVector2D((float)X / Grid, (float)Y / Grid));
    }
    for (int32 Y = 0; Y < Grid; Y++)
    for (int32 X = 0; X < Grid; X++)
    {
        int32 BL = Y*(Grid+1)+X, BR=BL+1, TL=BL+(Grid+1), TR=TL+1;
        Tris.Add(BL); Tris.Add(TL); Tris.Add(TR);
        Tris.Add(BL); Tris.Add(TR); Tris.Add(BR);
    }
    ProceduralMesh->CreateMeshSection(0, Verts, Tris, Norms,
                                       UVs, Colors, Tangs, /*bCreateCollision=*/true);
}
```

### Performance Notes

- One draw call per `CreateMeshSection`. Keep vertex count < 65K per section.
- `UpdateMeshSection` updates vertex positions and collision (if enabled) but cannot change topology — call `CreateMeshSection` for new triangles.
- ProceduralMesh does **not** support Nanite.
- Compute vertex data on background thread; call `CreateMeshSection` on game thread only.

### Async Mesh Generation

Generate vertices on a background thread, then apply on the game thread:

```cpp
// Background task — compute vertices
class FMeshGenTask : public FNonAbandonableTask
{
public:
    TArray<FVector> Vertices;
    TArray<int32> Triangles;
    void DoWork() { /* Marching cubes, noise sampling, etc. */ }
    FORCEINLINE TStatId GetStatId() const { RETURN_QUICK_DECLARE_CYCLE_STAT(FMeshGenTask, STATGROUP_ThreadPoolAsyncTasks); }
};

// Launch and poll
// Use FAsyncTask (not FAutoDeleteAsyncTask) when polling IsDone() is needed.
// FAutoDeleteAsyncTask deletes itself on completion — calling IsDone() afterward is a use-after-free.
auto* Task = new FAsyncTask<FMeshGenTask>();
Task->StartBackgroundTask();
// Poll safely: if (Task->IsDone()) { /* use Task->GetTask().Vertices */ delete Task; }
```

### Collision on Procedural Meshes

Set `UProceduralMeshComponent::bUseComplexAsSimpleCollision = true` to use the rendered triangles directly for collision. This is accurate but expensive — only use for static geometry. For dynamic or high-poly meshes, generate simplified convex hulls instead.

---

## 3. Instanced Static Meshes (ISM / HISM)

| Feature | ISM (`InstancedStaticMeshComponent.h`) | HISM (`HierarchicalInstancedStaticMeshComponent.h`) |
|---|---|---|
| Best for | < 1,000 dynamic instances | > 1,000 mostly static |
| Culling | Distance only | Hierarchical BVH + distance |
| LOD | GPU selection | Built-in transitions |
| Remove cost | O(n) | async BVH rebuild |

### Key ISM API (from `InstancedStaticMeshComponent.h`)

```cpp
virtual int32 AddInstance(const FTransform& T, bool bWorldSpace = false);
virtual TArray<int32> AddInstances(const TArray<FTransform>& Ts,
    bool bShouldReturnIndices, bool bWorldSpace = false, bool bUpdateNavigation = true);
virtual bool UpdateInstanceTransform(int32 Idx, const FTransform& NewT,
    bool bWorldSpace = false, bool bMarkRenderStateDirty = false, bool bTeleport = false);
virtual bool BatchUpdateInstancesTransforms(int32 StartIdx, const TArray<FTransform>& NewTs,
    bool bWorldSpace = false, bool bMarkRenderStateDirty = false, bool bTeleport = false);
bool GetInstanceTransform(int32 Idx, FTransform& OutT, bool bWorldSpace = false) const;
virtual bool RemoveInstance(int32 InstanceIndex);  // O(n) for ISM; triggers async BVH rebuild for HISM
virtual void PreAllocateInstancesMemory(int32 AddedCount);
int32 GetNumInstances() const;

// Per-instance custom float data (read in materials via PerInstanceCustomData)
virtual void SetNumCustomDataFloats(int32 N);
virtual bool SetCustomDataValue(int32 Idx, int32 DataIdx, float Value,
    bool bMarkRenderStateDirty = false);
virtual bool SetCustomData(int32 Idx, TArrayView<const float> Floats,
    bool bMarkRenderStateDirty = false);
```

Culling properties: `InstanceStartCullDistance`, `InstanceEndCullDistance`, `InstanceLODDistanceScale`, `bUseGpuLodSelection`.

### Vegetation Scatter (HISM + Terrain Trace)

```cpp
HISM->SetStaticMesh(TreeMesh);
HISM->SetNumCustomDataFloats(1);
HISM->PreAllocateInstancesMemory(Count);
FRandomStream Rand(Seed);
TArray<FTransform> Transforms; Transforms.Reserve(Count);

for (int32 i = 0; i < Count; i++)
{
    FVector Loc(Rand.FRandRange(Min.X, Max.X), Rand.FRandRange(Min.Y, Max.Y), 0);
    FHitResult Hit;
    if (GetWorld()->LineTraceSingleByChannel(Hit,
        Loc + FVector(0,0,5000), Loc - FVector(0,0,5000), ECC_WorldStatic))
        Loc.Z = Hit.Location.Z;

    Transforms.Add(FTransform(
        FRotator(0, Rand.FRandRange(0,360), 0), Loc,
        FVector(Rand.FRandRange(0.8f, 1.3f))));
}

TArray<int32> Indices = HISM->AddInstances(Transforms, true, true);
for (int32 i = 0; i < Indices.Num(); i++)
    HISM->SetCustomDataValue(Indices[i], 0, Rand.FRand(), false);
HISM->MarkRenderStateDirty();
```

### Foliage System

The editor's Foliage paint mode uses `AInstancedFoliageActor` which internally wraps `UHierarchicalInstancedStaticMeshComponent`. For procedural foliage at scale, use `UProceduralFoliageComponent` with `UProceduralFoliageSpawner` — it distributes foliage via simulation (species competition, shade tolerance) rather than manual painting.

**Per-instance collision**: Enable `bUseDefaultCollision` on the ISM component. Each instance inherits the static mesh's collision. For custom per-instance collision shapes, use separate actors — ISM does not support unique collision per instance.

**Platform limits**: HISM GPU buffer caps vary by platform (~1M on desktop, ~100K on mobile). Monitor with `stat Foliage`. Split large populations across multiple HISM components.

---

## 4. Noise and Math

```cpp
// Built-in Perlin (all output in [-1, 1])
float N1 = FMath::PerlinNoise1D(X * Freq);
float N2 = FMath::PerlinNoise2D(FVector2D(X, Y) * Freq);
float N3 = FMath::PerlinNoise3D(FVector(X, Y, Z) * Freq);

// Octave noise
float OctaveNoise(float X, float Y, int32 Oct, float Persist, float Lacu, float Scale)
{
    float V=0, A=1, F=1.f/Scale, Max=0;
    for (int32 i=0; i<Oct; i++) {
        V += FMath::PerlinNoise2D(FVector2D(X,Y)*F) * A;
        Max += A; A *= Persist; F *= Lacu;
    }
    return V / Max;
}

// Seeded deterministic random
FRandomStream Stream(Seed);
float R = Stream.FRandRange(Min, Max);
int32 I = Stream.RandRange(MinI, MaxI);
FVector Dir = Stream.VRand();
```

**Height/density maps**: Sample `UTexture2D` pixel data via `FTexturePlatformData` to drive terrain height or placement density. Lock with `BulkData.Lock(LOCK_READ_ONLY)`, read, then unlock.

**Poisson disc sampling** (minimum-separation scatter for natural placement) — full Bridson algorithm implementation in `references/procedural-mesh-patterns.md`.

---

## 5. Spline Components

### USplineComponent API (from `SplineComponent.h`)

```cpp
// Build spline (always batch with bUpdateSpline=false, call UpdateSpline() once after)
void AddSplinePoint(const FVector& Pos, ESplineCoordinateSpace::Type Space, bool bUpdate=true);
void SetSplinePoints(const TArray<FVector>& Pts, ESplineCoordinateSpace::Type Space, bool bUpdate=true);
void ClearSplinePoints(bool bUpdate=true);
virtual void UpdateSpline();  // Rebuild reparameterization table

// Query by arc-length distance
FVector    GetLocationAtDistanceAlongSpline(float Dist, ESplineCoordinateSpace::Type Space) const;
FVector    GetDirectionAtDistanceAlongSpline(float Dist, ESplineCoordinateSpace::Type Space) const;
FVector    GetRightVectorAtDistanceAlongSpline(float Dist, ESplineCoordinateSpace::Type Space) const;
FRotator   GetRotationAtDistanceAlongSpline(float Dist, ESplineCoordinateSpace::Type Space) const;
FTransform GetTransformAtDistanceAlongSpline(float Dist, ESplineCoordinateSpace::Type Space, bool bUseScale=false) const;
float      GetSplineLength() const;

// Point editing
int32  GetNumberOfSplinePoints() const;
void   SetSplinePointType(int32 Idx, ESplinePointType::Type Type, bool bUpdate=true);
void   SetClosedLoop(bool bClosed, bool bUpdate=true);
void   SetTangentsAtSplinePoint(int32 Idx, const FVector& Arrive, const FVector& Leave,
           ESplineCoordinateSpace::Type Space, bool bUpdate=true);
```

Point types: `Linear`, `Curve`, `Constant`, `CurveClamped`, `CurveCustomTangent`.

`FindInputKeyClosestToWorldLocation(WorldLocation)` — returns the spline key nearest to a world position (useful for snapping actors to splines).

**Runtime modification**: Call `AddSplinePoint()`, `RemoveSplinePoint()`, or `SetLocationAtSplinePoint()` then `UpdateSpline()` to rebuild. Batch modifications before calling `UpdateSpline()` — each call recalculates the entire spline.

### Spline Placement Example

```cpp
// Place instances evenly along spline
float Len = Spline->GetSplineLength();
for (float D = 0.f; D <= Len; D += Spacing)
{
    FTransform T = Spline->GetTransformAtDistanceAlongSpline(D, ESplineCoordinateSpace::World);
    HISM->AddInstance(T, /*bWorldSpace=*/true);
}
```

### USplineMeshComponent (Mesh Deformation)

```cpp
#include "Components/SplineMeshComponent.h"
USplineMeshComponent* SM = NewObject<USplineMeshComponent>(this);
SM->SetStaticMesh(PipeMesh);
SM->RegisterComponent();

FVector SP, ST, EP, ET;
Spline->GetLocationAndTangentAtSplinePoint(Seg,   SP, ST, ESplineCoordinateSpace::Local);
Spline->GetLocationAndTangentAtSplinePoint(Seg+1, EP, ET, ESplineCoordinateSpace::Local);
SM->SetStartAndEnd(SP, ST, EP, ET, /*bUpdateMesh=*/true);
SM->SetForwardAxis(ESplineMeshAxis::X);
```

---

## 6. Runtime Mesh Generation Patterns

See `references/procedural-mesh-patterns.md` for full implementations:
- **Marching Cubes** — isosurface from 3D density scalar field
- **Dungeon BSP** — BSP partition into rooms, L-corridor carving, tile-to-mesh
- **L-System** — string rewriting + turtle interpreter to HISM branches
- **Wave Function Collapse** — constraint-propagation tile grid layout
- **Async mesh generation** — background thread vertex computation, game thread `CreateMeshSection`
- **Spline road extrusion** — cross-section profile swept along `USplineComponent`

```cpp
// Marching Cubes result → ProceduralMesh
ProceduralMesh->CreateMeshSection(0, MarchVerts, MarchTris, MarchNormals,
    MarchUVs, {}, {}, /*bCreateCollision=*/true);
```

---

## 7. PCG Building Generation (六段管线: Input → Filter → Transform → Grammar → Output → Validate)

Workflow for modular buildings, blockouts, facade rules, lot-based building sets, and runtime scheduled generation. Converts designer constraints into reusable PCG graphs. Define upfront: generation target (blockout towers, modular facades, or lot-based sets), deterministic inputs (lot splines/points, district tags, style preset, seed), runtime mode (static bake, on-demand, or runtime scheduled), and output mode (Static Mesh instances first; Spawn Actor only for interactive/stateful parts).

### Graph Stage Contract

Every pipeline stage must explicitly declare:

- Input data type (`EPCGDataType` or asset source)
- Core node/classes (minimum two)
- Required parameters (seed, tags, ranges, radii, or style keys)
- Output data type
- Debug method (node-level checks, debug node, or log/assert path)

If a stage cannot satisfy these five items, the graph design is incomplete.

### Pipeline Stages

#### 1) Input
- Input data type: actor/spline/point sources.
- Core node/classes: `UPCGDataFromActorSettings`, `UPCGGetActorPropertySettings`, `UPCGCreatePointsSettings`.
- Required parameters: lot tag filters, district/style tags, seed source, source bounds.
- Output: normalized lot point or spline data with stable ordering (stable ordering is required for deterministic replay).
- Debug: run `UPCGDebugSettings` after the input stage and verify point count and bounds.

#### 2) Filter
- Input data type: point/spline data from the Input stage.
- Core node/classes: `UPCGAttributeFilteringSettings`, `UPCGDensityFilterSettings`, `UPCGFilterByTagSettings`.
- Required parameters: slope range, exclusion tags, min lot area/width, occupancy constraints.
- Output: only buildable lots/candidates.
- Debug: compare candidate count before/after filter and inspect rejected tag distribution.

#### 3) Transform
- Input data type: filtered buildable candidates.
- Core node/classes: `UPCGCopyPointsSettings`, `UPCGCreateSplineSettings`, `UPCGApplyScaleToBoundsSettings`.
- Required parameters: floor height, pivot convention, facade orientation basis, local axes.
- Output: footprint transforms and per-floor transforms.
- Debug: inspect transform axes and floor index attributes on output points.

#### 4) Grammar
- Input data type: segment/spline/point data from the Transform stage.
- Core node/classes: `UPCGSubdivideSplineSettings`, `UPCGSubdivideSegmentSettings`, `UPCGSelectGrammarSettings`.
- Required parameters: `GrammarSelection`, module size limits, style-based grammar key mapping.
- Output: grammar-resolved module placements/attributes.
- Debug: `UPCGPrintGrammarSettings` for grammar parse and token validation.

#### 5) Output
- Input data type: grammar-resolved placements.
- Core node/classes: `UPCGStaticMeshSpawnerSettings`, `UPCGSpawnActorSettings`, `UPCGCreateTargetActor`.
- Required parameters:
  - Static path: mesh selector, instance packer, ISM/HISM policy.
  - Actor path: actor class, spawn attributes, state/interaction requirements.
- Output: rendered buildings and optional interactive building elements.
- Debug: split output by layer/tag and validate per-layer counts.
- Default policy: prefer Static Mesh Spawner for high-count rendering; use Spawn Actor only when stateful behavior is required.

#### 6) Validate
- Input data type: final spawned result and runtime generation state.
- Core node/classes: `UPCGDebugSettings`, `UPCGComponent`, `UPCGSubsystem`.
- Required parameters: expected cell bounds, max per-update spawn budget, nav/collision expectations.
- Output: pass/fail signals and fix actions.
- Debug: run staged checks for overlap, navigation impact, per-cell generation time, and deterministic replay.

### Shape Grammar

- Grammar selection is driven by `UPCGSubdivisionBaseSettings::GrammarSelection`.
- Do NOT rely on deprecated grammar fields (`bGrammarAsAttribute_DEPRECATED`, `Grammar_DEPRECATED`).
- When grammar parse fails or produces empty modules, validate the grammar string, module dictionary, and segment size constraints.

### Runtime Generation Settings

Runtime generation must explicitly set:

- `EPCGComponentGenerationTrigger::GenerateAtRuntime` as the component's `GenerationTrigger`
- explicit `GenerationRadii` via `bOverrideGenerationRadii`/`GenerationRadii` — do not rely on implicit defaults
- explicit `SchedulingPolicyClass`/`SchedulingPolicy` for predictable scheduler behavior

Keep runtime generation bounds explicit to avoid uncontrolled world-wide regeneration; treat World Partition boundaries as hard constraints for runtime scopes. Avoid hidden dependency on editor-only data when runtime generation is expected.

### Runtime Scheduler Refresh Ops (`UPCGSubsystem`)

- Component refresh (`RefreshRuntimeGenComponent`) when one runtime component changed style/radii/scheduling inputs.
- Global refresh (`RefreshAllRuntimeGenComponents`) when style/global rules changed for many runtime components.
- Immediate local cleanup (`CleanupLocalComponentsImmediate`) when bounds shrink or partition ownership changed.
- After cleanup, trigger local regeneration only for the affected runtime scope.

### Building Failure Handling

- No buildings spawn → verify Input stage output count, source bounds, lot tags; confirm a non-empty candidate set.
- Editor preview works but runtime is empty → set `GenerateAtRuntime`, radii override, and a valid scheduling policy.
- Runtime update regenerates too wide an area → reduce generation/cleanup radii and tighten source bounds.
- Stale pieces remain after rules shrink → trigger cleanup with remove-components behavior and force local cleanup when needed.
- Heavy hitching during runtime generation → reduce per-cell complexity, cap actor spawns, move non-interactive parts to static mesh instances.
- Deterministic replay mismatch with same seed → normalize ordering before random selection and bind every stochastic path to explicit seed inputs.
- Overlap/collision issues → add clearance/slope filters and occupancy rejection before the output stage.
- Navmesh degradation → split nav-affecting vs non-nav-affecting outputs and rebuild nav only where required.
- Runtime changes not applied after parameter edits → request runtime scheduler refresh for the modified component or all runtime components.

### UE5.6 / UE5.7 Compatibility

- Core runtime trigger and grammar APIs are stable in both UE5.6 and UE5.7.
- Header path difference for the PCG subsystem:
  - UE5.6 commonly uses `Public/PCGSubsystem.h`
  - UE5.7 commonly uses `Public/Subsystems/PCGSubsystem.h`

---

## Common Mistakes and Anti-Patterns

**PCG**
- Calling `GenerateLocal()` in `Tick` — generation is not free; use `GenerateOnDemand` and regenerate only on data change.
- Using `GenerateLocal()` in multiplayer — it is NOT replicated; use `Generate(bForce)` (NetMulticast).
- Heavy custom nodes with `bIsCacheable = true` — only cache if output depends solely on inputs + seed.
- Graphs with `bIsEditorOnly = true` fail to cook into packaged builds.

**ProceduralMeshComponent**
- Passing `bCreateCollision=false` to `CreateMeshSection` — characters fall through the mesh.
- Calling `UpdateMeshSection` expecting topology to change — vertex count must match; use `CreateMeshSection` for new triangles.
- Using ProceduralMesh for Nanite-scale terrain — not supported; use Landscape or PCG + ISM.
- Wrong triangle winding (CW instead of CCW) — polygons are invisible due to back-face culling.

**ISM / HISM**
- Using ISM above ~500 instances — switch to HISM for BVH culling.
- Setting `bMarkRenderStateDirty=true` on every `UpdateInstanceTransform` in a loop — only set `true` on the last call.
- Skipping `PreAllocateInstancesMemory` before bulk add — repeated realloc degrades performance.

**Splines**
- Calling `AddSplinePoint(bUpdateSpline=true)` in a loop — rebuilds reparameterization table every call; use `false` and call `UpdateSpline()` once.
- Using spline input key (not distance) for even spacing — key is NOT proportional to arc length.

**Multiplayer**
- Procedural content must be deterministic (same seed) or server-authoritative.
- `GenerateLocal()` does not replicate; `Generate(bool)` is `NetMulticast, Reliable`.

---

## Related Skills

- `unreal-actor-component-architecture` — component lifecycle, registration, replication
- `unreal-physics-collision` — collision profiles, complex vs. simple on generated geometry
- `unreal-cpp-foundations` — `NewObject`, `TSubclassOf`, `TArray`, memory management

## Reference Files

- `references/pcg-node-reference.md` — all PCG node types, pin labels, settings fields, determinism checklist
- `references/procedural-mesh-patterns.md` — quad grid, marching cubes, dungeon BSP, L-system, WFC, spline road





