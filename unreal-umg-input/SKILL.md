---
name: unreal-umg-input
description: UMG focus, cursor visibility, input-mode ownership: FInputMode setup, keyboard focus, game/UI transitions, non-CommonUI navigation bugs.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - umg
  - input
  - focus
  - cursor
  - inputmode
  - keyboard focus
---

# Unreal UMG Input

Owns raw runtime input behavior for non-CommonUI widgets.

## Trigger Words

Use this skill when the user mentions:
- "FInputModeUIOnly"
- "FInputModeGameAndUI"
- "FInputModeGameOnly"
- "keyboard focus"
- "鼠标光标"
- "输入模式"
- "焦点丢失"

## Use When
- Setting `FInputModeUIOnly`, `FInputModeGameAndUI`, or `FInputModeGameOnly`
- Controlling mouse cursor visibility for menus or HUD overlays
- Setting keyboard focus on the intended widget
- Fixing focus loss after opening or closing a UMG screen

## Baseline Patterns

```cpp
FInputModeUIOnly UIMode;
UIMode.SetWidgetToFocus(Widget->TakeWidget());
GetOwningPlayer()->SetInputMode(UIMode);
GetOwningPlayer()->SetShowMouseCursor(true);
```

```cpp
FInputModeGameAndUI GameUIMode;
GameUIMode.SetLockMouseToViewportBehavior(EMouseLockMode::LockOnCapture);
GetOwningPlayer()->SetInputMode(GameUIMode);
GetOwningPlayer()->SetShowMouseCursor(true);
```

```cpp
GetOwningPlayer()->SetInputMode(FInputModeGameOnly());
GetOwningPlayer()->SetShowMouseCursor(false);
```

```cpp
// Save and restore focus for modal dialogs
TSharedPtr<SWidget> PreviousFocusedWidget;

void OpenModalDialog(UUserWidget* DialogWidget)
{
    // Save current focus before opening modal
    PreviousFocusedWidget = FSlateApplication::Get().GetUserFocusedWidget(0);
    
    FInputModeUIOnly UIMode;
    UIMode.SetWidgetToFocus(DialogWidget->TakeWidget());
    GetOwningPlayer()->SetInputMode(UIMode);
    GetOwningPlayer()->SetShowMouseCursor(true);
}

void CloseModalDialog()
{
    // Restore focus to previous widget or fall back to player controller
    if (PreviousFocusedWidget.IsValid())
    {
        FSlateApplication::Get().SetUserFocus(0, PreviousFocusedWidget);
    }
    
    GetOwningPlayer()->SetInputMode(FInputModeGameOnly());
    GetOwningPlayer()->SetShowMouseCursor(false);
    PreviousFocusedWidget.Reset();
}
```

```cpp
// Tool selection pattern for focus restoration tasks:
// 1. Use 'Read' to inspect existing modal dialog code
// 2. Use 'Edit' to modify the focus save/restore logic
// 3. Do NOT use 'Write' unless creating a brand new file
```

## Blueprint 便捷 API（UWidgetBlueprintLibrary）

- `UWidgetBlueprintLibrary::SetInputMode_UIOnlyEx(PlayerController, InWidgetToFocus, InMouseLockMode, bHideCursorDuringCapture)` — 蓝图可直接调用的 UIOnly 变体，等价 `FInputModeUIOnly` + `SetWidgetToFocus` + `SetLockMouseToViewportBehavior`。
- `UWidgetBlueprintLibrary::SetInputMode_GameAndUIEx(...)` — GameAndUI 变体，参数同上。
- `UWidgetBlueprintLibrary::SetInputMode_GameOnly(PlayerController)` — GameOnly 变体。
- 规则同 `FInputMode` 系：控件上屏后再设输入模式；关闭全屏菜单时恢复游戏输入归属。

## Rules
- Set input mode from the caller after the widget is on screen, not inside `NativeConstruct()` unless the flow is tightly controlled.
- Restore gameplay ownership when closing full-screen menus.
- Keep focus targets explicit for keyboard/controller navigation.
- Save focus target before opening modal dialogs and restore it on close using `FSlateApplication::Get().GetUserFocusedWidget()` / `SetUserFocus()`.
- Use `IsValid()` checks on saved focus targets before restoration, as widgets may have been destroyed.
- If the widget uses Common UI, route to `unreal-common-ui` instead of mixing systems.
- When fixing existing focus management code, use `Edit` (not `Write`) to modify the existing implementation. Use `Read` first to inspect the current code, then `Edit` to apply changes. Only use `Write` when creating entirely new files.

## Failure Patterns
- Keyboard stops responding after popup close: focus was not restored.
- Cursor stays visible in gameplay: menu teardown did not restore game-only mode.
- UI swallows gameplay input unexpectedly: wrong input mode or stale focus target.
- Focus saved before dialog open references destroyed widget: always check `IsValid()` before restoring.
- Focus not saved at all before opening modal: always capture `GetUserFocusedWidget()` before changing input mode.
- `Write` used instead of `Edit` when modifying existing focus restoration code: always use `Edit` to modify existing files, `Write` only for new files.

## Related Skills
- `unreal-umg-lifecycle`
- `unreal-common-ui`