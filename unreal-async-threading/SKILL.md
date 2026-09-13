---
name: unreal-async-threading
description: Unreal async tasks, threading, task systems, thread-safety: UE::Tasks, Async(), ParallelFor, futures, locks, game-thread ownership bugs.
author: Sx
version: 1.0.0
keywords:
  - unreal
  - async
  - threading
  - ue tasks
  - parallelfor
  - future
  - lock
---

# Unreal Async Threading

## Tool Usage Rules

When implementing threading code, you MUST:
1. First read `.agents/unreal-project-context.md` using the `Read` tool with proper XML: `<｜｜DSML｜｜invoke name="Read"><｜｜DSML｜｜parameter name="file_path" string="true">.agents/unreal-project-context.md</｜｜DSML｜｜parameter></｜｜DSML｜｜invoke>`
2. Write complete `.h` and `.cpp` files using the `Write` tool — do not output code in chat.
3. For existing files, use `Edit` with exact line ranges.
4. Always validate: `IsInGameThread()` for any UObject access, `std::atomic<bool>` for stop flags, `EQueueMode::Mpsc` for single-consumer queues.

## Trigger Words

Use this skill when the user mentions:
- "UE::Tasks"
- "Async()"
- "ParallelFor"
- "FAsyncTask"
- "thread safety"
- "game thread"
- "lock"

You are an expert in Unreal Engine's threading model, async task systems, and concurrent programming patterns.

## Context Check

Read `.agents/unreal-project-context.md` before proceeding. Engine version matters: `UE::Tasks::Launch` is the modern preferred API (UE 5.0+), while `FAsyncTask` and TaskGraph remain fully supported. Determine: What work needs to be offloaded? Is UObject access required? What latency/throughput tradeoff is acceptable?

## Information Gathering

Ask the user if unclear:
- **Offload type** — CPU-bound computation, I/O wait, or periodic background work?
- **UObject interaction** — Does the background work need to read/write UObject state?
- **Lifetime** — One-shot task, recurring work, or long-lived thread?
- **Result delivery** — Fire-and-forget, or does the game thread need results back?

---

## UE Threading Model

UE runs several named threads plus a scalable worker pool. Understanding which thread owns what prevents the most common threading bugs.

**Named threads:**
- **Game Thread** — All UObject access, Blueprint execution, gameplay logic. Check with `IsInGameThread()`.
- **Render Thread** — Render commands, scene proxy updates. `IsInRenderingThread()`.
- **RHI Thread** — GPU command submission (platform-dependent).
- **Worker Threads** — Unnamed pool threads for task dispatch. Count scales with CPU cores.

**The golden rule:** UObjects are game-thread-only. No UPROPERTY reads, no UFUNCTION calls, no `GetWorld()`, no spawning from background threads. Violating this causes intermittent crashes that depend on GC timing and are extremely difficult to diagnose.

---

## Pattern Selection Guide

Choose the simplest API that fits your needs.

| Pattern | Best For | Lifetime | Result? |
|---------|----------|----------|---------|
| `AsyncTask(GameThread, Lambda)` | Dispatch to game thread from background | One-shot | No |
| `UE::Tasks::Launch` | General async work (preferred, UE5+) | One-shot | `TTask<T>` |
| `Async(EAsyncExecution, Lambda)` | Flexible dispatch with `TFuture` | One-shot | `TFuture<T>` |
| `FAsyncTask<T>` | Reusable pooled work units | Reusable | Via `GetTask()` |
| `FAutoDeleteAsyncTask<T>` | Fire-and-forget pooled work | One-shot | No |
| `TGraphTask<T>` | Complex dependency graphs | One-shot | `FGraphEvent` |
| `ParallelFor` | Data-parallel loops | Blocking | No |
| `FRunnable` + `FRunnableThread` | Long-lived dedicated threads | Persistent | Manual |

---

## FRunnable and FRunnableThread

Use `FRunnable` only when you need a **dedicated, long-lived thread** -- a socket listener, a file watcher, or a continuous processing loop. For one-shot work, prefer `UE::Tasks::Launch` or `FAsyncTask`.

**Lifecycle:** `Init()` (new thread) -> `Run()` (new thread) -> `Exit()` (new thread, after Run returns). `Stop()` is called externally to request shutdown.

**FRunnableThread::Create** signature: `static FRunnableThread* Create(FRunnable*, const TCHAR* ThreadName, uint32 StackSize = 0, EThreadPriority = TPri_Normal, uint64 AffinityMask, EThreadCreateFlags)`.

**Key points:** `Stop()` signals the thread -- it does not block. `Kill(true)` calls `Stop()` then waits for completion. Always `delete` the `FRunnableThread*` after `Kill`. Use `std::atomic<bool> bShouldStop` in `Run()` loop, set it in `Stop()`.

## Complete FRunnable Example: File Watcher

This is the exact pattern for a dedicated file-watching thread. Copy and adapt this.

```cpp
// FileWatcherRunnable.h
#pragma once
#include "HAL/Runnable.h"
#include "HAL/RunnableThread.h"
#include "Containers/Queue.h"
#include "Misc/Paths.h"
#include "Misc/FileHelper.h"
#include "HAL/PlatformProcess.h"
#include <atomic>

class FFileWatcherRunnable : public FRunnable
{
public:
    FFileWatcherRunnable(const FString& InDirectory, const FString& InExtension, TQueue<FString, EQueueMode::Mpsc>& OutQueue)
        : Directory(InDirectory)
        , Extension(InExtension)
        , FileQueue(OutQueue)
        , bStopRequested(false)
    {}

    // FRunnable interface
    virtual bool Init() override { return true; }
    
    virtual uint32 Run() override
    {
        TSet<FString> KnownFiles;
        // Enumerate existing files
        IFileManager::Get().FindFiles(KnownFiles, *Directory, *Extension);
        
        while (!bStopRequested)
        {
            TArray<FString> CurrentFiles;
            IFileManager::Get().FindFiles(CurrentFiles, *Directory, *Extension);
            
            for (const FString& File : CurrentFiles)
            {
                if (!KnownFiles.Contains(File))
                {
                    FileQueue.Enqueue(Directory / File);
                    KnownFiles.Add(File);
                }
            }
            
            // Sleep 1 second between polls
            FPlatformProcess::Sleep(1.0f);
        }
        return 0;
    }
    
    virtual void Stop() override { bStopRequested = true; }
    
    virtual void Exit() override {}

private:
    FString Directory;
    FString Extension;
    TQueue<FString, EQueueMode::Mpsc>& FileQueue;
    std::atomic<bool> bStopRequested;
};
```

**Usage on game thread:**
```cpp
// In some manager class
TQueue<FString, EQueueMode::Mpsc> FileQueue;
FFileWatcherRunnable* Watcher = new FFileWatcherRunnable("/Game/Config", TEXT(".json"), FileQueue);
FRunnableThread* WatcherThread = FRunnableThread::Create(Watcher, TEXT("ConfigWatcher"));

// Tick: consume files on game thread
void AMyActor::Tick(float DeltaTime)
{
    FString FilePath;
    while (FileQueue.Dequeue(FilePath))
    {
        // Safe: we're on game thread now
        ProcessConfigFile(FilePath);
    }
}

// Cleanup
void AMyActor::BeginDestroy()
{
    if (WatcherThread)
    {
        WatcherThread->Kill(true);
        delete WatcherThread;
        WatcherThread = nullptr;
    }
    Super::BeginDestroy();
}
```

---

## FAsyncTask and FAutoDeleteAsyncTask

For **reusable work units** on the engine thread pool (`GThreadPool`). Subclass `FNonAbandonableTask` and implement `DoWork()` + `GetStatId()`.

```cpp
class FMyComputeTask : public FNonAbandonableTask
{
    friend class FAsyncTask<FMyComputeTask>;
    int32 Result = 0;
    TArray<int32> InputData;

    FMyComputeTask(TArray<int32> InData) : InputData(MoveTemp(InData)) {}

    void DoWork()
    {
        for (int32 Val : InputData) { Result += Val; }
    }

    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyComputeTask, STATGROUP_ThreadPoolAsyncTasks);
    }
};
```

**Usage:**

```cpp
// Reusable — you manage lifetime
auto* Task = new FAsyncTask<FMyComputeTask>(MoveTemp(Data));
Task->StartBackgroundTask();          // dispatches to GThreadPool
Task->EnsureCompletion();             // blocks or runs inline if not started
int32 R = Task->GetTask().Result;
delete Task;

// Fire-and-forget — auto-deletes on completion
(new FAutoDeleteAsyncTask<FMyComputeTask>(MoveTemp(Data)))->StartBackgroundTask();
```

`IsWorkDone()` is the non-blocking completion check. `Cancel()` prevents execution if not yet started. `StartSynchronousTask()` runs inline on the calling thread.

---

## TaskGraph

For work with **complex dependency chains**. Each task declares prerequisites; the scheduler handles ordering.

```cpp
class FMyGraphTask
{
public:
    FMyGraphTask(int32 InValue) : Value(InValue) {}

    static ESubsequentsMode::Type GetSubsequentsMode()
    { return ESubsequentsMode::TrackSubsequents; }

    ENamedThreads::Type GetDesiredThread()
    { return ENamedThreads::AnyThread; }

    TStatId GetStatId() const
    { RETURN_QUICK_DECLARE_CYCLE_STAT(FMyGraphTask, STATGROUP_TaskGraphTasks); }

    void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
    { /* work here */ }

private:
    int32 Value;
};
```

**Dispatching with prerequisites:**

```cpp
FGraphEventArray Prerequisites;  // TArray<FGraphEventRef, TInlineAllocator<4>>
Prerequisites.Add(SomePreviousTask->GetCompletionEvent());

TGraphTask<FMyGraphTask>::CreateTask(&Prerequisites)
    .ConstructAndDispatchWhenReady(42);
```

---

## UE::Tasks (UE 5.0+)

Modern, lightweight, and preferred for most async work. No UObject access from within tasks.

```cpp
#include "Tasks/Task.h"

// Fire-and-forget
UE::Tasks::Launch(TEXT("MyTask"), []()
{
    // Background work here
});

// With result
TTask<int32> Task = UE::Tasks::Launch(TEXT("ComputeTask"), []()
{
    return 42;
});

// Await result on game thread (blocks calling thread)
int32 Result = Task.GetResult();
```

**Key APIs:**
- `UE::Tasks::Launch(NAME, Callable, Prerequisites)` — returns `TTask<T>`
- `TTask<T>::GetResult()` — blocks until completion
- `TTask<T>::IsCompleted()` — non-blocking check
- `UE::Tasks::ParallelFor(Count, Callable, ParallelForParams)` — data-parallel

---

## ParallelFor

For **data-parallel loops** — processing large arrays where each element is independent.

```cpp
TArray<int32> Data = {1, 2, 3, 4, 5, 6, 7, 8};
TArray<int32> Results;
Results.SetNum(Data.Num());

// Simple parallel loop
ParallelFor(Data.Num(), [&](int32 Index)
{
    Results[Index] = Data[Index] * 2;
});

// With granularity control (process multiple elements per task)
ParallelFor(Data.Num(), [&](int32 Index)
{
    Results[Index] = Data[Index] * 2;
}, EParallelForFlags::None, 64); // 64 elements per task
```

**Flags:**
- `EParallelForFlags::None` — default
- `EParallelForFlags::ForceSingleThread` — debug/testing
- `EParallelForFlags::BackgroundPriority` — lower priority
- `EParallelForFlags::Unbalanced` — skip work-stealing for known-uniform workloads

---

## Async() and TFuture

Flexible one-shot dispatch with result futures.

```cpp
#include "Async/Async.h"

// Fire on thread pool, get result later
TFuture<int32> Future = Async(EAsyncExecution::ThreadPool, []()
{
    return 42;
});

// Block and get result
int32 Result = Future.Get();

// Fire on game thread from background
AsyncTask(ENamedThreads::GameThread, []()
{
    // Safe UObject access here
});
```

**Execution policies:**
- `EAsyncExecution::Thread` — dedicated OS thread (expensive)
- `EAsyncExecution::ThreadPool` — engine worker pool (preferred)
- `EAsyncExecution::TaskGraph` — TaskGraph main thread (use for short tasks)

---

## Thread Safety Patterns

### Locking with FCriticalSection

```cpp
FCriticalSection Mutex;
TArray<int32> SharedData;

// Write
{
    FScopeLock Lock(&Mutex);
    SharedData.Add(42);
}

// Read
int32 Value;
{
    FScopeLock Lock(&Mutex);
    Value = SharedData.Num();
}
```

### Lock-Free Queue (TQueue)

```cpp
// Producer thread
TQueue<int32, EQueueMode::Mpsc> Queue; // Multi-producer, single-consumer
Queue.Enqueue(42);

// Consumer thread (usually game thread)
int32 Value;
while (Queue.Dequeue(Value))
{
    Process(Value);
}
```

### Atomic Operations

```cpp
#include <atomic>

std::atomic<int32> Counter{0};
Counter.fetch_add(1);          // Thread-safe increment
int32 Current = Counter.load(); // Thread-safe read
```

### FThreadSafeCounter

```cpp
FThreadSafeCounter Counter;
Counter.Increment();           // Thread-safe increment
int32 Current = Counter.GetValue();
```

---

## Common Threading Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| Crash in GC | UObject access off game thread | Wrap in `AsyncTask(ENamedThreads::GameThread, ...)` |
| Deadlock | Lock ordering violation | Use `FScopeLock` consistently, never hold locks across task waits |
| Memory corruption | Shared mutable state without sync | Use `TQueue<EQueueMode::Mpsc>` or `FThreadSafeCounter` |
| Thread leak | `FRunnableThread*` not deleted | Call `Kill(true)` then `delete` in destructor |
| Stale data | File read while writing | Use `IFileManager::Get().FileExists()` + `FFileHelper::LoadFileToString` with retry |

---

## Best Practices

1. **Minimize shared state** — prefer message passing (`TQueue`) over shared memory with locks
2. **Keep tasks short** — long tasks block the thread pool; break into smaller chunks
3. **Use appropriate synchronization** — `FScopeLock` for rare writes, `TQueue` for streaming, atomics for counters
4. **Always check thread** — `IsInGameThread()` before touching UObjects
5. **Prefer UE::Tasks** — it's lighter, safer, and better integrated than raw `FRunnable`
6. **Handle cancellation** — check `IsEngineExitRequested()` or custom stop flags in long-running loops
7. **Profile first** — don't parallelize without measuring; overhead can exceed gains for small workloads