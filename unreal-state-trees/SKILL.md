---
name: unreal-state-trees
description: "State Tree, UStateTree, StateTreeTask/Condition/Evaluator, StateTreeSchema, AI/Mass State Tree, FStateTreeExecutionContext, data-driven state logic."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - state
  - trees
---

# Unreal State Trees

## Output Rules
- Always output complete, compilable C++ code in the response body (not as tool calls).
- Do not call Read/Write tools unless the user explicitly asks to save to a file.
- If the user asks for a task/condition/evaluator, output the full header and implementation code inline.

## Trigger Words

Use this skill when the user mentions:
- "state"
- "trees"
- "state trees"

## Context Check

Read `.agents/unreal-project-context.md` if it exists. If the file is missing or the read fails, assume defaults:
- `StateTreeModule` and `GameplayStateTreeModule` plugins are enabled
- No Mass Entity integration needed
- Using `UStateTreeComponentSchema` with `AAIController`
- No existing AI frameworks to migrate from

If `.agents/unreal-project-context.md` does not exist or cannot be read, proceed with implementation using the standard `UStateTreeComponentSchema` and `UStateTreeComponent`. Do not block on missing context.

## Information Gathering

Before implementing, clarify:
1. What is the use case? (AI behavior, game logic, UI state, entity processing)
2. What scale? (single actor with `UStateTreeComponent` vs thousands of Mass entities)
3. How complex? (simple linear FSM vs hierarchical states with linked subtrees)
4. Are there existing behavior trees to migrate from?
5. What external data do tasks need? (actor references, subsystems, world state)

## Implementation Templates

### UStateTreeTask (BlueprintType)
```cpp
// MyStateTreeTask_Wait.h
#pragma once

#include "StateTreeTaskBase.h"
#include "MyStateTreeTask_Wait.generated.h"

USTRUCT()
struct FMyStateTreeTask_WaitInstanceData
{
    GENERATED_BODY()
    
    /** Duration to wait in seconds */
    UPROPERTY(EditAnywhere, Category = Parameter)
    float Duration = 1.0f;
    
    /** Elapsed time since entering state */
    float ElapsedTime = 0.0f;
};

USTRUCT(meta = (DisplayName = "Wait"))
struct FMyStateTreeTask_Wait : public FStateTreeTaskBase
{
    GENERATED_BODY()

    using FInstanceDataType = FMyStateTreeTask_WaitInstanceData;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override
    {
        FInstanceDataType& InstanceData = Context.GetInstanceData(*this);
        InstanceData.ElapsedTime = 0.0f;
        return EStateTreeRunStatus::Running;
    }

    virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const override
    {
        FInstanceDataType& InstanceData = Context.GetInstanceData(*this);
        InstanceData.ElapsedTime += DeltaTime;
        if (InstanceData.ElapsedTime >= InstanceData.Duration)
        {
            return EStateTreeRunStatus::Succeeded;
        }
        return EStateTreeRunStatus::Running;
    }

    virtual void ExitState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override
    {
        // Cleanup if needed
    }
};
```

---

## StateTree Architecture

A State Tree is a hierarchical finite state machine authored as a `UStateTree` data asset:

```
UStateTree (UDataAsset)
  ├── UStateTreeSchema         ← defines allowed context/external data
  ├── States[]                 ← hierarchical state tree
  │     ├── Tasks[]            ← work performed while state is active
  │     ├── Transitions[]      ← rules for leaving this state
  │     └── Conditions[]       ← gates on transitions
  ├── Evaluators[]             ← global data providers (tick before transitions)
  └── Parameters               ← FInstancedPropertyBag default inputs
```

**Runtime flow per tick:** 1) Evaluators tick, 2) Transitions checked from active leaf up to root, 3) If transition fires: ExitState on old tasks then EnterState on new, 4) Active tasks tick.

**Key classes:**

| Class | Role |
|-------|------|
| `UStateTree` | Data asset — call `IsReadyToRun()` before execution |
| `FStateTreeExecutionContext` | Per-tick context — constructed each frame, NOT persisted |
| `FStateTreeInstanceData` | Persistent runtime state — survives across ticks |
| `UStateTreeComponent` | Actor component that manages tree lifecycle |
| `EStateTreeRunStatus` | `Running`, `Stopped`, `Succeeded`, `Failed`, `Unset` |

**Build.cs modules**: `StateTreeModule`, `GameplayStateTreeModule`

The execution context is constructed per-tick from persistent instance data:
```cpp
FStateTreeInstanceData InstanceData;  // persists across frames
// Each tick:
FStateTreeExecutionContext Context(Owner, *StateTree, InstanceData);
Context.Tick(DeltaTime);
```

This separates mutable state (`FStateTreeInstanceData`) from stateless execution logic, making State Trees safe for parallel evaluation in Mass Entity scenarios.

---

## Schema System

Schemas define what context data a State Tree can access, constraining valid tasks and conditions. This prevents authoring errors at edit time rather than runtime.

| Schema | Context Provided | Use Case |
|--------|-----------------|----------|
| `UStateTreeComponentSchema` | Actor + BrainComponent | General actor logic |
| `UStateTreeAIComponentSchema` | Above + `AIControllerClass` | AI behavior |
| `UMassStateTreeSchema` | Mass entity context | Mass Entity processing |

`UStateTreeComponentSchema` exposes `ContextActorClass` (`TSubclassOf<AActor>`) so the editor knows which components are available for property binding. `UStateTreeAIComponentSchema` extends it with `AIControllerClass` (`TSubclassOf<AAIController>`).

### Custom Schemas

Subclass `UStateTreeSchema` for project-specific trees:

```cpp
UCLASS()
class UMyGameSchema : public UStateTreeSchema
{
    GENERATED_BODY()
public:
    virtual bool IsStructAllowed(const UScriptStruct* InStruct) const override;
    virtual bool IsExternalItemAllowed(const UStruct& InStruct) const override;
    virtual TConstArrayView<FStateTreeExternalDataDesc> GetContextDataDescs() const override;

#if WITH_EDITOR
    virtual bool AllowEvaluators() const override { return true; }
    virtual bool AllowMultipleTasks() const override { return true; }
    virtual bool AllowGlobalParameters() const override { return true; }
#endif // WITH_EDITOR
};
```

Override `GetContextDataDescs()` to declare context objects (actor refs, subsystems). The editor uses this to validate property bindings.

---

## Tasks

Tasks are the primary work units in a state. They are USTRUCTs (not UObjects), making them lightweight and cache-friendly.

### FStateTreeTaskBase API

Key virtuals (all `const` — tasks are immutable at runtime):

| Virtual | Returns | Called When |
|---------|---------|-------------|
| `EnterState(Context, Transition)` | `EStateTreeRunStatus` (default: Running) | State becomes active |
| `ExitState(Context, Transition)` | `void` | State is exited |
| `Tick(Context, DeltaTime)` | `EStateTreeRunStatus` (default: Running) | Each frame (if `bShouldCallTick`) |
| `StateCompleted(Context, Status, CompletedStates)` | `void` | Child state completes (REVERSE order) |
| `TriggerTransitions(Context)` | `void` | Only if `bShouldAffectTransitions` |

### Behavioral Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `bShouldStateChangeOnReselect` | `true` | Exit+Enter when transitioning to same state |
| `bShouldCallTick` | `true` | Enable per-frame Tick calls |
| `bShouldCallTickOnlyOnEvents` | `false` | Tick only when events are pending |
| `bShouldCopyBoundPropertiesOnTick` | `true` | Refresh property bindings each tick |
| `bShouldAffectTransitions` | `false` | Enable `TriggerTransitions` calls |