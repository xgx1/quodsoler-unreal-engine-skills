---
name: unreal-umg-lifecycle
description: UUserWidget lifecycle: construction, teardown, viewport attachment, GC. Widget creation/removal/persistence, duplicate-instance bugs in Unreal UI.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - umg
  - lifecycle
  - uuserwidget
  - viewport
  - gc
  - addtoviewport
  - removefromparent
---

# Unreal UMG Lifecycle

Owns the lifetime of runtime UMG widgets.

## Trigger Words

Use this skill when the user mentions:
- "NativeConstruct"
- "NativeDestruct"
- "CreateWidget"
- "AddToViewport"
- "AddToPlayerScreen"
- "RemoveFromParent"
- "widget 生命周期"
- "widget 重复创建"
- "visibility toggle"
- "show/hide widget"
- "persistent widget"
- "widget manager"
- "event-driven"
- "tick replacement"
- "replace tick"
- "OnHealthChanged"
- "delegate binding"
- "modify existing widget"
- "change my widget"
- "update widget"

## Use When
- Creating widgets in C++ or Blueprint-backed C++ classes
- Choosing `CreateWidget`, `AddToViewport`, or `AddToPlayerScreen`
- Handling `NativeConstruct`, `NativeDestruct`, `NativeTick`
- Fixing leaked widgets, duplicate overlays, or widgets disappearing after removal
- Implementing toggleable UI (inventory, settings, menus) that should persist between show/hide cycles

## Core Rules
- Create player-owned runtime widgets with the owning player, not a bare world.
- Bind delegates in `NativeConstruct` after `BindWidget` references are valid.
- Unbind timers/delegates and clear transient state in `NativeDestruct`.
- `RemoveFromParent()` detaches the widget; it may be garbage collected unless another `UPROPERTY` holds it.
- Prefer `Collapsed` for invisible-but-retained widgets; prefer `RemoveFromParent()` when the widget should truly go away.
- `NativeDestruct` fires when widget is removed from viewport, NOT when visibility changes. For visibility toggling, handle cleanup in a custom `OnHidden()`/`OnShown()` pattern instead.
- For NEW widget code generation (e.g., 'write a health bar widget that uses events'), generate the complete code snippet directly without tools. For MODIFYING existing widgets (e.g., 'change my health bar to use events instead of tick'), ALWAYS request the Read tool first to see the current implementation, then use Edit to make targeted changes.
- When the user asks to modify existing project files (e.g., 'replace tick in my health bar widget'), you MUST request the Read tool to see the current implementation first. Only generate code snippets without tools when the user asks for a NEW widget from scratch or asks for a code pattern example.

## Baseline Pattern

```cpp
UCLASS()
class MYGAME_API UMyWidget : public UUserWidget
{
    GENERATED_BODY()
protected:
    virtual void NativeConstruct() override;
    virtual void NativeDestruct() override;

    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UButton> CloseButton;
};

void UMyWidget::NativeConstruct()
{
    Super::NativeConstruct();
    if (CloseButton)
    {
        CloseButton->OnClicked.AddDynamic(this, &UMyWidget::HandleCloseClicked);
    }
}

void UMyWidget::NativeDestruct()
{
    if (CloseButton)
    {
        CloseButton->OnClicked.RemoveAll(this);
    }
    Super::NativeDestruct();
}
```

When asked to implement this pattern, provide the complete code immediately. Do not describe tools you would use or steps you would take - just write the code.

## Attachment Patterns

```cpp
UMyWidget* Widget = CreateWidget<UMyWidget>(GetOwningPlayer(), WidgetClass);
Widget->AddToViewport(10);

UPROPERTY()
TObjectPtr<UMyWidget> CachedWidget;
CachedWidget = Widget;
```

```cpp
// Split-screen or per-player UI
Widget->AddToPlayerScreen(0);
```

## Visibility Toggle Pattern (Avoid Create/Destroy)

When toggling a widget repeatedly (e.g., inventory, settings), use visibility to avoid recreate costs and state loss:

```cpp
// Manager class or GameInstance subsystem
UCLASS()
class MYGAME_API UWidgetManager : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    UPROPERTY()
    TObjectPtr<UInventoryWidget> InventoryWidget;

    void ToggleInventory(APlayerController* PC)
    {
        if (!InventoryWidget)
        {
            InventoryWidget = CreateWidget<UInventoryWidget>(PC, InventoryWidgetClass);
            InventoryWidget->AddToViewport();
        }
        
        // Toggle visibility - widget stays in viewport but hidden
        ESlateVisibility CurrentVis = InventoryWidget->GetVisibility();
        if (CurrentVis == ESlateVisibility::Visible)
        {
            InventoryWidget->SetVisibility(ESlateVisibility::Collapsed);
            // Optionally show mouse cursor, set input mode
        }
        else
        {
            InventoryWidget->SetVisibility(ESlateVisibility::Visible);
            InventoryWidget->RefreshInventory(); // Rebuild data if needed
        }
    }
};
```

```cpp
// In widget NativeConstruct/NativeDestruct for visibility-aware cleanup
void UInventoryWidget::NativeConstruct()
{
    Super::NativeConstruct();
    // Bind only once - widget lives forever
    if (!CloseButton->OnClicked.IsAlreadyBound(this, &UInventoryWidget::HandleClose))
    {
        CloseButton->OnClicked.AddDynamic(this, &UInventoryWidget::HandleClose);
    }
}

void UInventoryWidget::HandleClose()
{
    // Don't RemoveFromParent - just hide
    SetVisibility(ESlateVisibility::Collapsed);
    // Notify manager or set input mode
}
```

Key rules:
- Store widget in a `UPROPERTY()` on a persistent object (GameInstance, PlayerController, subsystem)
- Use `SetVisibility(Collapsed)` instead of `RemoveFromParent()` for toggleable UI
- Bind delegates once in `NativeConstruct` (check `IsAlreadyBound` to avoid duplicates)
- Never call `RemoveFromParent()` on toggled widgets unless truly destroying them
- Call `Refresh()` or equivalent when showing to update stale data

## Diagnostics
- Widget shows twice: verify only one creation path and cache the instance.
- Widget vanishes after close/open: keep a strong `UPROPERTY` reference if reusing it.
- Hidden widget still costs CPU: avoid heavy `NativeTick`, or early-out when not visible.
- Input breaks after reopen: pair this skill with `unreal-umg-input` or `unreal-common-ui`.
- Widget response cut off or incomplete: ensure visibility toggling pattern is used instead of create/destroy, and that manager class properly owns the widget reference.

## UMG / Slate / Hybrid 决策

- 标准游戏 HUD/菜单 → UMG（`UUserWidget` 蓝图/C++）。
- 自定义渲染/输入行为，UMG 无法干净表达 → Slate。
- `UWidget` 宿主需要嵌入自定义 Slate 内容 → hybrid 桥接。

每个 UI 任务必须定义：UI 层归属（UMG-only / Slate-only / hybrid）、数据源与更新触发（pull/push/event/mixed）、焦点与输入归属切换、tooltip/popup 的视口安全放置、teardown/cleanup 路径（解绑 + 控件移除）。缺任一项即 UI 实现不完整。

## Slate 桥接（UMG 宿主嵌入 Slate）

- `UWidget::TakeWidget()` 是 UMG→Slate 桥接交接点：返回 `TSharedRef<SWidget>`，供 Slate 容器嵌入或 `SetWidgetToFocus` 使用。
- 自绘 Slate 控件用 `SCompoundWidget` + `SLATE_BEGIN_ARGS(SMyWidget)` / `SLATE_END_ARGS()` 声明，`Construct(const FArguments&)` 中组装子控件。
- Slate-only 代码必须隔离在清晰桥接边界后（如 `UWidget` 子类内部持有 `TSharedPtr<SMySlate>`），不向 UMG 层泄漏 Slate 细节。
- 焦点切换：`FSlateApplication::Get().SetKeyboardFocus(...)` / `SetUserFocus(...)`。
- 需要引擎级 Slate 深度定制（超出项目作用域）时升级处理。

## Tooltip / Popup 视口 Clamp

- 用锚点 + 光标/控件几何计算期望位置；`UGameViewportClient::GetViewportSize(...)` 取视口尺寸。
- 最终位置钳制到视口边界内，避免屏幕外渲染。
- 高频 hover 更新加防抖，避免闪烁。
- 故障模式：tooltip 在屏幕边缘抖动 → clamp 输出振荡 + hover 源抖动 → 防抖 hover 更新 + 用稳定视口度量 clamp。
- 数据源：绑定自唯一权威源（subsystem/component/view model），事件驱动刷新优先于每帧轮询。

## Related Skills
- `unreal-umg-binding`
- `unreal-umg-input`