# AsyncMixin Async Loading

## Architecture Overview

AsyncMixin is a lightweight, non-UObject asynchronous resource loading tool that allows chaining async loading sequences in a fluent manner. It supports preloading PrimaryAssets, custom conditions, and events.

Core file: AsyncMixin plugin source

## FAsyncMixin

```cpp
class FAsyncMixin : public FNoncopyable
{
public:
    // Load a single soft reference
    template <typename T = UObject>
    void AsyncLoad(TSoftClassPtr<T> SoftClass, TFunction<void()>&& Callback);

    // Load multiple resources
    void AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths,
                   const FSimpleDelegate& Callback = FSimpleDelegate());

    // Preload PrimaryAsset
    void AsyncPreloadPrimaryAssetsAndBundles(
        const TArray<FPrimaryAssetId>& AssetIds,
        const TArray<FName>& LoadBundles,
        const FSimpleDelegate& Callback = FSimpleDelegate());

    // Custom condition
    void AsyncCondition(TSharedRef<FAsyncCondition> Condition,
                        const FSimpleDelegate& Callback = FSimpleDelegate());

    // Placeholder event
    void AsyncEvent(const FSimpleDelegate& Callback);

    // Lifecycle
    void StartAsyncLoading();
    void CancelAsyncLoading();
    bool IsAsyncLoadingInProgress() const;

    // Virtual callbacks
    virtual void OnStartedLoading() {}
    virtual void OnFinishedLoading() {}
};
```

### Design Points

- `FNoncopyable` — Non-copyable, each Mixin holds uniquely
- Zero memory overhead — State is only allocated after calling `AsyncLoad`
- Uses a static `TMap<FAsyncMixin*, TSharedRef<FLoadingState>>` to manage all loading states

## Loading State (FLoadingState)

```cpp
class FLoadingState : public TSharedFromThis<FLoadingState>
{
    TArray<TUniquePtr<FAsyncStep>> AsyncSteps;
    int32 CurrentAsyncStep = 0;

    void Start();
    void TryCompleteAsyncLoading();
    void CompleteAsyncLoading();
    void RequestDestroyThisMemory();  // ticker deferred destruction
};
```

- Maintains a step queue
- Executes serially; automatically proceeds to the next step after one completes
- Calls `OnFinishedLoading` after all steps complete → deferred memory cleanup via ticker

## Loading Step (FAsyncStep)

```cpp
class FAsyncStep
{
    FSimpleDelegate UserCallback;
    TSharedPtr<FStreamableHandle> StreamingHandle;
    TSharedPtr<FAsyncCondition> Condition;

    bool IsComplete() const;
    void ExecuteUserCallback();
    bool BindCompleteDelegate(const FSimpleDelegate&);  // Re-entrancy guard
};
```

- Each `AsyncLoad` / `AsyncCondition` call generates a step
- Supports two completion modes: `StreamingHandle` (resource loading complete) and `Condition` (custom condition)
- `BindCompleteDelegate` prevents duplicate registration of completion callbacks

## FAsyncCondition

```cpp
class FAsyncCondition : public TSharedFromThis<FAsyncCondition>
{
    FTSTicker::FDelegateHandle RepeatHandle;
    FAsyncConditionDelegate UserCondition;
    FSimpleDelegate CompletionDelegate;

    bool TryToContinue(float);  // ticker polling
};
```

- Polls completion state via ticker (every 0.16 seconds)
- Returns `TryAgain` to continue polling, returns `Complete` to trigger next step

```cpp
enum class EAsyncConditionResult : uint8
{
    TryAgain,
    Complete
};
```

## Async Sequence Execution Flow

1. Call `AsyncLoad` / `AsyncCondition` / `AsyncEvent` to enqueue steps
2. Call `StartAsyncLoading()` to start loading
3. `FLoadingState` begins at `CurrentAsyncStep = 0`
4. Current step's `IsComplete()` checks completion status
5. On completion, executes `UserCallback`, `CurrentAsyncStep++`
6. Proceeds to next step
7. After all steps complete, calls `OnFinishedLoading`
8. `RequestDestroyThisMemory` schedules ticker-deferred destruction of loading state

## FAsyncScope

```cpp
class FAsyncScope : public FAsyncMixin {};
```

- Self-contained, does not require inheritance
- Suitable for direct use in non-Mixin scenarios
