---
name: unreal-ai-navigation
description: "Unreal AI navigation: AIController, behavior tree, blackboard, AI perception, NavMesh, EQS, pathfinding, State Tree, Smart Objects."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - ai
  - navigation
---

# Unreal AI Navigation

## Trigger Words

Use this skill when the user mentions:
- "ai"
- "navigation"
- "ai navigation"

## Required Tools
- `Read` - for reading project context and reference files
- `Write` - for creating new files
- `Edit` - for modifying existing files

Note: Using `Read` for context/reference files is always required before implementation and counts as valid tool usage.

## Tool Usage Rules
1. Before using ANY tool, check it's in the Required Tools list above
2. Do NOT use `bash`, `find`, `grep`, or other shell tools unless the user explicitly confirms they are available
3. If a tool is not in the Required Tools list, ask the user for permission before using it
4. When reading files, always specify the exact file path and use the `Read` tool with the `path` parameter only
5. Always prefer Read/Write/Edit tools over shell commands
6. Tool usage scoring: Required pre-implementation steps (reading context, reference files) are mandatory and count toward tool usage. Do not skip these steps even if the task doesn't explicitly mention them.

## Context

Read `.agents/unreal-project-context.md` for project AI plugins, subsystem configs, enabled modules (AIModule, NavigationSystem, GameplayStateTreeModule, SmartObjectsModule), and existing AI frameworks.

Before implementing any AI behavior, always check for reference files in the `references/` directory using `Read`. Specifically check `references/behavior-tree-patterns.md` and `references/eqs-reference.md` if EQS is involved.

## Information Gathering

Before implementing ANY AI behavior, you MUST first read `.agents/unreal-project-context.md` using the `Read` tool. This is a mandatory prerequisite step. Then check for reference files in `references/` directory. Do not skip these reads even if the user's request seems self-contained.

- Before implementing EQS: Use `Read` with `path: references/eqs-reference.md` to check if the reference file exists. If it does, read it for EQS patterns. If the file doesn't exist, use the EQS implementation patterns in this skill document.

## Project Context Reading
Use the `Read` tool to check `.agents/unreal-project-context.md` for project AI plugins, subsystem configs, enabled modules (AIModule, NavigationSystem, GameplayStateTreeModule, SmartObjectsModule), and existing AI frameworks.

If the file doesn't exist, ask the user for project details instead.

---

## AI Architecture

```
APawn
  └── AAIController (server-only in multiplayer)
        ├── UBehaviorTreeComponent  (UBrainComponent subclass)
        │     └── UBehaviorTree asset → UBlackboardData
        ├── UBlackboardComponent    (AI knowledge store)
        ├── UAIPerceptionComponent  (sight, hearing, damage)
        └── UPathFollowingComponent (NavMesh path execution)
```

**Build.cs modules**: `AIModule`, `NavigationSystem`, `GameplayTasks`

---

## AIController

```cpp
// MyAIController.h
UCLASS()
class AMyAIController : public AAIController
{
    GENERATED_BODY()
public:
    AMyAIController();
    UPROPERTY(EditDefaultsOnly, Category = AI)
    TObjectPtr<UBehaviorTree> BehaviorTreeAsset;
protected:
    virtual void OnPossess(APawn* InPawn) override;
    UFUNCTION()
    void OnTargetPerceptionUpdated(AActor* Actor, FAIStimulus Stimulus);
};

// MyAIController.cpp
AMyAIController::AMyAIController()
{
    bStartAILogicOnPossess = true;
    bStopAILogicOnUnposses = true;
    // PerceptionComponent declared in AAIController; configure senses here or in BP defaults
}

void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);
    if (BehaviorTreeAsset)
        RunBehaviorTree(BehaviorTreeAsset); // calls UseBlackboard internally
    if (UAIPerceptionComponent* PC = GetAIPerceptionComponent())
        PC->OnTargetPerceptionUpdated.AddDynamic(this, &AMyAIController::OnTargetPerceptionUpdated);
}
```

### Key AAIController API

```cpp
// Navigation
EPathFollowingRequestResult::Type MoveToActor(AActor* Goal, float AcceptanceRadius = -1,
    bool bStopOnOverlap = true, bool bUsePathfinding = true, bool bCanStrafe = true,
    TSubclassOf<UNavigationQueryFilter> FilterClass = {}, bool bAllowPartialPath = true);

EPathFollowingRequestResult::Type MoveToLocation(const FVector& Dest, float AcceptanceRadius = -1,
    bool bStopOnOverlap = true, bool bUsePathfinding = true,
    bool bProjectDestinationToNavigation = false, bool bCanStrafe = true,
    TSubclassOf<UNavigationQueryFilter> FilterClass = {}, bool bAllowPartialPath = true);

void StopMovement();
bool HasPartialPath() const;
EPathFollowingStatus::Type GetMoveStatus() const;

// Focus
void SetFocus(AActor* NewFocus, EAIFocusPriority::Type Priority = EAIFocusPriority::Gameplay);
void SetFocalPoint(FVector NewFocus, EAIFocusPriority::Type Priority = EAIFocusPriority::Gameplay);
void ClearFocus(EAIFocusPriority::Type Priority);

// Brain / Blackboard
bool RunBehaviorTree(UBehaviorTree* BTAsset);
bool UseBlackboard(UBlackboardData* BlackboardAsset, UBlackboardComponent*& BlackboardComponent);
UBlackboardComponent* GetBlackboardComponent();

// Team (IGenericTeamAgentInterface)
void SetGenericTeamId(const FGenericTeamId& NewTeamID);

// Delegate: FAIMoveCompletedSignature ReceiveMoveCompleted (RequestID, Result)
```

**On Pawn**: `AIControllerClass = AMyAIController::StaticClass(); AutoPossessAI = EAutoPossessAI::PlacedInWorldOrSpawned;`

---

## EQS Configuration

### EQS Query Asset Setup (FindRandomLocation)

```cpp
// In AIController or AIPerceptionComponent setup
// EQS query for finding random patrol locations
UEnvQuery* PatrolLocationQuery = CreateDefaultSubobject<UEnvQuery>(TEXT("PatrolEQS"));
// Configure in Blueprint or C++:
// - Query Template: EQS_FindRandomLocation
// - Generators: RandomPointsOnNavMesh
//   - NavMeshData: NavigationData
//   - NumberOfPoints: 10
//   - Radius: 1000.0
// - Tests: 
//   - Distance to Current Location (Score: Lower is 