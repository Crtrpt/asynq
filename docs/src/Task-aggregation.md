This page describes Asynq's task aggregation feature.

## Overview

Task aggregation allows you to enqueue multiple tasks successively, and have them passed to the `Handler` together rather than individually.
The feature allows you to batch multiple successive operations into one, in order to save on costs, optimize caching, or batch notifications, for example.

## How it works

In order to use the task aggregation feature, you need to enqueue the tasks in the same queue with the common **group name**.
Tasks enqueued with the same `(queue, group)` pairs are aggregated into one task by **`GroupAggregator`** that you provide and the aggregated task will be passed to the handler.

When creating an aggregated task, Asynq server will wait for more tasks until a configurable **grace period**. The grace period is renewed whenever you enqueue a new task with the same `(queue, group)`.

The grace period has configurable upper bound: you can set a **maximum aggregation delay**, after which Asynq server will aggregate the tasks regardless of the remaining grace period.

You can also set a **maximum number of tasks** that can be aggregated together. If that number is reached, Asynq server will aggregate the tasks immediately.

**Note**: Scheduling and aggregation of tasks are conflicting features and scheduling takes silent priority over aggregation.

## Quick Example

On the client side, use `Queue` and `Group` option to enqueue the task in the same group.
```go
// Enqueue three tasks to the same group.
client.Enqueue(task1, asynq.Queue("notifications"), asynq.Group("user1:email"))
client.Enqueue(task2, asynq.Queue("notifications"), asynq.Group("user1:email"))
client.Enqueue(task3, asynq.Queue("notifications"), asynq.Group("user1:email"))
```

On the server side, provide `GroupAggregator` to enable task aggregation feature.
You can optionally configure `GroupGracePeriod`, `GroupMaxDelay`, and `GroupMaxSize` to customize the aggregation policy.
```go
// This function is used to aggregate multiple tasks into one.
func aggregate(group string, tasks []*asynq.Task) *asynq.Task {
    // ... Your logic to aggregate the given tasks and return the aggregated task.
    // ... Use NewTask(typename, payload, opts...) to create a new task and set options if needed.
    // ... (Note) Queue option will be ignored and the aggregated task will always be enqueued to the same queue the group belonged.
}

srv := asynq.NewServer(
           redisConnOpt,
           asynq.Config{
               GroupAggregator:  asynq.GroupAggregatorFunc(aggregate),
               GroupMaxDelay:    10 * time.Minute,
               GroupGracePeriod: 2 * time.Minute,
               GroupMaxSize:     20,
               Queues: map[string]int{"notifications": 1},
           },
       )
```

## Tutorial
In this section, we provide a simple programs to see the aggregation feature in action.

First, create a client program with the following:

```go
// client.go
package main

import (
        "flag"
        "log"

        "github.com/hibiken/asynq"
)

var (
       flagRedisAddr = flag.String("redis-addr", "localhost:6379", "Redis server address")
       flagMessage = flag.String("message", "hello", "Message to print when task gets processed")
)

func main() {
        flag.Parse()

        c := asynq.NewClient(asynq.RedisClientOpt{Addr: *flagRedisAddr})
        defer c.Close()

        task := asynq.NewTask("aggregation-tutorial", []byte(*flagMessage))
        info, err := c.Enqueue(task, asynq.Queue("tutorial"), asynq.Group("example-group"))
        if err != nil {
                log.Fatalf("Failed to enqueue task: %v", err)
        }
        log.Printf("Successfully enqueued task: %s", info.ID)
}
```

You can run this program a few times:

```sh
$ go build -o client client.go 
$ ./client --redis-addr=<REDIS_SERVER_ADDR>
$ ./client --message=hi --redis-addr=<REDIS_SERVER_ADDR>
$ ./client --message=bye --redis-addr=<REDIS_SERVER_ADDR>
```

Now if you inspect the queue via CLI or Web UI, you can see that you have **aggregating** tasks in the queue.

Next, create a server program with the following:

```go
// server.go
package main

import (
        "context"
        "flag"
        "log"
        "strings"
        "time"

        "github.com/hibiken/asynq"
)

var (
        flagRedisAddr = flag.String("redis-addr", "localhost:6379", "Redis server address")
        flagGroupGracePeriod = flag.Duration("grace-period", 10*time.Second, "Group grace period")
        flagGroupMaxDelay = flag.Duration("max-delay", 30*time.Second, "Group max delay")
        flagGroupMaxSize = flag.Int("max-size", 20, "Group max size")
)

// Simple aggregation function.
// Combines all tasks messages, each message on a separate line.
func aggregate(group string, tasks []*asynq.Task) *asynq.Task {
        log.Printf("Aggregating %d tasks from group %q", len(tasks), group)
        var b strings.Builder
        for _, t := range tasks {
                b.Write(t.Payload())
                b.WriteString("\n")
        }
        return asynq.NewTask("aggregated-task", []byte(b.String()))
}

func handleAggregatedTask(ctx context.Context, task *asynq.Task) error {
        log.Print("Handler received aggregated task")
        log.Printf("aggregated messags: %s", task.Payload())
        return nil
}

func main() {
        flag.Parse()

        srv := asynq.NewServer(
                asynq.RedisClientOpt{Addr: *flagRedisAddr},
                asynq.Config{
                        Queues:           map[string]int{"tutorial": 1},
                        GroupAggregator:  asynq.GroupAggregatorFunc(aggregate),
                        GroupGracePeriod: *flagGroupGracePeriod,
                        GroupMaxDelay:    *flagGroupMaxDelay,
                        GroupMaxSize:     *flagGroupMaxSize,
                },
        )

        mux := asynq.NewServeMux()
        mux.HandleFunc("aggregated-task", handleAggregatedTask)

        if err := srv.Run(mux); err != nil {
                log.Fatalf("Failed to start the server: %v", err)
        }
}
```

You can run this program and observe the output:

```sh
$ go build -o server server.go
$ ./server --redis-addr=<REDIS_SERVER_ADDR>
```

You should be able to see in the output that the server has aggregated the tasks in the group and handler processed the aggregated task.
Feel free to play around with `--grace-period`, `--max-delay` and `--max-size` flags in the above program to see how it affects the aggregation policy.