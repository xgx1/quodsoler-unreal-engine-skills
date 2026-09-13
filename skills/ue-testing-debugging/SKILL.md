---
name: unreal-testing-debugging
description: 'Unreal tests: UE_LOG, logging, log categories, assertion, check, ensure, verify, DrawDebug, debug draw, console command, profiling, Unreal Insights.'
author: Sx
version: 1.0.0
keywords:
  - unreal
  - testing
  - debugging
---

# Unreal Testing Debugging

## Trigger Words

Use this skill when the user mentions:
- "testing"
- "debugging"
- "testing debugging"
- "stat group"
- "cycle stat"
- "CSV profiler"
- "profiling"
- "SCOPE_CYCLE_COUNTER"

## Context

Read `.agents/unreal-project-context.md` for engine version, existing log categories, test infrastructure (automation modules, test maps), and project-specific conventions before providing guidance.

## Before You Start

**Direct response rule**: When the user asks for a specific code implementation (e.g., "write a test", "create a function", "add debug drawing"), provide the complete code immediately. Do NOT check project context, search for files, or ask clarifying questions.

**Exception for asset paths**: If the request involves loading specific assets (e.g., weapon data assets, blueprint classes, data tables), you MUST first read `.agents/unreal-project-context.md` to discover the correct asset paths, naming conventions, and any existing test data setup. Do not guess asset paths or class names.

**Exception**: If the implementation requires referencing project-specific classes, enums, or systems (e.g., enemy actor class, health component, combat manager, interaction interfaces), you MUST first read `.agents/unreal-project-context.md` and any relevant source files to get the correct class names and APIs. Then provide the complete code. Do NOT start writing code until you have gathered this information.

**Critical rule**: If the user's request involves creating new classes, components, systems, OR loading/validating project assets (weapon data assets, blueprints, data tables, etc.), you MUST first read `.agents/unreal-project-context.md`. Do not write any code until you have verified the project's existing class hierarchy, asset paths, naming conventions, and module structure. Guessing asset paths or class names will produce broken code.

**Profiling exception**: When the user requests profiling setup (stat groups, CSV profiler, Unreal Insights), ALWAYS check `.agents/unreal-project-context.md` for existing stat groups, profiling headers, and module startup code. Provide the complete implementation including header guards, proper includes, and module registration code. Never omit the closing quote in include statements or leave code snippets incomplete.

If the request is ambiguous or missing critical details, ask which area the user needs help with:
- **Automation tests** — unit/integration tests using IMPLEMENT_SIMPLE_AUTOMATION_TEST
- **Functional tests** — actor-based AFunctionalTest in maps
- **Logging** — UE_LOG, custom categories, verbosity filtering
- **Assertions** — check, ensure, verify and when to use each
- **Debug drawing** — DrawDebug helpers for runtime visualization
- **Console commands** — UFUNCTION(Exec), FAutoConsoleCommand, CVars
- **Profiling** — Unreal Insights, stat commands, SCOPE_CYCLE_COUNTER
- **Asset validation** — Complex automation tests that load and validate project assets (weapon data, blueprints, etc.) require project context for correct paths and class names
- **Context gathering** — If the request involves creating new systems or components, I should first check project context for existing patterns and conventions before writing code.

---
## Automation Framework

Automation tests live in a dedicated module (e.g., `MyGameTests`) that depends on `"AutomationController"`. Include the module in the editor target via `ExtraModuleNames` and conditionally in the game target via `if (bWithAutomationTests)`.

### Simple Tests

```cpp
// Source/MyGameTests/Private/MyFeature.spec.cpp
#include "Misc/AutomationTest.h"
// Must specify one application context flag AND exactly one filter flag
IMPLEMENT_SIMPLE_AUTOMATION_TEST(FMyInventoryTest, "MyGame.Inventory.AddItem",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

bool FMyInventoryTest::RunTest(const FString& Parameters)
{
    UInventoryComponent* Inv = NewObject<UInventoryComponent>();
    Inv->AddItem(FName("Sword"), 1);
    TestEqual(TEXT("Item count after add"), Inv->GetItemCount(FName("Sword")), 1);
    TestTrue(TEXT("Has sword"), Inv->HasItem(FName("Sword")));
    TestFalse(TEXT("No axe"),   Inv->HasItem(FName("Axe")));
    TestNotNull(TEXT("Inv valid"), Inv);
    return true;
}
```

> **Important**: Always use exactly one application context flag (`EditorContext`, `ClientContext`, `ServerContext`, or `CommandletContext`) AND exactly one filter flag (`ProductFilter`, `PerfFilter`, `StressFilter`, or `NegativeFilter`). Missing or incorrect flags will cause tests to not appear in the automation window. Common valid combinations include:
> - `EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter` (editor tests)
> - `EAutomationTestFlags::ClientContext | EAutomationTestFlags::ProductFilter` (runtime tests)

### Complex / Parameterized Tests

`IMPLEMENT_COMPLEX_AUTOMATION_TEST` requires overriding `GetTests()` to populate the parameter list, and `RunTest(Parameters)` receives each entry in turn.

> **GetTests Pattern**: When implementing `IMPLEMENT_COMPLEX_AUTOMATION_TEST`, you MUST override `GetTests` to provide parameter pairs. Each pair consists of a beautified name (display name) and a test command (the parameter string passed to `RunTest`). The number of pairs must match exactly between `OutBeautifiedNames` and `OutTestCommands`.

```cpp
IMPLEMENT_COMPLEX_AUTOMATION_TEST(FMyAssetLoadTest, "MyGame.Assets.LoadByPath",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

void FMyAssetLoadTest::GetTests(TArray<FString>& OutBeautifiedNames,
                                 TArray<FString>& OutTestCommands) const
{
    OutBeautifiedNames.Add(TEXT("Sword"));  OutTestCommands.Add(TEXT("/Game/Items/BP_Sword"));
    OutBeautifiedNames.Add(TEXT("Shield")); OutTestCommands.Add(TEXT("/Game/Items/BP_Shield"));
}

bool FMyAssetLoadTest::RunTest(const FString& Parameters)
{
    UObject* Asset = StaticLoadObject(UObject::StaticClass
```
## 诊断工作流（Debugging Workflow）

Every debug task must define:
- reproducible scenario and expected vs observed behavior
- data capture set (logs, runtime state, asset/config snapshot)
- first bad transition candidate in the execution pipeline
- hypothesis list ranked by probability and verification cost
- fix validation and regression scope

If any item is missing, diagnosis output is incomplete.

### 1) Reproduce and Freeze Context
- Build a minimal deterministic repro with exact steps and preconditions.
- Capture map, actor setup, input sequence, and runtime mode.
- Define expected result and observed deviation.

### 2) Capture Signals
- Filter logs by relevant categories and timestamps around the failure window.
- Add targeted debug markers (`UE_LOG`, on-screen debug, or Blueprint print) if needed.
- Capture relevant state snapshots at stage boundaries.

### 3) Validate Data and Assets
- Verify key assets/classes/config entries exist and resolve correctly.
- Check dependencies/referencers for missing or mismatched assets (`IAssetRegistry::GetDependencies` / `GetReferencers`).
- Confirm runtime-loaded data matches the expected environment.

### 4) Locate First Bad Transition
- Walk the pipeline step-by-step and identify the earliest divergence point.
- Separate root cause from downstream noise symptoms.
- Prioritize the smallest fixable cause with the highest confidence.

### 5) Hypothesis and Verification
- Rank hypotheses by probability and verification cost.
- Run one focused test per hypothesis to avoid cross-contamination.
- Keep rejected hypotheses documented with evidence.

### 6) Fix and Regression Validation
- Apply a minimal fix and rerun the same repro scenario.
- Validate no regression on adjacent systems/paths.
- Output fix summary with confidence and residual risk.

## 调试纪律（Debugging Discipline）

- Avoid broad refactors during diagnosis.
- Keep repro deterministic and documented.
- Prefer observable checks over assumptions.
- Separate root cause from secondary noise.
- Do not mix instrumentation changes with functional fixes in one step.
- **证据先于修改（Evidence before modification）**: preserve failing evidence before introducing mitigation changes.
- **最小修复 + 回归矩阵（Minimal fix + regression matrix）**: keep the fix minimal and extend the regression matrix around impacted paths.
- Always keep a minimal repro artifact (steps, map, config) with the diagnosis.
- Always include first-failure evidence, not only final symptom logs.
- Always provide a verification checklist for the proposed fix.
- Always state residual risk when confidence is below high.

## 故障处理（Failure Handling）

- Symptom: cannot reproduce the issue consistently.
  - Locate: missing preconditions, race windows, or nondeterministic setup.
  - Fix: tighten repro setup and add targeted instrumentation checkpoints.
- Symptom: logs contain too much unrelated noise.
  - Locate: broad log categories and missing temporal scoping.
  - Fix: narrow category filters and focus around failure timestamps.
- Symptom: multiple plausible causes remain.
  - Locate: shared downstream symptom without first-failure isolation.
  - Fix: split into independent hypotheses and run low-cost discriminating tests.
- Symptom: issue disappears after adding debug output.
  - Locate: timing-sensitive/race-sensitive behavior.
  - Fix: use low-overhead markers and repeat with controlled timing.
- Symptom: fix resolves one path but breaks another.
  - Locate: hidden coupling between systems or config layers.
  - Fix: keep the fix minimal and extend the regression matrix around impacted paths.
- Symptom: runtime mismatch only happens on packaged builds.
  - Locate: build config/cook differences versus editor run.
  - Fix: compare packaged and editor config/assets and validate load order.

## 可复现性能基线（Reproducible Performance Baseline）

- Freeze test map, camera path, scalability, and net mode.
- Capture `stat unit` and `stat gpu` under the same scenario repeatedly.
- Use median or percentile metrics, not single-frame spikes.
- Always report metric units (ms, fps, memory) and capture duration.
- Warm up the run before measuring; compare median/percentile values when data is noisy.

## 热点隔离（Hotspot Isolation）

- Identify top-cost gameplay/render systems in the capture window.
- Correlate high-cost actors/assets with scene context.
- Add scoped CPU markers for custom systems when attribution is unclear:
  - `TRACE_CPUPROFILER_EVENT_SCOPE(...)` — Unreal Insights scope marker
  - `DECLARE_CYCLE_STAT(...)` + `SCOPE_CYCLE_COUNTER(...)` — stat group counters
- Keep the profiling scenario reproducible (map, camera path, net mode).
- Do not claim optimization wins without measurable before/after data.

## 打包 vs PIE 差异（Packaged vs PIE Differences）

- Distinguish editor overhead from packaged runtime behavior.
- Symptom: packaged build perf is worse than the PIE baseline.
  - Locate: packaged-only config differences and runtime content path.
  - Fix: compare packaged config to baseline and align scalability/device settings.
- Symptom: memory usage regresses near content-heavy scenes.
  - Locate: high-memory assets and streaming policy behavior.
  - Fix: reduce resident asset pressure and adjust streaming/cook strategy.

## 发布就绪 go/no-go 检查清单（Release Readiness Go/No-Go Checklist）

Every performance/packaging readiness task must define:
- target platform + build config + test map
- reproducible capture scenario (camera path, duration, net mode)
- baseline metrics and acceptance thresholds
- packaging configuration set and dependency scan scope
- go/no-go output with explicit blockers

If any item is missing, readiness evaluation is incomplete.

Decision rules:
- Approve only when metrics and packaging checks meet thresholds.
- Report remaining risks and recommended mitigation actions.
- Lock the scenario and settings used for sign-off evidence.
- Keep packaging checks deterministic; avoid ad-hoc config toggles during sign-off.
- Re-run readiness checks after any packaging setting change.
- Escalate when the bottleneck requires engine-level profiling or renderer changes.
