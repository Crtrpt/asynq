## Overview
If you'd like to add and remove periodic tasks dynamically (i.e. without restarting the Scheduler process), use `PeriodicTaskManager`.
The PeriodicTaskManager uses the `PeriodicTaskConfigProvider` to fetch the current periodic task configurations periodically and syncs the Scheduler's entries with the current configs.  

For example, you can store your periodic task configurations in a database or a local file and update this config source to dynamically add and remove periodic tasks. The Example below shows how this can be achieved with a local file, but you can easily modify the example to work with a database or other config sources.

## Example
In this example, we are going to store the periodic task config in a YAML file.
The yaml file `periodic_task_config.yml` looks like this.

```yml
configs:
  - cronspec: "* * * * *"
    task_type: foo

  - cronspec: "* * * * *"
    task_type: bar
```

Now we need to implement our `PeriodicTaskConfigProvider` which reads this file and return a list of `PeriodicTaskConfig`.

```go
func main() {
    provider := &FileBasedConfigProvider{filename: "./periodic_task_config.yml"}

    mgr, err := asynq.NewPeriodicTaskManager(
        asynq.PeriodicTaskManagerOpts{
            RedisConnOpt:               asynq.RedisClientOpt{Addr: "localhost:6379"},
            PeriodicTaskConfigProvider: provider,         // this provider object is the interface to your config source
            SyncInterval:               10 * time.Second, // this field specifies how often sync should happen
    })
    if err != nil {
        log.Fatal(err)
    }

    if err := mgr.Run(); err != nil {
         log.Fatal(err)
    }
}

// FileBasedConfigProvider implements asynq.PeriodicTaskConfigProvider interface.
type FileBasedConfigProvider struct {
     filename string
}

// Parses the yaml file and return a list of PeriodicTaskConfigs.
func (p *FileBasedConfigProvider) GetConfigs() ([]*asynq.PeriodicTaskConfig, error) {
    data, err := os.ReadFile(p.filename)
    if err != nil {
        return nil, err
    }
    var c PeriodicTaskConfigContainer
    if err := yaml.Unmarshal(data, &c); err != nil {
        return nil, err
    }
    var configs []*asynq.PeriodicTaskConfig
    for _, cfg := range c.Configs {
         configs = append(configs, &asynq.PeriodicTaskConfig{Cronspec: cfg.Cronspec, Task: asynq.NewTask(cfg.TaskType, nil)})
    }
    return configs, nil
}

type PeriodicTaskConfigContainer struct {
    Configs []*Config `yaml:"configs"`
}

type Config struct {
    Cronspec string `yaml:"cronspec"`
    TaskType string `yaml:"task_type"`
}

```

Run the go program above and while it's running, try changing the config file. You should see log messages which indicate that a new config is added or removed.


