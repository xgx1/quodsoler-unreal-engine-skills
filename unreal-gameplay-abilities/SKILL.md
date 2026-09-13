---
name: unreal-gameplay-abilities
description: "GAS / Gameplay Ability System: GameplayAbility, GameplayEffect, AttributeSet, GameplayTags, ability system, buffs, debuffs, cooldowns, attribute modification."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - gameplay
  - abilities
---

# Unreal Gameplay Abilities

## Trigger Words

Use this skill when the user mentions:
- "gameplay"
- "abilities"
- "gameplay abilities"

## Context Check

Before proceeding, read `.agents/unreal-project-context.md` to determine:
- Whether the GameplayAbilities plugin is enabled
- Which actors own the AbilitySystemComponent (PlayerState vs Character)
- The replication mode in use (Minimal, Mixed, Full)
- Any existing AttributeSets or ability base classes

## Information Gathering

Ask the developer:
1. What type of abilities are needed? (active, passive, triggered, instant)
2. What attributes are required? (health, mana, stamina, custom stats)
3. Is this multiplayer? If so, which actors carry the ASC?
4. Are cooldowns and costs required, or is this a passive/trigger system?
5. Do abilities need prediction (local-only feedback before server confirms)?

---

## GAS Architecture Overview

GAS has three pillars that live on `UAbilitySystemComponent` (ASC):

| Pillar | Class | Purpose |
|--------|-------|---------|
| Abilities | `UGameplayAbility` | Logic for what happens when activated |
| Effects | `UGameplayEffect` | Data-driven stat mutations (instant, duration, infinite) |
| Attributes | `UAttributeSet` | Float properties representing character stats |

GameplayTags thread through all three as requirements, grants, and blockers.

---

## GAS Setup

### 1. Enable the Plugin

Enable `GameplayAbilities` in `.uproject` Plugins array, then in `[ProjectName].Build.cs`:
```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "GameplayAbilities", "GameplayTags", "GameplayTasks"
});
```

### 2. AbilitySystemComponent Ownership

**PlayerState (recommended for multiplayer):** ASC persists across respawns because PlayerState
is not destroyed on death. Use this for player characters in networked games.

**Character/Pawn:** Simpler. Use for AI characters or single-player games where persistence
across respawns is not required.

See `references/gas-setup-patterns.md` for full initialization sequences for both patterns.

### 3. IAbilitySystemInterface

Every actor that owns or exposes an ASC must implement `IAbilitySystemInterface`:

```cpp
#include "AbilitySystemInterface.h"

UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()
public:
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override
        { return AbilitySystemComponent; }
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "GAS")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
};
```

### 4. Replication Modes

Set on the ASC after creation (server-side only):

```cpp
// In BeginPlay or PossessedBy on the server:
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

| Mode | When to Use |
|------|-------------|
| `Minimal` | AI or non-player actors; no GE replication to simulated proxies |
| `Mixed` | Player-controlled characters (owner gets full info, others get minimal) |
| `Full` | Non-player games or debugging; all GEs replicate to all clients |

### 5. InitAbilityActorInfo

Must be called on both server and client after possession. Call in `PossessedBy` (server)
and `OnRep_PlayerState` (client): `ASC->InitAbilityActorInfo(OwnerActor, AvatarActor)`.
See `references/gas-setup-patterns.md` for full dual-path code with respawn handling.

---

## GameplayAbilities

### Subclass UGameplayAbility

```cpp
#include "Abilities/GameplayAbility.h"

UCLASS()
class UMyFireballAbility : public UGameplayAbility
{
    GENERATED_BODY()
public:
    UMyFireballAbility();
    virtual void ActivateAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;
    virtual void EndAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility, bool bWasCancelled) override;
    // Ability Tasks — async building blocks for latent abilities:
    // UAbilityTask_WaitTargetData    — waits for targeting (crosshair/AoE confirm)
    // UAbilityTask_WaitGameplayEvent — waits for a GameplayEvent tag (e.g., anim notify)
    // UAbilityTask_WaitDelay         — simple timer
    // UAbilityTask_PlayMontageAndWait — montage with callbacks (see unreal-animation-system)
    // See references/ability-task-reference.md for full list and custom task pattern.
    // CancelAbility — called by CancelAbilitiesWithTag or ASC->CancelAbility(Handle)
    // Internally calls EndAbility with bWasCancelled=true. Override to add cleanup:
    virtual void CancelAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateCancelAbility) override;
    // Custom activation guard — return false to block activation beyond tag checks
    virtual bool CanActivateAbility(const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo, /*...*/) const override;
    // Must call Super first. Add custom checks (resource availability, cooldown state).
protected:
    UPROPERTY(EditDefaultsOnly, Category = "GAS")
    TSubclassOf<UGameplayEffect> DamageEffect;
};
```

### ActivateAbility Pattern

```cpp
void UMyFireballAbility::ActivateAbility(const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
        return;
    }
    // Apply a GameplayEffect to the target
    // Use Ability Tasks for targeting, montages, etc.
}
```

---

## AttributeSet Implementation

### Subclass UAttributeSet

```cpp
// MyRPGAttributeSet.h
#pragma once
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"
#include "MyRPGAttributeSet.generated.h"

// Macro for attribute accessors
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

UCLASS()
class UMyRPGAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
public:
    UMyRPGAttributeSet();

    // Attributes
    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Attributes")
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Attributes")
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, MaxHealth)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Mana, Category = "Attributes")
    FGameplayAttributeData Mana;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, Mana)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxMana, Category = "Attributes")
    FGameplayAttributeData MaxMana;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, MaxMana)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Stamina, Category = "Attributes")
    FGameplayAttributeData Stamina;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, Stamina)

    UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxStamina, Category = "Attributes")
    FGameplayAttributeData MaxStamina;
    ATTRIBUTE_ACCESSORS(UMyRPGAttributeSet, MaxStamina)

    // Clamping: Called before attribute changes, clamps to valid ranges
    virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;

    // Death trigger: Called after gameplay effects modify attributes
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // Replication
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    // OnRep functions for clients
    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
    UFUNCTION()
    virtual void OnRep_Mana(const FGameplayAttributeData& OldMana);
    UFUNCTION()
    virtual void OnRep_MaxMana(const FGameplayAttributeData& OldMaxMana);
    UFUNCTION()
    virtual void OnRep_Stamina(const FGameplayAttributeData& OldStamina);
    UFUNCTION()
    virtual void OnRep_MaxStamina(const FGameplayAttributeData& OldMaxStamina);
};
```

```cpp
// MyRPGAttributeSet.cpp
#include "MyRPGAttributeSet.h"
#include "GameplayEffectExtension.h"
#include "Net/UnrealNetwork.h"

UMyRPGAttributeSet::UMyRPGAttributeSet()
    : Health(100.0f), MaxHealth(100.0f),
      Mana(50.0f), MaxMana(50.0f),
      Stamina(100.0f), MaxStamina(100.0f)
{
}

void UMyRPGAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    // Clamp to [0, Max] for all attributes
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
    else if (Attribute == GetManaAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxMana());
    }
    else if (Attribute == GetStaminaAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxStamina());
    }
}

void UMyRPGAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        float CurrentHealth = GetHealth();
        // Clamp after execution (catches instant effects)
        SetHealth(FMath::Clamp(CurrentHealth, 0.0f, GetMaxHealth()));

        // Death trigger
        if (CurrentHealth <= 0.0f && !bDead)
        {
            bDead = true;
            // Broadcast death or call death logic
            // Example: GetOwningActor()->Destroy();
            // Or trigger a gameplay event
            if (UAbilitySystemComponent* ASC = GetOwningAbilitySystemComponent())
            {
                FGameplayEventData EventData;
                EventData.Instigator = ASC->GetAvatarActor();
                EventData.Target = ASC->GetOwnerActor();
                ASC->HandleGameplayEvent(FGameplayTag::RequestGameplayTag("Event.Death"), &EventData);
            }
        }
    }
}

void UMyRPGAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, Mana, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, MaxMana, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, Stamina, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyRPGAttributeSet, MaxStamina, COND_None, REPNOTIFY_Always);
}

void UMyRPGAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, Health, OldHealth);
}

void UMyRPGAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, MaxHealth, OldMaxHealth);
}

void UMyRPGAttributeSet::OnRep_Mana(const FGameplayAttributeData& OldMana)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, Mana, OldMana);
}

void UMyRPGAttributeSet::OnRep_MaxMana(const FGameplayAttributeData& OldMaxMana)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, MaxMana, OldMaxMana);
}

void UMyRPGAttributeSet::OnRep_Stamina(const FGameplayAttributeData& OldStamina)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, Stamina, OldStamina);
}

void UMyRPGAttributeSet::OnRep_MaxStamina(const FGameplayAttributeData& OldMaxStamina)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyRPGAttributeSet, MaxStamina, OldMaxStamina);
}
```

**Important Notes**:
- `PreAttributeChange` is for **clamping** only — do not trigger game logic here (can run on clients)
- `PostGameplayEffectExecute` is for **game logic** like death — runs only on the server for instant effects
- Use `REPNOTIFY_Always` to ensure clients get OnRep calls even if the value hasn't changed
- The `ATTRIBUTE_ACCESSORS` macro generates Get/Set/Init functions for each attribute

---

## GameplayEffects

### Creating a GameplayEffect

Create a Blueprint or C++ subclass of `UGameplayEffect`. Key fields:

| Field | Purpose |
|-------|---------|
| Duration Policy | Instant, Has Duration, Infinite |
| Modifiers | Which attributes to modify and how (Add, Multiply, Override) |
| Tags (GameplayEffectAssetTag) | Tags that identify this effect |
| GrantedTags | Tags applied to the target while effect is active |
| Requirements | Tags the target must have (or not have) for the effect to apply |
| Stacking | How multiple instances combine (AggregateBySource, Target, etc.) |

### Applying a GameplayEffect

```cpp
// From within a GameplayAbility:
FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(DamageEffect, GetAbilityLevel());
if (SpecHandle.IsValid())
{
    SpecHandle.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag("Data.Damage"), 50.0f);
    ApplyGameplayEffectSpecToTarget(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, SpecHandle, TargetASC);
}

// From any actor with an ASC:
FGameplayEffectContextHandle ContextHandle = ASC->MakeEffectContext();
ContextHandle.AddHitResult(HitResult);
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingGameplayEffectSpec(DamageEffectClass, 1, ContextHandle);
ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
```

### SetByCaller Magnitudes

For dynamic values (e.g., damage that varies by charge level), use `SetByCaller`:

1. In the GameplayEffect modifier, set Magnitude Calculation Type to `SetByCaller`
2. Define a Data Tag (e.g., `Data.Damage`) on the modifier
3. Set the value at runtime:
```cpp
SpecHandle.Data->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag("Data.Damage"), CalculatedDamage);
```

---

## GameplayTags

### Tag Structure

Use hierarchical tags: `Ability.Action.Fire`, `State.Dead`, `Effect.Damage.Physical`.

### Tag Operations

```cpp
// Create
FGameplayTag Tag = FGameplayTag::RequestGameplayTag("Ability.Action.Fire");

// Check
if (ASC->HasMatchingGameplayTag(Tag)) { }

// Add/Remove
ASC->AddLooseGameplayTag(Tag);
ASC->RemoveLooseGameplayTag(Tag);

// Blocking
Ability->BlockedTags.AddTag(Tag); // Prevents activation while tag is present
```

### Tag Relationships

- **Asset Tags**: Tags on the GE itself (for identification)
- **Granted Tags**: Tags applied to the target while GE is active
- **Required Tags**: Tags target must have for GE to apply
- **Blocked Ability Tags**: Tags that block ability activation

---

## Ability Tasks

### Common Tasks

| Task | Purpose |
|------|---------|
| `UAbilityTask_WaitTargetData` | Waits for player to confirm target (crosshair, AoE) |
| `UAbilityTask_WaitGameplayEvent` | Waits for a GameplayEvent tag (e.g., from anim notify) |
| `UAbilityTask_WaitDelay` | Simple timer |
| `UAbilityTask_PlayMontageAndWait` | Plays a montage with start/end/interrupt callbacks |
| `UAbilityTask_WaitGameplayEffectApplied` | Waits for a GE to be applied to self/target |

### Custom Ability Tasks

See `references/ability-task-reference.md` for the full list and pattern for creating custom tasks.

---

## Cooldowns and Costs

### Cooldown GameplayEffect

Create a GE with:
- Duration Policy: Has Duration
- Add a modifier (optional, e.g., reduce a cooldown attribute)
- Grant a tag like `Cooldown.Fireball`

In the ability:
```cpp
UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
if (CooldownGE)
{
    FGameplayEffectSpecHandle CooldownSpec = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(), GetAbilityLevel());
    ApplyGameplayEffectToOwner(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, CooldownSpec);
}
```

### Cost GameplayEffect

Create a GE with:
- Duration Policy: Instant
- Modifier: Subtract from resource attribute (Health, Mana, Stamina)

In the ability, set `CostGameplayEffectClass` in the ability defaults.

---

## Prediction

### Client-side Prediction

GAS handles prediction automatically for:
- Ability activation/deactivation
- Attribute changes from instant effects
- GameplayTag changes

### Server-side Authority

For critical logic (damage, death, spawning), always validate on the server.
Use `HasAuthority()` checks in `PostGameplayEffectExecute` to ensure server-only execution.

---

## Failed Tasks

### Common Issues

| Symptom | Likely Cause |
|---------|--------------|
| Ability doesn't activate | Missing tag requirements, blocked tags, or ASC not initialized |
| Attributes not replicating | Missing `DOREPLIFETIME` or `ReplicatedUsing` on property |
| Client doesn't see attribute changes | Missing `OnRep` with `GAMEPLAYATTRIBUTE_REPNOTIFY` |
| Death not triggering | Logic in `PreAttributeChange` instead of `PostGameplayEffectExecute` |
| Values not clamping | Missing `PreAttributeChange` override or clamping logic |

### Debugging

- Enable `log AbilitySystem` in console
- Use `AbilitySystem.Debug.NextCategory` to cycle debug displays
- Check `ASC->GetOwnerActor()` and `ASC->GetAvatarActor()` are valid
- Verify `InitAbilityActorInfo` was called on both server and client

---

## References

See the `references/` directory for:
- `gas-setup-patterns.md` — Full initialization code for PlayerState and Character patterns
- `ability-task-reference.md` — Complete list of built-in tasks and custom task template
- `effect-configuration.md` — Detailed GameplayEffect setup with stacking, tags, and modifiers