This guide walks you through how to upgrade from Asynq v0.17 to v0.18.
There were a few breaking changes introduced in v0.18 and the following steps should be followed to upgrade from v0.17.
There are two sections to this guide:
- Migrate existing tasks and queues in Redis
- Update the application code to be v0.18 compatible

## Migrate Tasks and Queues in Redis

### Step 1 - Back up your data in Redis
First, since we will be running destructive commands against existing data, please make sure to create a backup using RDB snapshot file (See  [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-back-up-and-restore-your-redis-data-on-ubuntu-14-04) for how to back up your data).

### Step 2 - Install Asynq CLI v0.18
Once you have created a backup, install the new Asynq CLI by running (`go get -u github.com/hibiken/asynq/tools/asynq`).  
Make sure you installed the `0.18.x` version of the CLI by running `asynq version`.  

### Step 3 - Run the migrate command
Once you confirmed your CLI is v0.18.x, then run the `asynq migrate` command against your redis db (by default it connects to `localhost:6379` and `db=0`, but you can pass these values via flags: Run `asynq --help` for details).

**Note**: If you encounter any errors in `asynq migrate`, please restore your data using the backup file and report the error at https://github.com/hibiken/asynq/issues. 


## Update Application Code

Once the existing data has been migrated, the next step is to update your application code.

**Note:** All API changes are documented in the CHANGELOG.

### Step 4 - Update Task Payload
Task payload data type has changed from `map[string]interface{}` to `[]byte` to allow arbitrary bytes. Since all previously created tasks' payloads were encoded in JSON, you need to unmarshal your task payloads with the `json` package for the existing tasks.

Example:
```go
// In v0.17 or earlier version.
func myHandler(ctx context.Context, task *asynq.Task) error {
    // Error check skipped for brevity
    foo, err := task.Payload.GetString("foo")
    baz, err := task.Payload.GetInt("baz")

    // ...
}

// In v0.18.
func myHandler(ctx context.Context, task *asynq.Task) error {
    // Error check skipped for brevity
    var payload map[string]interface{}
    if err := json.Unmarshal(task.Payload(), &payload); err != nil {
        return err
    }
    foo, ok := payload["foo"]
    baz, ok := payload["baz")

   // ...
}
```

Note that this only applies to tasks enqueued using previous version of Asynq. For new tasks, I recommend that you define a type for each payload instead of using `map[string]interface{}`.

Example:
```go
type MyTaskPayload struct {
    Foo string
    Baz string
}

data, err := json.Marshal(MyTaskPayload{Foo: "a", Baz: "b"})
task := asynq.NewTask("my_task", data)
```


### Step 5 - Update Server and Scheduler
**Important Note**: The semantics of `Stop` has changed in `Server` so please make sure to update your code if you called `Stop` with the previous versions.  

A few methods on `Server` and `Scheduler` have changed.

Previously called `Server.Quiet` is now `Server.Stop`.

Previously called `Server.Stop` is now `Server.Shutdown`.

Previously called `Scheduler.Stop` is now `Scheduler.Shutdown`.