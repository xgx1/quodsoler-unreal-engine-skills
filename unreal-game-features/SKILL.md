---
name: unreal-game-features
description: "GameFeatureAction/Data, GameFrameworkComponentManager, init state, experience, modular components, UPawn/UController/UGameState/UPlayerStateComponent, Lyra."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - game
  - features
---

# Unreal Game Features

## Tool Constraints

You have access to exactly these tools: `Read`, `Write`, `Edit`. Do not use `bash`, `cat`, `ls`, `grep`, or any other shell commands. If a tool fails, do not retry with a different tool — proceed with the fallback behavior described in the relevant section.

## Trigger Words

Use this skill when the user mentions:
- "game"
- "features"
- "game features"

## Context Check

Use the `Read` tool to read `.agents/unreal-project-context.md` before proceeding. Do not use `bash`, `cat`, or other shell commands. Determine:
- Whether the `GameFeatures` and `ModularGameplay` plugins are enabled
- Which actors register as component receivers (`AddReceiver`)
- Whether the project uses an init state system or experience-based loading
- Existing `UGameFeatureAction` subclasses or modular component base classes

**If the context file is missing or unreadable**: Proceed with reasonable defaults. Assume `GameFeatures` and `ModularGameplay` plugins are enabled (they are engine plugins). Ask the developer to confirm the project setup, then implement based on standard Lyra-style patterns. Do not block implementation on missing context.

## Information Gathering

Ask the developer:
1. Are you creating a new Game Feature plugin or extending an existing one?
2. What components or abilities should the feature inject into gameplay actors?
3. Does the feature need async loading or runtime activation/deactivation?
4. Is there an experience/game mode composition system (Lyra-style)?
5. Do components need ordered initialization across features?

> **Note**: If the context file cannot be read (e.g., `Read` tool returns an error or the file doesn't exist), skip asking about project-specific setup and proceed directly to implementing the requested feature with standard patterns. The developer can adjust after seeing the implementation. If you attempted to use a tool and it failed, do not retry with a different tool — just proceed with defaults.

---

## Game Feature Plugin Structure

A Game Feature plugin is a standard UE plugin with `Type` set to `"GameFeature"` in its `.uplugin` descriptor. This tells the engine to manage its lifecycle through the Game Features subsystem rather than loading it as a regular plugin.

### .uplugin Descriptor

```cpp
{
    "Type": "GameFeature",
    "BuiltInInitialFeatureState": "Active",  // or "Registered", "Installed"
    "Plugins": [
        { "Name": "GameFeatures", "Enabled": true },
        { "Name": "ModularGameplay", "Enabled": true }
    ]
}
```

`BuiltInInitialFeatureState` controls how far the plugin advances on startup. Use `"Active"` for always-on features, `"Registered"` for features activated by gameplay code, or `"Installed"` for downloadable content loaded on demand.

### UGameFeatureData

Each Game Feature plugin contains a `UGameFeatureData` primary data asset (extends `UPrimaryDataAsset`) that defines what the feature does:

```cpp
// From GameFeatureData.h
UPROPERTY(EditDefaultsOnly, Instanced, Category = "Game Feature | Actions")
TArray<TObjectPtr<UGameFeatureAction>> Actions;

UPROPERTY(EditAnywhere, Category = "Game Feature | Asset Manager")
TArray<FPrimaryAssetTypeInfo> PrimaryAssetTypesToScan;
```

`Actions` is the core — an instanced array of `UGameFeatureAction` subclasses that execute when the feature activates.

### Directory Convention

```
Plugins/GameFeatures/
├── ShooterCore/
│   ├── ShooterCore.uplugin          (Type: GameFeature)
│   ├── Content/
│   │   └── ShooterCore.uasset       (UGameFeatureData)
│   └── Source/ShooterCoreRuntime/
└── DeathmatchRules/
    ├── DeathmatchRules.uplugin
    └── Content/DeathmatchRules.uasset
```

---

## Plugin State Machine

Game Feature plugins transition through a well-defined state machine. Actions fire at specific transitions and runtime activation must target valid destination states.

### EGameFeaturePluginState Lifecycle

```
Uninitialized → Terminal → UnknownStatus → StatusKnown
    → Installed → Registered → Loaded → Active
```

Each major state has transition states between them (e.g., `Registering`, `Loading`, `Activating`). You target a destination state and the subsystem walks the chain.

### Destination States

| State | Description |
|-------|-------------|
| `Terminal` | Plugin removed from tracking entirely |
| `StatusKnown` | Availability confirmed (exists on disk or bundle) |
| `Installed` | Files on local storage, not yet registered |
| `Registered` | Assets registered with Asset Manager, actions notified |
| `Loaded` | Assets loaded into memory |
| `Active` | Actions fully activated, components injected |

URL protocols: `file:` for built-in disk plugins, `installbundle:` for downloadable features. Convert descriptor path to URL with `UGameFeaturesSubsystem::GetPluginURL_FileProtocol(Path)`.

---

## UGameFeatureAction

`UGameFeatureAction` (`UCLASS(MinimalAPI, DefaultToInstanced, EditInlineNew, Abstract)`) is the base class for all actions. `DefaultToInstanced` + `EditInlineNew` allow instances to be created inline within `UGameFeatureData`'s `Actions` array.

### Lifecycle Methods

```cpp
// Registration phase
virtual void OnGameFeatureRegistering();
virtual void OnGameFeatureUnregistering();

// Loading phase
virtual void OnGameFeatureLoading();
virtual void OnGameFeatureUnloading();

// Activation — primary override point
virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context);
virtual void OnGameFeatureActivating();  // legacy no-arg fallback

// Post-activation confirmation
virtual void OnGameFeatureActivated();

// Deactivation — supports async via context
virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context);
```

`OnGameFeatureActivating(Context)` is the primary override. The base calls the legacy no-arg version for backward compatibility.

### Async Deactivation

When deactivation requires async work, pause it via the context:

```cpp
void UMyAction::OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context)
{
    Context.PauseDeactivation();
    // Start async work, call Context.UnpauseDeactivation() when done
}
```

---

## UGameFrameworkComponentManager

`UGameFrameworkComponentManager` is the central registry for modular component injection. It lives on the game instance and manages component requests per actor class.

### Key Methods

```cpp
// Register an actor class as a component receiver
void AddReceiver(AActor* Receiver, FGameFeatureStateChangeContext const& Context);

// Remove a receiver
void RemoveReceiver(AActor* Receiver, FGameFeatureStateChangeContext const& Context);

// Add a component request for a specific actor class
FDelegateHandle AddComponentRequest(
    TSubclassOf<AActor> ReceiverClass,
    const FComponentRequestHandle& Request
);

// Remove a component request
void RemoveComponentRequest(
    TSubclassOf<AActor> ReceiverClass,
    FDelegateHandle Handle
);
```

### Component Request Structure

```cpp
struct FComponentRequestHandle
{
    TSubclassOf<UActorComponent> ComponentClass;
    EComponentCreationMethod CreationMethod;  // EComponentCreationMethod::Instance
    ENetRole RoleFilter;                      // ROLE_Authority typically
    bool bAddInactive;                        // false for active components
};
```

### Typical Usage Pattern

```cpp
void UMyFeatureAction::AddComponentForActor(
    TSubclassOf<AActor> ActorClass,
    TSubclassOf<UActorComponent> ComponentClass,
    FGameFeatureStateChangeContext const& Context)
{
    UGameFrameworkComponentManager* Manager = 
        UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GetWorld()->GetGameInstance());
    
    if (Manager)
    {
        FComponentRequestHandle Request;
        Request.ComponentClass = ComponentClass;
        Request.CreationMethod = EComponentCreationMethod::Instance;
        Request.RoleFilter = ROLE_Authority;
        
        FDelegateHandle Handle = Manager->AddComponentRequest(ActorClass, Request);
        Handles.Add(Handle);
    }
}
```

---

## Modular Components

Modular components are `UActorComponent` subclasses designed for injection via `UGameFrameworkComponentManager`. They follow a pattern where the component registers itself with the owning actor's feature state.

### Base Class Pattern

```cpp
UCLASS(Abstract)
class UModularComponentBase : public UActorComponent
{
    GENERATED_BODY()
    
public:
    virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params);
    virtual bool CanChangeInitState(UGameFrameworkComponentManager* Manager, 
                                     FGameplayTag CurrentState, 
                                     FGameplayTag DesiredState) const;
    virtual void HandleChangeInitState(UGameFrameworkComponentManager* Manager,
                                        FGameplayTag CurrentState,
                                        FGameplayTag DesiredState);
};
```

### Lyra-Specific Component Types

| Component Type | Base Class | Typical Receiver |
|---------------|------------|------------------|
| Pawn Component | `UPawnComponent` | `APawn` |
| Controller Component | `UControllerComponent` | `AController` |
| Game State Component | `UGameStateComponent` | `AGameStateBase` |
| Player State Component | `UPlayerStateComponent` | `APlayerState` |
| Ability System Component | `UAbilitySystemComponent` | `APawn` or `APlayerState` |

---

## Init State System

The init state system provides ordered initialization across modular components. Components declare dependencies and transition through states in a controlled sequence.

### Common Init States

```cpp
// From LyraGameplayTags.h
FGameplayTag InitState_Spawned;      // Component created
FGameplayTag InitState_DataAvailable; // Data dependencies resolved
FGameplayTag InitState_DataInitialized; // Data loaded and validated
FGameplayTag InitState_GameplayReady;  // Fully operational
```

### State Transition Flow

```
Spawned → DataAvailable → DataInitialized → GameplayReady
```

Components implement `CanChangeInitState` to declare prerequisites:

```cpp
bool UMyComponent::CanChangeInitState(UGameFrameworkComponentManager* Manager,
                                       FGameplayTag CurrentState,
                                       FGameplayTag DesiredState) const
{
    if (CurrentState == InitState_Spawned && DesiredState == InitState_DataAvailable)
    {
        return GetOwner()->HasActorBegunPlay();
    }
    if (CurrentState == InitState_DataAvailable && DesiredState == InitState_DataInitialized)
    {
        return MyDataAsset != nullptr;
    }
    if (CurrentState == InitState_DataInitialized && DesiredState == InitState_GameplayReady)
    {
        return true; // No further dependencies
    }
    return false;
}
```

---

## Experience System

The experience system (Lyra-style) uses a primary data asset (`UGameExperienceData`) that defines the game mode composition. It loads Game Feature plugins and initializes modular components.

### Experience Data Asset

```cpp
UCLASS()
class UGameExperienceData : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditDefaultsOnly, Category = "Game Feature Plugins")
    TArray<FString> GameFeaturesToEnable;
    
    UPROPERTY(EditDefaultsOnly, Category = "Actions")
    TArray<TObjectPtr<UGameFeatureAction>> Actions;
    
    UPROPERTY(EditDefaultsOnly, Category = "Pawn")
    TSubclassOf<APawn> PawnClass;
    
    UPROPERTY(EditDefaultsOnly, Category = "HUD")
    TSubclassOf<AHUD> HUDClass;
    
    UPROPERTY(EditDefaultsOnly, Category = "Game State")
    TSubclassOf<AGameStateBase> GameStateClass;
    
    UPROPERTY(EditDefaultsOnly, Category = "Player State")
    TSubclassOf<APlayerState> PlayerStateClass;
};
```

### Experience Loading Flow

```
1. Experience requested → Load experience data asset
2. Enable Game Feature plugins listed in GameFeaturesToEnable
3. Activate experience's own UGameFeatureActions
4. Set game mode classes (Pawn, HUD, GameState, PlayerState)
5. Initialize modular components via init state system
```

---

## Common Patterns

### Pattern 1: Simple Component Injection

```cpp
UCLASS()
class UMyFeatureAction_AddComponents : public UGameFeatureAction
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, Category = "Components")
    TMap<TSubclassOf<AActor>, TSubclassOf<UActorComponent>> ComponentMap;
    
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        UGameFrameworkComponentManager* Manager = GetManager();
        if (!Manager) return;
        
        for (auto& Pair : ComponentMap)
        {
            FComponentRequestHandle Request;
            Request.ComponentClass = Pair.Value;
            Request.CreationMethod = EComponentCreationMethod::Instance;
            Request.RoleFilter = ROLE_Authority;
            
            FDelegateHandle Handle = Manager->AddComponentRequest(Pair.Key, Request);
            Handles.Add(Handle);
        }
    }
    
    virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override
    {
        UGameFrameworkComponentManager* Manager = GetManager();
        if (!Manager) return;
        
        for (auto& Handle : Handles)
        {
            Manager->RemoveComponentRequest(nullptr, Handle);
        }
        Handles.Empty();
    }
    
private:
    TArray<FDelegateHandle> Handles;
    
    UGameFrameworkComponentManager* GetManager() const
    {
        return UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(
            GetWorld()->GetGameInstance());
    }
};
```

### Pattern 2: Experience-Based Feature

```cpp
UCLASS()
class UMyExperienceAction : public UGameFeatureAction
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, Category = "Experience")
    TSoftObjectPtr<UGameExperienceData> ExperienceToLoad;
    
    virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override
    {
        if (!ExperienceToLoad.IsValid())
        {
            ExperienceToLoad.LoadSynchronous();
        }
        
        // Register experience with experience manager
        UGameInstance* GameInstance = GetWorld()->GetGameInstance();
        UExperienceManager* ExpManager = GameInstance->GetSubsystem<UExperienceManager>();
        if (ExpManager)
        {
            ExpManager->RegisterExperience(ExperienceToLoad.Get());
        }
    }
};
```

---

## Error Handling & Debugging

### Common Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Plugin not activating | Feature state stuck at `Registered` | Check `BuiltInInitialFeatureState` in `.uplugin` |
| Components not injecting | Actor missing expected components | Verify `AddReceiver` was called on the actor |
| Init state deadlock | Components stuck in `Spawned` | Check `CanChangeInitState` dependencies |
| Missing dependencies | Plugin fails to load | Verify plugin dependencies in `.uplugin` |

### Debugging Commands

```
// List all game feature plugins and their states
GameFeaturePlugin.List

// Activate a specific plugin
GameFeaturePlugin.Activate <PluginURL>

// Deactivate a specific plugin
GameFeaturePlugin.Deactivate <PluginURL>

// Dump component manager state
GameFrameworkComponentManager.Dump
```

---

## References

See `references/` directory for:
- `UGameFeatureAction` template implementations
- `UGameFrameworkComponentManager` usage examples
- Experience system patterns from Lyra
- Init state system integration examples