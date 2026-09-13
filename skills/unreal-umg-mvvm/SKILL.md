---
name: unreal-umg-mvvm
description: 'UE5 MVVM and event-driven UI data flow for Unreal: ViewModels, FieldNotify, push-based refresh, separating gameplay state from widget state.'
author: Sx
version: 1.0.0
keywords:
  - unreal
  - umg
  - mvvm
  - fieldnotify
  - viewmodel
  - data binding
  - ue5
---

# Unreal UMG MVVM

Owns UI data flow and ViewModel design.

## Trigger Words

Use this skill when the user mentions:
- "MVVM"
- "ViewModel"
- "FieldNotify"
- "数据绑定"
- "事件驱动刷新"
- "状态同步"
- "ModelViewViewModel"

## Use When
- The project enables `ModelViewViewModel`
- A widget needs stable, event-driven refresh without manual polling
- Separating gameplay systems from display formatting
- Reviewing whether a screen should use ViewModel objects instead of direct actor coupling

## Principles
- Gameplay systems own truth; widgets display state.
- ViewModels translate gameplay state into UI-facing fields.
- Prefer push-based updates through `FieldNotify` over per-frame pull logic.
- Keep mutation logic outside passive display widgets whenever possible.

## Example

```cpp
UCLASS()
class MYGAME_API UMyGameViewModel : public UMVVMViewModelBase
{
    GENERATED_BODY()
public:
    void SetScore(int32 NewScore) { UE_MVVM_SET_PROPERTY_VALUE(Score, NewScore); }
    int32 GetScore() const { return Score; }

private:
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    int32 Score = 0;
};
```

## Concrete HUD ViewModel Pattern

```cpp
UCLASS()
class MYGAME_API UPlayerHUDViewModel : public UMVVMViewModelBase
{
    GENERATED_BODY()

public:
    // Health
    void SetHealth(float NewHealth) { UE_MVVM_SET_PROPERTY_VALUE(Health, NewHealth); }
    float GetHealth() const { return Health; }
    
    void SetMaxHealth(float NewMaxHealth) { UE_MVVM_SET_PROPERTY_VALUE(MaxHealth, NewMaxHealth); }
    float GetMaxHealth() const { return MaxHealth; }
    
    // Ammo
    void SetAmmo(int32 NewAmmo) { UE_MVVM_SET_PROPERTY_VALUE(Ammo, NewAmmo); }
    int32 GetAmmo() const { return Ammo; }
    
    // Score
    void SetScore(int32 NewScore) { UE_MVVM_SET_PROPERTY_VALUE(Score, NewScore); }
    int32 GetScore() const { return Score; }

private:
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    float Health = 100.0f;
    
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    float MaxHealth = 100.0f;
    
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    int32 Ammo = 30;
    
    UPROPERTY(BlueprintReadOnly, FieldNotify, Getter, meta=(AllowPrivateAccess))
    int32 Score = 0;
};
```

## Recommended Flow
- Gameplay component/subsystem receives the real event.
- ViewModel converts it to widget-ready properties.
- Widget Blueprint binds to the ViewModel.
- Widget avoids manual imperative refresh where bindings are sufficient.

## Blueprint Integration Steps
1. Create ViewModel class in C++ (as shown above)
2. In Widget Blueprint, add a "Viewmodel" entry in Class Defaults
3. Set Viewmodel class to your C++ ViewModel
4. Bind widget properties (e.g., Text blocks) to ViewModel fields via the binding dropdown
5. The FieldNotify attribute automatically triggers UI updates when values change

## Gameplay Integration Pattern
```cpp
// In your player state or game mode:
UPlayerHUDViewModel* HUDViewModel = NewObject<UPlayerHUDViewModel>(this);
HUDViewModel->SetHealth(CurrentHealth);
HUDViewModel->SetScore(CurrentScore);

// Pass to widget:
UUserWidget* HUDWidget = CreateWidget<UUserWidget>(GetWorld(), HUDWidgetClass);
Cast<UMyHUDWidget>(HUDWidget)->SetViewmodel(HUDViewModel);
```

## Escalate To MVVM When
- Many controls reflect the same state domain.
- The same data drives multiple widgets.
- Polling and ad-hoc event wiring are becoming fragile.

## Related Skills
- `unreal-umg-lifecycle`
- `unreal-umg-lists`
- `unreal-umg-lifecycle`