### Welcome to a Tour of Asynq!

![Task Queue Diagram](https://github.com/hibiken/asynq/raw/master/docs/assets/overview.png)

In this tutorial, we are going to write two programs, `client` and `workers`.

- `client.go` will create and schedule tasks to be processed asynchronously by the background workers.
- `workers.go` will start multiple concurrent workers to process the tasks created by the client.

**This guide assumes that you are running a Redis server at `localhost:6379`**.
Before we start, make sure you have Redis installed and running.

Let's start by creating our two main files.

```sh
mkdir quickstart && cd quickstart
go mod init asynq-quickstart
mkdir client workers
touch client/client.go workers/workers.go
```

And install `asynq` package.

```sh
go get -u github.com/hibiken/asynq
```

Before we start writing code, let's review a few core types that we'll use in both of our programs.

### Redis Connection Option
Asynq uses Redis as a message broker.  
Both `client.go` and `workers.go` need to connect to Redis to write to and read from it. 
We are going to use `RedisClientOpt` to specify the connection to a Redis server running locally.

```go
redisConnOpt := asynq.RedisClientOpt{
    Addr: "localhost:6379",
    // Omit if no password is required
    Password: "mypassword",
    // Use a dedicated db number for asynq.
    // By default, Redis offers 16 databases (0..15)
    DB: 0,
}
```

### Tasks

In `asynq`, a unit of work is encapsulated in a type called `Task`,
which conceptually has two fields: `Type` and `Payload`.

```go
// Type is a string value that indicates the type of the task. 
func (t *Task) Type() string

// Payload is the data needed for task execution.
func (t *Task) Payload() []byte
```

Now that we've taken a look at the core types, let's start writing our programs.

### Client Program

In `client.go`, we are going to create a few tasks and enqueue them using `asynq.Client`. 

To create a task, use `NewTask` function and pass type and payload for the task.

The [`Enqueue`](https://godoc.org/github.com/hibiken/asynq#Client.Enqueue) method takes a task and any number of options.  
Use [`ProcessIn`](https://godoc.org/github.com/hibiken/asynq#ProcessIn) or [`ProcessAt`](https://godoc.org/github.com/hibiken/asynq#ProcessAt) option to schedule tasks to be processed in the future.

```go
// Task payload for any email related tasks.
type EmailTaskPayload struct {
    // ID for the email recipient.
    UserID int
}

// client.go
func main() {
    client := asynq.NewClient(asynq.RedisClientOpt{Addr: "localhost:6379"})

    // Create a task with typename and payload.
    payload, err := json.Marshal(EmailTaskPayload{UserID: 42})
    if err != nil {
        log.Fatal(err)
    }
    t1 := asynq.NewTask("email:welcome", payload)

    t2 := asynq.NewTask("email:reminder", payload)

    // Process the task immediately.
    info, err := client.Enqueue(t1)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf(" [*] Successfully enqueued task: %+v", info)

    // Process the task 24 hours later.
    info, err = client.Enqueue(t2, asynq.ProcessIn(24*time.Hour))
    if err != nil {
        log.Fatal(err)
    }
    log.Printf(" [*] Successfully enqueued task: %+v", info)
}
```

That's all we need for the client program. 

### Workers Program

In `workers.go`, we'll create a `asynq.Server` instance to start the workers.

`NewServer` function takes `RedisConnOpt` and `Config`.

`Config` is used to tune the server's task processing behavior.  
You can take a look at the documentation on [`Config`](https://pkg.go.dev/github.com/hibiken/asynq#Config) to see all the available config options.

To keep it simple, we are only going to specify the concurrency in this example.

```go
// workers.go
func main() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{Concurrency: 10},
    )

    // NOTE: We'll cover what this `handler` is in the section below.
    if err := srv.Run(handler); err != nil {
        log.Fatal(err)
    }
}
```

The argument to `(*Server).Run` is an interface `asynq.Handler` which has one method `ProcessTask`.

```go
type Handler interface {
    // ProcessTask should return nil if the task was processed successfully.
    // If ProcessTask returns a non-nil error or panics, the task will be retried again later.
    ProcessTask(context.Context, *Task) error
}
```

The simplest way to implement a handler is to define a function with the same signature and use `asynq.HandlerFunc` adapter type when passing it to `Run`.

```go
func handler(ctx context.Context, t *asynq.Task) error {
    switch t.Type() {
    case "email:welcome":
        var p EmailTaskPayload
        if err := json.Unmarshal(t.Payload(), &p); err != nil {
            return err
        }
        log.Printf(" [*] Send Welcome Email to User %d", p.UserID)

    case "email:reminder":
        var p EmailTaskPayload
        if err := json.Unmarshal(t.Payload(), &p); err != nil {
            return err
        }
        log.Printf(" [*] Send Reminder Email to User %d", p.UserID)

    default:
        return fmt.Errorf("unexpected task type: %s", t.Type())
    }
    return nil
}

func main() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{Concurrency: 10},
    )

    // Use asynq.HandlerFunc adapter for a handler function
    if err := srv.Run(asynq.HandlerFunc(handler)); err != nil {
        log.Fatal(err)
    }
}
```

We could keep adding switch cases to this handler function, but in a realistic application, it's convenient to define the logic for each case in a separate function.

To refactor our code, let's use `ServeMux` to create our handler.
Just like the `ServeMux` from `"net/http"` package, you register a handler by calling `Handle` or `HandleFunc`. `ServeMux` satisfies the `Handler` interface, so that you can pass it to `(*Server).Run`. 

```go
// workers.go
func main() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{Concurrency: 10},
    )

    mux := asynq.NewServeMux()
    mux.HandleFunc("email:welcome", sendWelcomeEmail)
    mux.HandleFunc("email:reminder", sendReminderEmail)

    if err := srv.Run(mux); err != nil {
        log.Fatal(err)
    }
}

func sendWelcomeEmail(ctx context.Context, t *asynq.Task) error {
    var p EmailTaskPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Send Welcome Email to User %d", p.UserID)
    return nil
}

func sendReminderEmail(ctx context.Context, t *asynq.Task) error {
    var p EmailTaskPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Send Reminder Email to User %d", p.UserID)
    return nil
}
```
Now that we've extracted functions to handle each task type, the code looks a bit more organized.  
However, the code is a bit too implicit, we have these string values for task types and payload types that should be encapsulated in a cohesive package. Let's refactor our code by writing a package that encapsulates task creations and handling. We'll simply create a package called `task`.

```sh
mkdir task && touch task/task.go
```

```go
package task

import (
    "context"
    "fmt"
   
    "github.com/hibiken/asynq"
)

// A list of task types.
const (
    TypeWelcomeEmail  = "email:welcome"
    TypeReminderEmail = "email:reminder"
)

// Task payload for any email related tasks.
type emailTaskPayload struct {
    // ID for the email recipient.
    UserID int
}

func NewWelcomeEmailTask(id int) (*asynq.Task, error) {
    payload, err := json.Marshal(emailTaskPayload{UserID: id})
    if err != nil {
        return nil, err
    }
    return asynq.NewTask(TypeWelcomeEmail, payload), nil
}

func NewReminderEmailTask(id int) (*asynq.Task, error) {
    payload, err := json.Marshal(emailTaskPayload{UserID: id})
    if err != nil {
        return nil, err
    }
    return asynq.NewTask(TypeReminderEmail, payload), nil
}

func HandleWelcomeEmailTask(ctx context.Context, t *asynq.Task) error {
    var p emailTaskPayload  
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Send Welcome Email to User %d", p.UserID)
    return nil
}

func HandleReminderEmailTask(ctx context.Context, t *asynq.Task) error {
    var p emailTaskPayload  
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Send Reminder Email to User %d", p.UserID)
    return nil
}
```

And now we can import this package in both `client.go` and `workers.go`.

```go
// client.go
func main() {
    client := asynq.NewClient(asynq.RedisClientOpt{Addr: "localhost:6379"})

    t1, err := task.NewWelcomeEmailTask(42)
    if err != nil {
        log.Fatal(err)
    }

    t2, err := task.NewReminderEmailTask(42)
    if err != nil {
        log.Fatal(err)
    }

    // Process the task immediately.
    info, err := client.Enqueue(t1)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf(" [*] Successfully enqueued task: %+v", info)

    // Process the task 24 hours later.
    info, err = client.Enqueue(t2, asynq.ProcessIn(24*time.Hour))
    if err != nil {
        log.Fatal(err)
    }
    log.Printf(" [*] Successfully enqueued task: %+v", info)
}
```

```go
// workers.go
func main() {
    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: "localhost:6379"},
        asynq.Config{Concurrency: 10},
    )

    mux := asynq.NewServeMux()
    mux.HandleFunc(task.TypeWelcomeEmail, task.HandleWelcomeEmailTask)
    mux.HandleFunc(task.TypeReminderEmail, task.HandleReminderEmailTask)

    if err := srv.Run(mux); err != nil {
        log.Fatal(err)
    }
}
```

And now the code looks much nicer!

### Running the Programs

Now that we have both `client` and `workers`, we can run both programs.
Let's run the `client` program to create and schedule tasks.

```sh
go run client/client.go
```

This will create two tasks: One that should be processed immediately and another to be processed 24 hours later.

Let's use `asynq` CLI to inspect the tasks.

```sh
asynq dash
```

You should be able to see that there's one task in **Enqueued** state and another in **Scheduled** state.

**Note**: To learn more about the meaning of each state, check out [Life of a Task](https://github.com/hibiken/asynq/wiki/Life-of-a-Task).

And finally, let's start the `workers` program to process tasks.

```sh
go run workers/workers.go
```

**Note**: This will not exit until you send a signal to terminate the program. See [Signal Wiki page](https://github.com/hibiken/asynq/wiki/Signals) for best practice on how to safely terminate background workers.

You should be able to see some text printed in your terminal indicating that the task was processed successfully.

You can run the `client` program again to see how workers pick up the tasks and process them.

### Task Retry

It's not uncommon that a task doesn't get processed successfully in the first attempt. By default, a failed task will be retried with exponential backoff up to 25 times. 
Let's update our handler to return an error to simulate an unsuccessful scenario.

```go
// tasks.go
func HandleWelcomeEmailTask(ctx context.Context, t *asynq.Task) error {
    var p emailTaskPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Attempting to Send Welcome Email to User %d...", p.UserID)
    return fmt.Errorf("could not send email to the user") // <-- Return error 
}
```

Let's restart our workers program and enqueue a task.

```sh
go run workers/workers.go

go run client/client.go
```

If you are running `asynq dash`, you should be able to see that there's a task in the **Retry** state (by navigating to the queue details view and highlighting the "retry" tab).

To inspect which tasks are in retry state, you can also run

```sh
asynq task ls --queue=default --state=retry
```

This will list all the task that will be retried in the future. The output includes ETA of the task's next execution. 

Once a task exhausts its retry count, the task will transition to the **Archived** state and won't be retried (You can still manually run archived tasks using CLI or WebUI tool). 

Let's fix our handler before we wrap up this tutorial.

```go
func HandleWelcomeEmailTask(ctx context.Context, t *asynq.Task) error {
    var p emailTaskPayload
    if err := json.Unmarshal(t.Payload(), &p); err != nil {
        return err
    }
    log.Printf(" [*] Send Welcome Email to User %d", p.UserID)
    return nil 
}
```

Now that we fixed the handler, task will be processed successfully in the next attempt :)

This was a whirlwind tour of `asynq` basics. To learn more about all of its features such as **[priority queues](https://github.com/hibiken/asynq/wiki/Queue-Priority)** and **[custom retry](https://github.com/hibiken/asynq/wiki/Task-Retry)**, see our [Wiki page](https://github.com/hibiken/asynq/wiki).

Thanks for reading!