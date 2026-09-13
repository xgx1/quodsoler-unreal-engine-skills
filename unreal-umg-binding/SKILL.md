---
name: unreal-umg-binding
description: BindWidget contracts between C++ and Widget Blueprints: name matching, optional/animation bindings, missing-control compile errors.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - umg
  - bindwidget
  - widget blueprint
  - binding
  - bindwidgetoptional
  - bindwidgetanim
---

# Unreal UMG Binding

Owns the contract between C++ widget classes and Widget Blueprint designer trees.

## Trigger Words

Use this skill when the user mentions:
- "BindWidget"
- "BindWidgetOptional"
- "BindWidgetAnim"
- "missing binding"
- "控件绑定丢失"
- "Widget Blueprint 命名"
- "蓝图控件树"

## Use When
- Declaring `BindWidget`, `BindWidgetOptional`, or `BindWidgetAnim`
- Matching widget names exactly between C++ and Blueprint
- Fixing compile errors caused by missing bound controls
- Deciding which controls must be bound versus purely visual children

## Binding Rules
- `BindWidget` means the Blueprint must contain a widget with the exact same case-sensitive name.
- `BindWidgetOptional` must always be null-checked before use.
- `BindWidgetAnim` should be `Transient`.
- Do not guess names. If a control is required by C++, read the header and mirror it exactly.
- Keep business-owned controls bound; keep purely decorative children unbound unless code truly needs them.

## Example

```cpp
UCLASS()
class MYGAME_API UInventoryPanel : public UUserWidget
{
    GENERATED_BODY()
protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UButton> ConfirmButton;

    UPROPERTY(meta=(BindWidgetOptional))
    TObjectPtr<UTextBlock> SubtitleText;

    UPROPERTY(meta=(BindWidgetAnim), Transient)
    TObjectPtr<UWidgetAnimation> IntroAnim;
};
```

## Good Practices
- Keep required names stable once shipped to multiple Blueprints.
- Prefer update-in-place over deleting and recreating required widgets.
- Separate control binding from style composition.
- Pair this skill with `unreal-dev-umg` when repairing assets by script.

## Failure Patterns
- Compile says missing control: the Blueprint tree is missing the widget or the name differs.
- Binding exists but pointer is null: wrong widget type, wrong hierarchy asset, or stale generated class.
- Animation binding fails: missing `Transient` or animation name mismatch.

## Related Skills
- `unreal-umg-lifecycle`
- `unreal-dev-umg`
