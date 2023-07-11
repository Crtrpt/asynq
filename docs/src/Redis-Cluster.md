This wiki page describes how to use Redis Cluster as a message broker with Asynq.
This wiki assumes that you've read [Redis Cluster Tutorial](https://redis.io/topics/cluster-tutorial), and you have 6 instance Redis Cluster running locally as described in the tutorial.

### Advantages of using Redis Cluster 
With Redis cluster, you get
* Ability to easily shard data across multiple Redis nodes
* Ability to stay available when some nodes are failing
* Ability to automatically failover

### Overview

![Cluster Queue Diagram](https://github.com/hibiken/asynq/raw/master/docs/assets/cluster.png)

Asynq shards data by queue.  
In the above diagram, we have 6-instance Redis Cluster (3 masters, 3 slaves) and 4 queues (q1, q2, q3, q4).
* Master1 (and its replica Slave1) hosts q1 and q2.
* Master2 (and its replica Slave2) hosts q3.
* Master3 (and its replica Slave3) hosts q4.

When you enqueue a task using `asynq.Client` you can specify the queue with the `Queue` option.  
The enqueued tasks will be consumed by `asynq.Server`(s) that are pulling tasks from those queues.

### Tutorial

In this section, we are going to look at how to use Redis Cluster as a message broker with Asynq.  
We assume that you are running 6-instance Redis cluster on port 7000-7005 as described in this [Redis Cluster Tutorial](https://redis.io/topics/cluster-tutorial).  
Here's an example `redis.conf` file:

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

Next, we are going to create two binaries: client and worker.

```sh
go mod init asynq-redis-cluster-quickstart
mkdir client worker
touch client/client.go worker/worker.go
```

In `client.go`, we are going to create a new `asynq.Client` and specify how to connect to Redis Cluster by passing `RedisClusterClientOpt`.

```go
client := asynq.NewClient(asynq.RedisClusterClientOpt{
    Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
})
```

Once we have the client, we are going to create tasks and enqueue them to three different queues: 
* notifications
* webhooks
* images

```go
// client.go

package main

import (
    "fmt"
    "log"

    "github.com/hibiken/asynq"
)

// List of queue names.
const (
     QueueNotifications = "notifications"
     QueueWebhooks      = "webhooks"
     QueueImages        = "images"
)

func main() {
    client := asynq.NewClient(asynq.RedisClusterClientOpt{
        Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
    })
    defer client.Close()

    // Create "notifications:email" task and enqueue it to "notifications" queue.
    task := asynq.NewTask("notifications:email", map[string]interface{}{"to": 123, "from": 456})
    res, err := client.Enqueue(task, asynq.Queue(QueueNotifications))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("successfully enqueued: %+v\n", res)

    // Create "webhooks:sync" task and enqueue it to "webhooks" queue.
    task = asynq.NewTask("webhooks:sync", map[string]interface{}{"data": 123})
    res, err = client.Enqueue(task, asynq.Queue(QueueWebhooks))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("successfully enqueued: %+v\n", res)

    // Create "images:resize" task and enqueue it to "images" queue.
    task = asynq.NewTask("images:resize", map[string]interface{}{"src": "some/path/to/image"})
    res, err = client.Enqueue(task, asynq.Queue(QueueImages))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("successfully enqueued: %+v\n", res)
}
```

Let's run this to enqueue three tasks.

```sh
go run client/client.go
```

Now let's move on to the worker to consume those tasks.
In `worker.go`, we are going to create a `asynq.Server` that consumes tasks from the three queues.
Again, we'll use `RedisClusterClientOpt` to connect to our Redis Cluster.

```go
// worker.go

package main

import (
    "context"
    "fmt"
    "log"

    "github.com/hibiken/asynq"
)

func main() {
    redisConnOpt := asynq.RedisClusterClientOpt{
        Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
    }

    srv := asynq.NewServer(redisConnOpt, asynq.Config{
        Concurrency: 20,
        // we'll give each queue the same priority number here.
        Queues: map[string]int{
            "notifications": 1,
            "webhooks":      1,
            "images":        1,
        },
    })

    mux := asynq.NewServeMux()
    mux.HandleFunc("notifications:email", handleEmailTask)
    mux.HandleFunc("webhooks:sync", handleWebhookSyncTask)
    mux.HandleFunc("images:resize", handleImageResizeTask)

    if err := srv.Run(mux); err != nil {
        log.Fatalf("Could not start a server: %v", err)
    }
}

func handleEmailTask(ctx context.Context, t *asynq.Task) error {
    to, err := t.Payload.GetInt("to")
    if err != nil {
        return err
    }
    from, err := t.Payload.GetInt("from")
    if err != nil {
        return err
    }
    fmt.Printf("Send email from %d to %d\n", from, to)
    return nil
}

func handleWebhookSyncTask(ctx context.Context, t *asynq.Task) error {
    data, err := t.Payload.GetInt("data")
    if err != nil {
        return err
    }
    fmt.Printf("Handle webhook task: %d\n", data)
    return nil
}

func handleImageResizeTask(ctx context.Context, t *asynq.Task) error {
    src, err := t.Payload.GetString("src")
    if err != nil {
        return err
    }
    fmt.Printf("Resize image: %s\n", src)
    return nil
}
```

Let's run this worker server to processed the three tasks we created earlier.

```sh
go run worker/worker.go
```

You should be able to see the message printed from each handler.

### Redis Nodes and Queues

As described in the overview section, Asynq shards data by queue. All tasks enqueued to the same queue belongs to the same Redis node.  
But which Redis node hosts which queues?  

We can use CLI to answer that question.

```sh
asynq queue ls --cluster
```

This command will print a list of queues along with:
- cluster nodes the queue belongs to
- cluster hash slot the queue maps to

The output may look something like this:

```sh
Queue          Cluster KeySlot  Cluster Nodes
-----          ---------------  -------------
images         9450             [{d54231bccd6c1765ea15caf95a41c67b10b91e58 127.0.0.1:7001} {70a7d4569eac28eed577ee91863703ffab98d2e0 127.0.0.1:7005}]
webhooks       4418             [{d58959f6057ad0911d92d86d1d16dc2242e9ec48 127.0.0.1:7004} {e2fb9f1296a8d3a49818e0f9be3bfd74fdc052ea 127.0.0.1:7000}]
notifications  16340            [{c738a8a98c5f5f9161e9563fa739f9c8191b7f1a 127.0.0.1:7002} {18cdaa0712191d74656f08017371df41eeaad5fa 127.0.0.1:7003}]
```

You can run `redis-cli --cluster reshard` command to move queue from one node to another. Note that operations may become unavailable for some time during resharding since Asynq uses multi-key operations.


***

This was a quick walkthrough of how to use Redis Cluster with Asynq.
If you have any questions or feature requests, please create an issue on Github.
Thank you!

