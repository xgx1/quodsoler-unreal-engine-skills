---
name: unreal-input-system
description: Enhanced Input, gameplay input mapping in Unreal: InputAction, InputMappingContext, triggers, modifiers, gamepad, keyboard, input binding setup.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - input
  - enhanced input
  - inputaction
  - mapping context
  - triggers
  - modifiers
---

# Unreal Input System

## Trigger Words

Use this skill when the user mentions:
- "Enhanced Input"
- "InputAction"
- "InputMappingContext"
- "trigger"
- "modifier"
- "gamepad"
- "keyboard"
- "input binding"

You are an expert in Unreal Engine's Enhanced Input system.

## Context Check

Read `.agents/unreal-project-context.md` before proceeding. Confirm:

- `EnhancedInput` plugin is listed as enabled
- Target platforms (affects which modifiers are needed per platform)
- Whether CommonUI is in use (it manages input mode switching automatically)
- Whether the project still uses legacy input (migration may be needed)

- **IMPORTANT**: If the user's request is self-contained (e.g., "create a custom X class" with all details provided), skip the file read and proceed directly to providing the code. Only read the context file when the task requires project-specific information (e.g., existing class names, plugin status, platform details).

## Information Gathering

Ask the developer: what actions are needed and their value types (Bool/Axis1D/Axis2D/Axis3D), which platforms, any complex input requirements (hold-to-charge, double-tap, combos, chord shortcuts), and whether multiple input modes are required (gameplay vs UI vs vehicle).

---

## Enhanced Input Setup

### Plugin and Module

`.uproject`: add `{ "Name": "EnhancedInput", "Enabled": true }` to Plugins.

`Build.cs`: add `"EnhancedInput"` to `PublicDependencyModuleNames`.

`DefaultInput.ini`:
```ini
[/Script/Engine.InputSettings]
DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

### UInputAction Asset

`UInputAction : UDataAsset`. Create one per logical player action. Key properties (from `InputAction.h`):

```cpp
EInputActionValueType ValueType = EInputActionValueType::Boolean;
// Boolean | Axis1D (float) | Axis2D (FVector2D) | Axis3D (FVector)

EInputActionAccumulationBehavior AccumulationBehavior
    = EInputActionAccumulationBehavior::TakeHighestAbsoluteValue;
// TakeHighestAbsoluteValue — highest magnitude wins across all mappings to this action
// Cumulative — all mapping values sum (W + S cancel each other for WASD)

bool bConsumeInput = true;  // blocks lower-priority Enhanced Input mappings to same keys

TArray<TObjectPtr<UInputTrigger>> Triggers;   // applied AFTER per-mapping triggers
TArray<TObjectPtr<UInputModifier>> Modifiers; // applied AFTER per-mapping modifiers
```

### UInputMappingContext Asset

`UInputMappingContext : UDataAsset`. Maps physical keys to actions.

- `DefaultKeyMappings.Mappings` — `TArray<FEnhancedActionKeyMapping>` of key-to-action entries
- `MappingProfileOverrides` — per-profile key overrides for player remapping support
- `RegistrationTrackingMode`: `Untracked` (default, first Remove wins) or `CountRegistrations` (IMC stays until Remove called N times, safe when multiple systems share it)

---

## Binding Actions in C++

### SetupPlayerInputComponent

```cpp
// MyCharacter.h — declare assets and handlers
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputMappingContext> DefaultMappingContext;
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> MoveAction;
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
TObjectPtr<UInputAction> JumpAction;

void Move(const FInputActionValue& Value);
void StartJump();
void StopJump();
```

```cpp
// MyCharacter.cpp
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);
    UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    if (!EIC) { return; }

    EIC->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyCharacter::Move);
    EIC->BindAction(JumpAction, ETriggerEvent::Started,   this, &AMyCharacter::StartJump);
    EIC->BindAction(JumpAction, ETriggerEvent::Completed, this, &AMyCharacter::StopJump);
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    if (APlayerController* PC = Cast<APlayerController>(GetController()))
    {
        if (auto* Sub = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(
                PC->GetLocalPlayer()))
        {
            Sub->AddMappingContext(DefaultMappingContext, 0); // priority 0 = lowest
        }
    }
}
```

### Callback Signatures

`BindAction` accepts four delegate signatures:

```cpp
// No params — press/release without value needed
void AMyCharacter::StartJump() { Jump(); }

// FInputActionValue — for axis values
void AMyCharacter::Move(const FInputActionValue& Value)
{
    const FVector2D Input = Value.Get<FVector2D>();
    AddMovementInput(GetActorForwardVector(), Input.Y);
    AddMovementInput(GetActorRightVector(),   Input.X);
}

// FInputActionInstance — when elapsed/triggered time is needed
void AMyCharacter::OnChargeAttack(const FInputActionInstance& Instance)
{
    const float HeldFor = Instance.GetElapsedTime();    // Started + Ongoing + Triggered
    const float ActiveFor = Instance.GetTriggeredTime(); // Triggered only
}

// Lambda variant
EIC->BindActionValueLambda(InteractAction, ETriggerEvent::Triggered,
    [this](const FInputActionValue& Value) { TryInteract(); });
```

Storing and removing a binding:
```cpp
FEnhancedInputActionEventBinding& B =
    EIC->BindAction(DebugAction, ETriggerEvent::Started, this, &AMyCharacter::DebugToggle);
uint32 Handle = B.GetHandle();
// ...
EIC->RemoveBindingByHandle(Handle);        // remove one binding
EIC->ClearBindingsForObject(this);         // remove all bindings for an object
```

---

## Trigger Events (ETriggerEvent)

Bitmask enum from `InputTriggers.h`:

| Event | State Transition | Use for |
|---|---|---|
| `Started` | None -> Ongoing/Triggered | First frame of input; press-once actions |
| `Triggered` | *->Triggered, Triggered->Tr |

---

## Custom UInputModifier Example

When asked to create a custom modifier, provide the full C++ implementation immediately without reading project context files first. Example:

```cpp
// MyInputModifiers.h
#pragma once

#include "CoreMinimal.h"
#include "InputModifiers.h"
#include "MyInputModifiers.generated.h"

/** Clamps input magnitude to MaxMagnitude, preserving direction */
UCLASS()
class MYPROJECT_API UInputModifierClampMagnitude : public UInputModifier
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Settings")
    float MaxMagnitude = 1.0f;

protected:
    virtual FInputActionValue ModifyRaw_Implementation(
        const UEnhancedPlayerInput* PlayerInput,
        FInputActionValue CurrentValue,
        float DeltaTime) override;
};
```

```cpp
// MyInputModifiers.cpp
#include "MyInputModifiers.h"
#include "InputActionValue.h" // for FInputActionValue

FInputActionValue UInputModifierClampMagnitude::ModifyRaw_Implementation(
    const UEnhancedPlayerInput* PlayerInput,
    FInputActionValue CurrentValue,
    float DeltaTime)
{
    // Get the input value type and raw data
    const EInputActionValueType ValueType = CurrentValue.GetValueType();
    
    // Handle Axis1D (float) values
    if (ValueType == EInputActionValueType::Axis1D)
    {
        float Value = CurrentValue.Get<float>();
        Value = FMath::Clamp(Value, -MaxMagnitude, MaxMagnitude);
        return FInputActionValue(Value);
    }
    
    // Handle Axis2D (FVector2D) values
    if (ValueType == EInputActionValueType::Axis2D)
    {
        FVector2D Value = CurrentValue.Get<FVector2D>();
        float Magnitude = Value.Size();
        if (Magnitude > MaxMagnitude && Magnitude > KINDA_SMALL_NUMBER)
        {
            Value *= (MaxMagnitude / Magnitude);
        }
        return FInputActionValue(Value);
    }
    
    // Handle Axis3D (FVector) values
    if (ValueType == EInputActionValueType::Axis3D)
    {
        FVector Value = CurrentValue.Get<FVector>();
        float Magnitude = Value.Size();
        if (Magnitude > MaxMagnitude && Magnitude > KINDA_SMALL_NUMBER)
        {
            Value *= (MaxMagnitude / Magnitude);
        }
        return FInputActionValue(Value);
    }
    
    // Boolean values: pass through unchanged
    return CurrentValue;
}