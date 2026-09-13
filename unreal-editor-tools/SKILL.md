---
name: unreal-editor-tools
description: Unreal Editor extensions, menus, toolbars, detail customization, editor modes, Blutilities, editor utility workflows, custom editor UX.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - editor
  - tooling
  - blutility
  - detail customization
  - tool menus
  - editor utility widget
---

# Unreal Editor Tools

Covers extending the Unreal Editor with custom tools and editor workflows.

## Trigger Words

Use this skill when the user mentions:
- "Editor Utility Widget"
- "Blutility"
- "UToolMenus"
- "FExtender"
- "detail customization"
- "toolbar"
- "Content Browser menu"
- "EditorSubsystem"

## Use When
- Building editor-only UIs, actions, menus, toolbars, and context entries
- Customizing details panels with `IDetailCustomization` or `IPropertyTypeCustomization`
- Creating editor modes, asset editors, or editor subsystems
- Integrating custom commands and shortcuts into the Unreal Editor

## Core Rules
- Editor-extending code belongs in an Editor module, not a Runtime module.
- Keep `UnrealEd` and `PropertyEditor` dependencies out of shipping runtime code.
- Prefer `UToolMenus` for modern menu/toolbar extension where possible.
- Pair with `unreal-module-build` when module dependencies or editor module setup are involved.

## Related Skills
- `>unreal-auto-assistant`
- `unreal-module-build`
- `unreal-umg-lifecycle`
- `unreal-python`
