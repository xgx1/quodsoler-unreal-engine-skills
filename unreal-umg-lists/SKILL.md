---
name: unreal-umg-lists
description: 'UListView, UTileView, entry widget virtualization in Unreal UMG: list population, entry refresh, selection, scrolling, IUserObjectListEntry.'
author: Sx
version: 1.0.0
keywords:
  - unreal
  - umg
  - listview
  - tileview
  - virtualization
  - list entry
  - userobjectlistentry
---

# Unreal UMG Lists

Owns virtualized list and tile workflows.

## Trigger Words

Use this skill when the user mentions:
- "ListView"
- "TileView"
- "IUserObjectListEntry"
- "虚拟列表"
- "列表项刷新"
- "entry widget"
- "列表滚动"

## Use When
- Implementing `UListView` or `UTileView`
- Writing entry widgets with `IUserObjectListEntry`
- Populating list items from data objects
- Fixing list refresh, selection, or recycled entry bugs

## Core Model
- Data objects are not widgets.
- Entry widgets are recycled as the list scrolls.
- Entry widgets should render data, not own authoritative gameplay state.

## Entry Pattern

```cpp
UCLASS()
class UItemEntryWidget : public UUserWidget, public IUserObjectListEntry
{
    GENERATED_BODY()
protected:
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UTextBlock> NameText;

    virtual void NativeOnListItemObjectSet(UObject* ListItemObject) override
    {
        IUserObjectListEntry::NativeOnListItemObjectSet(ListItemObject);
        if (const UItemData* Data = Cast<UItemData>(ListItemObject))
        {
            NameText->SetText(FText::FromString(Data->ItemName));
        }
    }
};
```

## Populate Pattern

```cpp
ItemList->ClearListItems();
for (const FItemInfo& Info : Items)
{
    UItemData* Data = NewObject<UItemData>(this);
    Data->ItemName = Info.Name;
    ItemList->AddItem(Data);
}
```

## Rules
- Keep list item objects small and display-oriented.
- Rebind every visible field inside `NativeOnListItemObjectSet()` because entries are recycled.
- Do not cache stale widget pointers from list entries unless visibility/lifetime is guaranteed.
- Use `ScrollIndexIntoView()` or `BP_ScrollItemIntoView()` for navigation-driven reveal.

## Common Bugs
- Wrong data appears after scroll: entry widget forgot to refresh all visual fields.
- Selection API seems inconsistent: verify selection mode and whether you need single or multi-select.
- Large list stutters: avoid non-virtualized container patterns and per-frame polling.

## Related Skills
- `unreal-umg-lifecycle`
- `unreal-umg-mvvm`
