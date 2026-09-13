---
name: unreal-python
description: "Unreal Python editor scripting: asset automation, batch operations, scene setup, tool scripts. Trigger: Unreal automation scripts, Content/Python workflows."
author: Sx
version: 1.0.0
keywords:
  - unreal
  - python
  - editor scripting
  - automation
  - assets
  - batch
  - content python
---

# Unreal Python

Unreal Engine Python editor scripting for safe automation and batch operations.

## Trigger Words

Use this skill when the user mentions:
- "python脚本"
- "unreal python"
- "editor脚本"
- "批量处理"
- "资产管理脚本"
- "自动化"
- "Content/Python"

## Use When
- Writing Python scripts to automate asset edits, imports, renames, moves, or generation
- Building editor-time tooling without requiring a full C++ editor extension
- Batch-processing content or validating editor-side asset state

## Core Rules
- Default script location is `Content/Python/` unless the project already uses another convention.
- Prefer idempotent scripts for batch operations.
- Pair with `unreal-cmd` when scripts will be executed through `UnrealEditor-Cmd.exe`.
- Pair with `unreal-dev-umg` for automated Widget Blueprint work.

## Related Skills
- `>unreal-auto-assistant`
- `unreal-cmd`
- `unreal-dev-umg`
