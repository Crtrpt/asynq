In this page, I'll explain the design behind the [`Handler`](https://pkg.go.dev/github.com/hibiken/asynq?tab=doc#Handler) interface.  

## Handler Interface

Core of your asynchronous task processing logic lives inside the [`Handler`](https://pkg.go.dev/github.com/hibiken/asynq?tab=doc#Handler) you provide to run a server.  
Handler's responsibility is to take a task and process it, while taking the context into account.  
It should report any errors to retry the task later, if the processing is unsuccessful.

Here's the interface definition:

```go
type Handler interface {
    ProcessTask(context.Context, *Task) error
}
```

It's a simple interface which describes the Handler's responsibility succinctly.


## Implementing Interface
 
Implementing this handler interface can be done in many ways.   

Here's an example of defining your own struct type to process tasks. 
```go
type MyTaskHandler struct {
   // ... fields
}

func (h *MyTaskHandler) ProcessTask(ctx context.Context, t *asynq.Task) error {
   // ... task processing logic
}
```

You can even define a function to satisfy the interface, thanks to the [`HandlerFunc`](https://pkg.go.dev/github.com/hibiken/asynq?tab=doc#HandlerFunc) adapter type.

```go
func myHandler(ctx context.Context, t *asynq.Task) error {
    // ... task processing logic
}

// h satisfies Handler
h := asynq.HandlerFunc(myHandler) 
```

In most cases, you'd probably want to examine the `Type` of the input task and process it accordingly.

```go
func (h *MyTaskHandler) ProcessTask(ctx context.Context, t *asynq.Task) error {
   switch t.Type() {
   case "type1":
      // process type 1
   case "type2":
      // process type2
   case "typeN":
      // process typeN

   default:
      return fmt.Errorf("unexpected task type: %q", t.Type())
   }
}
```

You can see that a Handler can be composed of many different handlers, each case in the above example can be handled by a dedicated handler. This is where [`ServeMux`](https://pkg.go.dev/github.com/hibiken/asynq?tab=doc#ServeMux) type can be useful.

## Using ServeMux 

*NOTE: You don't have to use `ServeMux` type to implement a Handler, but it can be useful in many cases.*  
With [`ServeMux`](https://pkg.go.dev/github.com/hibiken/asynq?tab=doc#ServeMux), you can register multiple Handlers. **It matches the type of each task against a list of registered patterns and calls the handler for the pattern that most closely matches the task's type name.**

```go
mux := asynq.NewServeMux()
mux.Handle("email:welcome", welcomeEmailHandler)
mux.Handle("email:reminder", reminderEmailHandler)
mux.Handle("email:" defaultEmailHandler) // catchall for all other task types with a prefix "email:" 
```

## Using Middleware

If you need to execute some code before and/or after handlers, you can accomplish that using middlewares.
Middleware is a function that takes a `Handler` and returns a `Handler`.  
Here's an example of a middleware that logs the start and end of task processing.

```go
func loggingMiddleware(h asynq.Handler) asynq.Handler {
    return asynq.HandlerFunc(func(ctx context.Context, t *asynq.Task) error {
        start := time.Now()
        log.Printf("Start processing %q", t.Type())
        err := h.ProcessTask(ctx, t)
        if err != nil {
            return err
        }
        log.Printf("Finished processing %q: Elapsed Time = %v", t.Type(), time.Since(start))
        return nil
    })
}
```

And now you can *wrap* your Handler with this middleware.

```go
myHandler = loggingMiddleware(myHandler)
```

Alternatively, if you are using `ServeMux` you can provide middlewares like this.

```go
mux := NewServeMux()
mux.Use(loggingMiddleware)
```

### Grouping middlewares

If you have a situation where you want to apply a middleware to a group of tasks, you can accomplish it by composing multiple `ServeMux` instances. *One limitation is that the tasks in each group need to have the same prefix in their type name*.

**Example:**    
If you have some tasks process orders and some tasks process products, and you want to apply one shared logic for all "product" tasks and another shared logic for all "order" tasks, you can achieve it like this:

```go
productHandlers := asynq.NewServeMux()
productHandlers.Use(productMiddleware) // shared logic for all product tasks
productHandlers.HandleFunc("product:update", productUpdateTaskHandler)
// ... register other "product" task handlers

orderHandlers := asynq.NewServeMux()
orderHandler.Use(orderMiddleware) // shared logic for all order tasks
orderHandlers.HandleFunc("order:refund", orderRefundTaskHandler)
// ... register other "order" task handlers.

// Top level handler
mux := asynq.NewServeMux()
mux.Use(someGlobalMiddleware) // shared logic for all tasks
mux.Handle("product:", productHandlers)
mux.Handle("order:", orderHandlers)

if err := srv.Run(mux); err != nil {
    log.Fatal(err)
}
``` 






