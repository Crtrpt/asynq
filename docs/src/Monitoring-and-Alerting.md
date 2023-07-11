We recommend using a monitoring tool such as [Prometheus](https://prometheus.io/) to monitor your worker processes and queues in production.

## Queue metrics
If you are using the [Web UI](https://github.com/hibiken/asynqmon), you can enable integration with Prometheus by passing two flags:
* `--enable-metrics-exporter`: Enable collection of queue metrics and exports it under `/metrics` endpoint.
* `--prometheus-addr`: Enable visualization of queue metrics within Web UI.

The queue metrics page looks like this:

<img width="60%" alt="Screen Shot 2021-12-19 at 4 37 19 PM" src="https://user-images.githubusercontent.com/10953044/146777420-cae6c476-bac6-469c-acce-b2f6584e8707.png">

If you are not using the Web UI, Asynq ships with [a binary](https://github.com/hibiken/asynq/tree/master/tools/metrics_exporter) you can run to export queue metrics. It also has a package to collect queue metrics under [`x/metrics`](https://github.com/hibiken/asynq/tree/master/x/metrics).  

## Worker metrics

Asynq `Handler` interface and `ServeMux` can be instrumented with metrics tracking code.

Here is an example of using [Prometheus](https://prometheus.io) to export worker metrics.
We can instrument our code to track additional application specific metrics as well as the default metrics (e.g. memory, cpu) tracked by prometheus.

Application specific metrics we are tracking in the example code below:
* Total Number of tasks processed by the worker process (both successfully and failed)
* Number of failed tasks by the worker process
* Number of tasks currently being processed by the worker process.


```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "runtime"

    "github.com/hibiken/asynq"
    "github.com/hibiken/asynq/examples/tasks"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "golang.org/x/sys/unix"
)

// Metrics variables.
var (
    processedCounter = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "processed_tasks_total",
            Help: "The total number of processed tasks",
        },
        []string{"task_type"},
    )

    failedCounter = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "failed_tasks_total",
	    Help: "The total number of times processing failed",
	},
        []string{"task_type"},
    )

    inProgressGauge = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
	    Name: "in_progress_tasks",
	    Help: "The number of tasks currently being processed",
	},
        []string{"task_type"},
    )
)

func metricsMiddleware(next asynq.Handler) asynq.Handler {
    return asynq.HandlerFunc(func(ctx context.Context, t *asynq.Task) error {
        inProgressGauge.WithLabelValues(t.Type()).Inc()
        err := next.ProcessTask(ctx, t)
        inProgressGauge.WithLabelValues(t.Type()).Dec()
	if err != nil {
	    failedCounter.WithLabelValues(t.Type()).Inc()
	}
	processedCounter.WithLabelValues(t.Type()).Inc()
	return err
    })
}

func main() {
    httpServeMux := http.NewServeMux()
    httpServeMux.Handle("/metrics", promhttp.Handler())
    metricsSrv := &http.Server{
        Addr:    ":2112",
	Handler: httpServeMux,
    }
    done := make(chan struct{})

    // Start metrics server.
    go func() {
        err := metricsSrv.ListenAndServe()
	if err != nil && err != http.ErrServerClosed {
	    log.Printf("Error: metrics server error: %v", err)
	}
	close(done)
    }()

    srv := asynq.NewServer(
        asynq.RedisClientOpt{Addr: ":6379"},
	asynq.Config{Concurrency: 20},
    )

    mux := asynq.NewServeMux()
    mux.Use(metricsMiddleware)
    mux.HandleFunc(tasks.TypeEmail, tasks.HandleEmailTask)

    // Start worker server.
    if err := srv.Start(mux); err != nil {
        log.Fatalf("Failed to start worker server: %v", err)
    }

    // Wait for termination signal.
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, unix.SIGTERM, unix.SIGINT)
    <-sigs
 	
    // Stop worker server.
    srv.Shutdown()
	
    // Stop metrics server.
    if err := metricsSrv.Shutdown(context.Background()); err != nil {
        log.Printf("Error: metrics server shutdown error: %v", err)
    }
    <-done
}
```
