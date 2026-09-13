# quodsoler-unreal-engine-skills

DSH 技能分组仓：**quodsoler-unreal-engine-skills**

- **上游**：https://github.com/quodsoler/unreal-engine-skills
- **结构**：本仓的树 = **上游最新树**（2026-09-13 起对齐），上游的目录层级原样保留。
  本地改动叠在对应文件上（改中文、平台分节、改 frontmatter 的 `name:` 等）——
  `git diff upstream/main` 就是「本机改了什么」的权威答案。
- **本文件**（`README.dsh-local.md`）是本地附加的说明，上游没有；上游的 `README.md` 原样保留。

## 本机改写过的技能（24 个）

- `unreal-actor-component-architecture`
- `unreal-ai-navigation`
- `unreal-animation-system`
- `unreal-async-threading`
- `unreal-audio-system`
- `unreal-character-movement`
- `unreal-cpp-foundations`
- `unreal-data-assets-tables`
- `unreal-editor-tools`
- `unreal-game-features`
- `unreal-gameplay-abilities`
- `unreal-gameplay-framework`
- `unreal-input-system`
- `unreal-mass-entity`
- `unreal-materials-rendering`
- `unreal-networking-replication`
- `unreal-niagara-effects`
- `unreal-physics-collision`
- `unreal-procedural-generation`
- `unreal-sequencer-cinematics`
- `unreal-serialization-savegames`
- `unreal-state-trees`
- `unreal-testing-debugging`
- `unreal-world-level-streaming`

由 `dsh-extensions/install-skill.sh` 软链进 `~/.dsh/skills/`。
