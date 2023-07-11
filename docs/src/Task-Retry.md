This page explains how to configure task retries.

## Default behavior

By default, `asynq` will retry a task up to 25 times. Every time a task is retried it uses an exponential backoff strategy to calculate the retry delay. 
If a task exhausts all of its retry count (default: 25), the task will moved to the **archive** for debugging and inspection purposes and won't be automatically retried (You can still manually run task using CLI or WebUI).

The following properties of task-retry can be customized:
- Max retry count per task
- Time duration to wait (i.e. delay) before a failed task can be retried again
- Whether to consume the retry count for the task
- Whether to skip retry and send the task directly to the archive

The rest of the page describes each of these customizations.

## Customize Task Max Retry

You can specify the maximum number of times a task can be retried using `asynq.MaxRetry` option when enqueueing a task.

Example:
```go
client.Enqueue(task, asynq.MaxRetry(5))
```
This specifies that the `task` should be retried up to five times.

Alternatively, if you want to specify the maximum retry count for some task, you can set it as a default option for the task.

```go
task := asynq.NewTask("feed:import", nil, asynq.MaxRetry(5))
client.Enqueue(task) // MaxRetry is set to 5
```


## Customize Retry Delay

You can specify how to calculate retry delay using `RetryDelayFunc` option in `Config`.

Signature of `RetryDelayFunc`:

```go
// n is the number of times the task has been retried
// e is the error returned by the task handler
// t is the task in question
RetryDelayFunc func(n int, e error, t *asynq.Task) time.Duration
```

Example:
```go
srv := asynq.NewServer(redis, asynq.Config{
    Concurrency: 20,
    RetryDelayFunc: func(n int, e error, t *asynq.Task) time.Duration {
        return 2 * time.Second
    },
})
```

This specifies that all failed task will wait two seconds before being processed again.

The default behavior is exponential backoff, and is defined by `DefaultRetryDelayFunc`.
The example below shows how to customize retry delay for a specific task type:

```go
srv := asynq.NewServer(redis, asynq.Config{
    // Always use 2s delay for "foo" task, other tasks use the default behavior.
    RetryDelayFunc: func(n int, e error, t *asynq.Task) time.Duration {
        if t.Type() == "foo" {
            return 2 * time.Second 
        }
        return asynq.DefaultRetryDelayFunc(n, e, t) 
    },
})
``` 

## Non-Failure error

Sometimes you may want to return an error from the `Handler` and retry the task later, but don't want to consume the retry count. For example, you may want to retry later since the worker doesn't have enough resource to process the task.  
You can optionally provide `IsFailure(error) bool` function to `Config` when you initialize a server. This predicate function determines whether the error returned from the Handler counts as a failure. If the function returns false (i.e. non-failure error), server won't consume the retry-count of the task and simply schedule the task to be retried later.

Example:

```go
var ErrResourceNotAvailable = error.New("no resource is available")

func HandleResourceIntensiveTask(ctx context.Context, task *asynq.Task) error {
    if !IsResourceAvailable() {
        return ErrResourceNotAvailalbe
    }
    // ... logic of handling resource intensive task
}

// ...

srv := asynq.NewServer(redisConnOpt, asynq.Config{
    // ... other config options
    IsFailure: func(err error) bool {
        return err != ErrResourceNotAvailable // If resource is not available, it's a non-failure error
    },
})
```

## Skip Retry

If `Handler.ProcessTask` returns a `SkipRetry` error, the task will be archived regardless of the number of remaining retry count.
The returned error can be `SkipRetry` or an error which wraps the `SkipRetry` error.

```go
func ExampleHandler(ctx context.Context, task *asynq.Task) error {
    // Task handling logic here...
    // If the handler knows that the task does not need a retry, then return SkipRetry
    return fmt.Errorf("<reason for skipping retry>: %w", asynq.SkipRetry)
}
```