This page shows an example of how to configure asynq Server to rate limit task processing.

*Note that this is a per server instance rate limit, and not a global rate limit.*

In this example, we are going to use `golang.org/x/time/rate` package to demonstrate rate limiting.
The key configuration here is `IsFailure` and `RetryDelayFunc` in the config when you initialize your server.
We are going to create a custom error type and type assert the given error in the `IsFailure` and `RetryDelayFunc` functions.

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "math/rand"
    "time"

    "golang.org/x/time/rate"
    "github.com/hibiken/asynq"
)

func main() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: ":6379"},
        asynq.Config{
            Concurrency:    10,
            // If error is due to rate limit, don't count the error as a failure.
            IsFailure:      func(err error) bool { return !IsRateLimitError(err) },
            RetryDelayFunc: retryDelay,
        },
    )

    if err := srv.Run(asynq.HandlerFunc(handler)); err != nil {
        log.Fatal(err)
    }
}

type RateLimitError struct {
    RetryIn time.Duration
}

func (e *RateLimitError) Error() string {
    return fmt.Sprintf("rate limited (retry in  %v)", e.RetryIn)
}

func IsRateLimitError(err error) bool {
    _, ok := err.(*RateLimitError)
    return ok
}

func retryDelay(n int, err error, task *asynq.Task) time.Duration {
    var ratelimitErr *RateLimitError
    if errors.As(err, &ratelimitErr) {
        return ratelimitErr.RetryIn
    }
    return asynq.DefaultRetryDelayFunc(n, err, task)
}

// Rate is 10 events/sec and permits burst of at most 30 events.
var limiter = rate.NewLimiter(10, 30)

func handler(ctx context.Context, task *asynq.Task) error {
    if !limiter.Allow() {
        return &RateLimitError{
            RetryIn: time.Duration(rand.Intn(10)) * time.Second,
        }
    }
    log.Printf("[*] processing %s", task.Payload())
    return nil
}
```