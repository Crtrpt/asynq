Asynq tasks go through a number of states in their lifetime. This page documents a life of a task, from the task's creation to its deletion.

## Task Lifecycle

When you enqueue a task, `asynq` manages the task internally to make sure that a handler gets invoked with the task at the specified time. In the process, the task can go through different lifecycle states.

Here's the list of different lifecycle states:

- **Scheduled** : task is waiting to be processed in the future (*Only applies to tasks with `ProcessAt` or `ProcessIn` option*).
- **Pending** : task is ready to be processed and will be picked up by a free worker.
- **Active** : task is being processed by a worker (i.e. handler is invoked with the task).
- **Retry** : worker failed to process the task and the task is waiting to be retried in the future.
- **Archived** : task reached its max retry and stored in an archive for manual inspection.
- **Completed**: task was successfully processed and retained until retention TTL expires (*Only applies to tasks with `Retention` option*).

Let's use an example to look at different lifecycle states.

```go
// Task 1 : Scheduled to be processed 24 hours later.
client.Enqueue(task1, asynq.ProcessIn(24*time.Hour))

// Task 2 : Enqueued to be processed immediately.
client.Enqueue(task2)

// Task 3: Enqueued with a Retention option.
client.Enqueue(task3, asynq.Retention(2*time.Hour))
```

In this example, `task1` will stay in the *scheduled* state for the next 24 hours. After 24 hours, it will transition to the *pending* state and then to the *active* state. If the task was processed successfully then the task data is removed from Redis. If the task was *NOT* processed successfully (i.e. handler returned an error OR panicked), then the task will transition to the *retry* state to be retried later.
[After some delay](https://github.com/hibiken/asynq/wiki/Task-Retry#customize-retry-delay), the task will transition to the *pending* state again and then to the *active*. This loop will continues until either the task gets processed successfully OR the task exhausts all of its retry count. In the latter case, the task will transition to the *archived* state.

The only difference between `task2` and `task1` in the example is that `task2` will skip the **scheduled** state and goes directly to the **pending** state. 

`task3` is enqueued with a `Retention` option of 2 hours. This means that after task3 gets processed successfully by a worker, the task will remain in the queue in the **completed** state for 2 hours before it gets deleted from the queue. By default, if a task doesn't have retention option set, the task will be deleted immediately after completion.

The diagram below shows the state transitions. 
```txt
+-------------+            +--------------+          +--------------+           +-------------+
|             |            |              |          |              | Success   |             |
|  Scheduled  |----------->|   Pending    |--------->|    Active    |---------> |  Completed  |
|  (Optional) |            |              |          |              |           |  (Optional) |
+-------------+            +--------------+          +--------------+           +-------------+
                                  ^                       |                            |
                                  |                       |                            | Deletion
                                  |                       | Failed                     |
                                  |                       |                            V
                                  |                       |
                                  |                       |
                           +------+-------+               |        +--------------+
                           |              |               |        |              |
                           |    Retry     |<--------------+------->|   Archived   |
                           |              |                        |              |
                           +--------------+                        +--------------+


```

