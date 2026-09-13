---
name: unreal-common-ui
description: 'CommonUI and CommonInput for Unreal menus and gamepad-first flows: activatable widgets, button styles, screen stacks, back handling, input routing.'
author: Sx
version: 1.0.0
keywords:
  - unreal
  - commonui
  - commoninput
  - activatable widget
  - gamepad
  - menu
  - stack
---

# Unreal Common UI

Owns Epic's Common UI framework and its activation-tree-driven input model.

## Trigger Words

Use this skill when the user mentions:
- "CommonUI"
- "CommonInput"
- "UCommonActivatableWidget"
- "UCommonButtonBase"
- "gamepad menu"
- "激活栈"
- "手柄导航"

## Use When
- The project has `CommonUI` or `CommonInput` enabled
- Building screen stacks, modal flows, or controller-first menus
- Working with `UCommonActivatableWidget`, `UCommonButtonBase`, `UCommonActivatableWidgetStack`, or `UCommonInputSubsystem`
- Diagnosing conflicts between Common UI behavior and raw `SetInputMode` calls

## Setup Requirements
- Enable `CommonUI` and `CommonInput`
- Add module dependencies in `Build.cs`
- Set `GameViewportClientClassName=/Script/CommonUI.CommonGameViewportClient`

## Core Pattern

```cpp
UCLASS()
class MYGAME_API UMyMenuScreen : public UCommonActivatableWidget
{
    GENERATED_BODY()
protected:
    virtual void NativeOnActivated() override;
    virtual void NativeOnDeactivated() override;
    virtual UWidget* NativeGetDesiredFocusTarget() const override { return ConfirmButton; }
    virtual TOptional<FUIInputConfig> GetDesiredInputConfig() const override
    {
        return FUIInputConfig(ECommonInputMode::Menu, EMouseCaptureMode::NoCapture);
    }

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UCommonButtonBase> ConfirmButton;
};
```

```cpp
void UMyMenuScreen::NativeOnActivated()
{
    Super::NativeOnActivated();
    ConfirmButton->OnClicked().AddUObject(this, &UMyMenuScreen::HandleConfirm);
}

void UMyMenuScreen::NativeOnDeactivated()
{
    ConfirmButton->OnClicked().RemoveAll(this);
    Super::NativeOnDeactivated();
}
```

## Containers
- `UCommonActivatableWidgetStack`: stack-style screen layers
- `UCommonActivatableWidgetQueue`: sequential notifications or queued screens
- Do not confuse these with `UWidgetSwitcher`

## Rules
- Use `AddUObject`, not `AddDynamic`, on `UCommonButtonBase::OnClicked()`.
- Do not bypass the framework with raw `SetInputMode` unless you are intentionally integrating systems.
- Always call `Super::NativeOnActivated()` and `Super::NativeOnDeactivated()`.
- Define the desired focus target and back-handler behavior deliberately.

## Related Skills
- `unreal-umg-lifecycle`
- `unreal-umg-binding`
- `unreal-umg-input`
