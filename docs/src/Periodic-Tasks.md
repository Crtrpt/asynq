## Overview

You can run a `Scheduler` alongside with the `Server` to process tasks periodically. Scheduler enqueues tasks at regular intervals, that are then executed by available worker servers in the cluster.

You have to ensure only a single scheduler is running for a schedule at a time, otherwise you’d end up with duplicate tasks. Using a centralized approach means the schedule doesn’t have to be synchronized, and the service can operate without using locks.

If you need to dynamically add and remove periodic tasks, use `PeriodicTaskManager` instead of using `Scheduler` directly. See [this wiki](https://github.com/hibiken/asynq/wiki/Dynamic-Periodic-Task) page for more details.

## Time Zones

The periodic task schedules uses the UTC time zone by default, but you can change the time zone used using the `SchedulerOpts`.

```go
// Example of using America/Los_Angeles timezone instead of the default UTC timezone.
loc, err := time.LoadLocation("America/Los_Angeles")
if err != nil {
    panic(err)
}
scheduler := asynq.NewScheduler(
    redisConnOpt, 
    &asynq.SchedulerOpts{
        Location: loc,
    },
)
```

## Entries
To enqueue a task periodically you have to register an entry with the scheduler.

```go
scheduler := asynq.NewScheduler(redisConnOpt, nil)

task := asynq.NewTask("example_task", nil)

// You can use cron spec string to specify the schedule. 
entryID, err := scheduler.Register("* * * * *", task)
if err != nil {
    log.Fatal(err)
}
log.Printf("registered an entry: %q\n", entryID)

// You can use "@every <duration>" to specify the interval.
entryID, err = scheduler.Register("@every 30s", task)
if err != nil {
    log.Fatal(err)
}
log.Printf("registered an entry: %q\n", entryID)

// You can also pass options.
entryID, err = scheduler.Register("@every 24h", task, asynq.Queue("myqueue"))
if err != nil {
    log.Fatal(err)
}
log.Printf("registered an entry: %q\n", entryID)
```

## Run the Scheduler

To start the scheduler, call `Run` on the scheduler.

```go
scheduler := asynq.NewScheduler(redisConnOpt, nil)

// ... Register tasks

if err := scheduler.Run(); err != nil {
    log.Fatal(err)
}
```

The call to `Run` will wait for TERM or INT signal (e.g. Ctrl-C keypress).


## Error Handling

You can provide a function to handle an error if the scheduler could not enqueue a task.

```go
func handleEnqueueError(task *asynq.Task, opts []asynq.Option, err error) {
    // your error handling logic
}

scheduler := asynq.NewScheduler(
    redisConnOpt, 
    &asynq.SchedulerOpts{
        EnqueueErrorHandler: handleEnqueueError,
    },
)
```

## Inspection via CLI

The CLI has a subcommand `cron` to inspect scheduler entries.

To see all entries from the currently running scheduler, you can run:

```
asynq cron ls
```

This command will output a list of entries each with its IDs, Schedule Spec, Next enqueue time, Previous enqueue time.

You can also see a history of each entry by running:

```
asynq cron history <entryID>
```


