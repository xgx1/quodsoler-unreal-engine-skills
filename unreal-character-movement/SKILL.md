---
name: unreal-character-movement
description: "Unreal character movement: CharacterMovementComponent, CMC, walk/fall/swim/fly, network prediction, FSavedMove, root motion, floor detection, step-up."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - character
  - movement
---

# Unreal Character Movement

## Trigger Words

Use this skill when the user mentions:
- "character"
- "movement"
- "character movement"

## Context Check

Read `.agents/unreal-project-context.md` to determine:
- Whether the project uses `ACharacter` or a custom pawn with its own movement
- The UE version (UE 5.4+ adds `GravityDirection` support, UE 5.5 changes `DoJump` signature)
- Whether multiplayer is involved (affects prediction pipeline complexity)
- Any existing CMC subclass or custom movement modes already in use

## Information Gathering

Ask the developer:
1. Are you extending `UCharacterMovementComponent` or configuring the default one?
2. Do you need custom movement modes (wall-running, climbing, dashing)?
3. Is this multiplayer? If so, do custom abilities need network prediction?
4. Are you integrating root motion from animations or gameplay code?
5. Do you need custom gravity directions (UE 5.4+)?

---
## Tool Naming Convention

**Critical**: All tool names must be **lowercase** when invoked. 
- ✅ Correct: `read`, `write`, `edit`
- ❌ Incorrect: `Read`, `Write`, `Edit`, `READ`

The system is case-sensitive. Using uppercase will cause tool call failures.

---

## Tool Usage Guidelines

When implementing movement features, use these tools appropriately:

- **read**: Use to examine existing code, project context (`.agents/unreal-project-context.md`), or reference files
- **write**: Use to create new source files (e.g., new ability classes, new CMC subclasses)
- **edit**: Use to modify existing source files (e.g., adding `LaunchCharacter` calls to character classes)

**Important**: Always use the correct tool name (lowercase) and valid parameters. Invalid tool calls will fail.

---

## Common Configuration Patterns

### Basic Movement Properties

Configure movement properties in `ACharacter` subclass constructor or `BeginPlay`:

```cpp
// In AMyCharacter::AMyCharacter()
GetCharacterMovement()->MaxWalkSpeed = 600.0f;
GetCharacterMovement()->MaxAcceleration = 2048.0f;
GetCharacterMovement()->AirControl = 0.8f;
GetCharacterMovement()->JumpZVelocity = 600.0f;
GetCharacterMovement()->BrakingDecelerationWalking = 800.0f;
GetCharacterMovement()->BrakingFrictionFactor = 1.0f;
```

### Crouching Setup

Crouching requires both capsule half-height and CMC configuration:

```cpp
// In AMyCharacter::AMyCharacter()
// 1. Set capsule half-height (default character height)
GetCapsuleComponent()->InitCapsuleSize(42.0f, 96.0f); // Radius, HalfHeight

// 2. Enable crouching on the CMC
GetCharacterMovement()->GetNavAgentPropertiesRef().bCanCrouch = true;
GetCharacterMovement()->CrouchedHalfHeight = 40.0f; // Crouched capsule half-height

// 3. Optional: adjust crouching speed
GetCharacterMovement()->MaxWalkSpeedCrouched = 300.0f;
```

For blueprint-assignable values, expose to BlueprintReadWrite:
```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
float MaxWalkSpeed = 600.0f;

// In BeginPlay or constructor
GetCharacterMovement()->MaxWalkSpeed = MaxWalkSpeed;
```

### Jump Configuration

```cpp
// Jump height is determined by JumpZVelocity and gravity
GetCharacterMovement()->JumpZVelocity = 600.0f;       // Initial upward velocity
GetCharacterMovement()->GravityScale = 1.0f;           // Default gravity multiplier
GetCharacterMovement()->AirControl = 0.8f;              // 0.0 = no air control, 1.0 = full
GetCharacterMovement()->AirControlBoostMultiplier = 2.0f; // Boost when holding forward
GetCharacterMovement()->AirControlBoostVelocityThreshold = 25.0f; // Boost threshold
```

### Full FPS Configuration Example

```cpp
// AMyFPSCharacter constructor
AMyFPSCharacter::AMyFPSCharacter()
{
    // Capsule
    GetCapsuleComponent()->InitCapsuleSize(42.0f, 96.0f);
    
    // Movement speeds
    UCharacterMovementComponent* CMC = GetCharacterMovement();
    CMC->MaxWalkSpeed = 600.0f;
    CMC->MaxWalkSpeedCrouched = 300.0f;
    CMC->MaxAcceleration = 2048.0f;
    CMC->BrakingDecelerationWalking = 800.0f;
    
    // Jump & air
    CMC->JumpZVelocity = 600.0f;
    CMC->AirControl = 0.8f;
    CMC->GravityScale = 1.0f;
    
    // Crouching
    CMC->GetNavAgentPropertiesRef().bCanCrouch = true;
    CMC->CrouchedHalfHeight = 40.0f;
    
    // Rotation
    CMC->bOrientRotationToMovement = false;  // FPS: don't rotate character to movement
    CMC->bUseControllerDesiredRotation = true; // Use controller yaw
    CMC->RotationRate = FRotator(0.0f, 540.0f, 0.0f);
}
```

### LaunchCharacter Implementation Pattern

When implementing a `LaunchCharacter` ability with cooldown:

1. **Create a new ability class** (use write tool):
   - Define cooldown timer (e.g., `FTimerHandle CooldownTimerHandle`)
   - Implement ability activation with `LaunchCharacter(LaunchVelocity, bXYOverride, bZOverride)`

2. **Modify character class** (use edit tool):
   - Add ability reference member variable
   - Bind input to trigger ability

3. **Example** (add to character header):
   ```cpp
   UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Abilities")
   float LaunchCooldown = 2.0f;
   
   UFUNCTION(BlueprintCallable, Category = "Abilities")
   void PerformLaunch(FVector LaunchVelocity);
   ```

**Common mistake**: Using `read` tool to create new files. Use `write` for new files, `edit` for existing files.

---

## Workflow for Common Tasks

### Configuring Movement Properties
1. Read `.agents/unreal-project-context.md` for existing CMC subclass
2. If using default CMC: configure via `GetCharacterMovement()` in constructor/BeginPlay
3. If using custom CMC subclass: add `UPROPERTY` exposed properties and apply in constructor
4. Write the configuration code (see Common Configuration Patterns above)

### Setting Up Crou