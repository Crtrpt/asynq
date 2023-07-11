This page explains how to configure `asynq` background processing to suite your needs.

## Weighted Priority

By default, `Server` will create a single queue named "default" to process all your tasks.

If you need to assign a priority to each task, you can create multiple queues with different priority level.

Example:
```go
srv := asynq.NewServer(redis, asynq.Config{
    Concurrency: 10,
    Queues: map[string]int{
        "critical": 6,
        "default":  3,
        "low":      1,
    },
})
```
This will create a `Background` instance with three queues: **critical**, **default**, and **low**. 
The number associated with the queue name is the priority level for the queue.

With this above configuration:
- tasks in **critical** queue will be processed **60%** of the time
- tasks in **default** queue will be processed **30%** of the time
- tasks in **low** queue will be processed **10%** of the time

Now that we have multiple queues with different priority level, we can specify which queue to use when we schedule a task.

Example:
```go
client := asynq.NewClient(redis)
task := asynq.NewTask("send_notification", map[string]interface{}{"user_id": 42})

// Specify a task to use "critical" queue using `asynq.Queue` option.
err := client.Enqueue(task, asynq.Queue("critical"))

// By default, task will be enqueued to "default" queue.
err = client.Enqueue(task)
```

We can inspect queues with `asynq stats` command.

![Output of asynq stats command](https://github.com/hibiken/asynq/blob/master/docs/assets/asynq_stats.gif?raw=true)

You can see the number of tasks in each queue in the "QUEUES" section of the output.

## Strict Priority

If you need to create multiple queues and need to process all tasks in one queue over other queues, you can use `StrictPriority` option.

Example:
```go
srv := asynq.NewServer(redis, asynq.Config{
    Concurrency: 10,
    Queues: map[string]int{
        "critical": 3,
        "default":  2,
        "low":      1,
    },
    StrictPriority: true, // strict mode!
})
```
This will create a `Background` instance with three queues: **critical**, **default**, and **low** with strict priority. In strict priority mode, the queues with higher priority is always processed first, and queues with lower priority is processed only if all the other queues with higher priorities are empty.

So in this example, tasks in **critical** queue is always processed first.
If **critical** queue is empty, then **default** queue is processed.
If both **critical** and **default** queue are empty, then **low** queue is processed.

