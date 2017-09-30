app.Action
=====================================================
## 简介
向命令行对象结构体App的Action参数注册一个函数，类型：func(context *cli.Context)。在没有subcommands时执行Action函数。

## app.Action
路径：

    github.com/docker/containerd/contaienrd/main.go

定义：

    func main(){
        ...
        app.Action = func(context *cli.Context) {
            if err := daemon(context); err != nil {
                logrus.Fatal(err)
            }
        }
        ...
    }

## daemon
含义：

    创建containerd。使用supervisor来启动containerd http server，提供grpc API。

路径：

    github.com/docker/containerd/contaienrd/main.go

定义：

    func daemon(context *cli.Context) error {
        s := make(chan os.Signal, 2048)
        //处理SIGTERM和SIGINT信号，SIGTERM信号是kill不带-9发出的，让程序终止。SIGINT信号是Ctrl+c发出的，让程序终止。
        signal.Notify(s, syscall.SIGTERM, syscall.SIGINT)

        //创建supervisor，使用supervisor来启动daemon进程，并管理daemon的生命周期。
        sv, err := supervisor.New(
            context.String("state-dir"),
            context.String("runtime"),
            context.String("shim"),
            context.StringSlice("runtime-args"),
            context.Duration("start-timeout"),
            context.Int("retain-count"))
        if err != nil {
            return err
        }
        wg := &sync.WaitGroup{}
        for i := 0; i < 10; i++ {
            wg.Add(1)
            w := supervisor.NewWorker(sv, wg)
            go w.Start()                                                 //启动容器
        }
        if err := sv.Start(); err != nil {                               
            return err
        }
        // Split the listen string of the form proto://addr
        listenSpec := context.String("listen")                           //unix:///var/run/docker/libcontainerd/docker-containerd.sock
        listenParts := strings.SplitN(listenSpec, "://", 2)
        if len(listenParts) != 2 {
            return fmt.Errorf("bad listen address format %s, expected proto://address", listenSpec)
        }

        //启动docker-containerd http server
        server, err := startServer(listenParts[0], listenParts[1], sv)
        if err != nil {
            return err
        }
        for ss := range s {
            switch ss {
            default:
                logrus.Infof("stopping containerd after receiving %s", ss)
                server.Stop()
                os.Exit(0)
            }
        }
        return nil
    }

（1）supervisor.New

含义：

    创建Supervisor对象，管理containerd进程。

路径：

    github.com/docker/containerd/supervisor/supervisor.go

定义：

    func New(stateDir string, runtimeName, shimName string, runtimeArgs []string, timeout time.Duration, retainCount int) (*Supervisor, error) {
        startTasks := make(chan *startTask, 10)

        //创建/var/run/docker/libcontainerd/containerd目录
        if err := os.MkdirAll(stateDir, 0755); err != nil {
            return nil, err
        }

        //检查机器信息，返回cpu数量，内存数量。
        machine, err := CollectMachineInformation()
        if err != nil {
            return nil, err
        }

        //创建进程监视器
        monitor, err := NewMonitor()
        if err != nil {
            return nil, err
        }

        //创建Supervisor对象
        s := &Supervisor{
            stateDir:    stateDir,
            containers:  make(map[string]*containerInfo),     //containerInfo结构体包含runtime.Container接口
            startTasks:  startTasks,
            machine:     machine,
            subscribers: make(map[chan Event]struct{}),
            tasks:       make(chan Task, defaultBufferSize),
            monitor:     monitor,
            runtime:     runtimeName,                         //docker-runc
            runtimeArgs: runtimeArgs,
            shim:        shimName,                            //docker-containerd-shim
            timeout:     timeout,
        }
        if err := setupEventLog(s, retainCount); err != nil {
            return nil, err
        }
        go s.exitHandler()
        go s.oomHandler()
        if err := s.restore(); err != nil {
            return nil, err
        }
        return s, nil
    }

    //github.com/docker/containerd/supervisor/worker.go
    type startTask struct {
        Container      runtime.Container    //接口，定义了container上的一些操作。
        CheckpointPath string               //检查点路径
        Stdin          string
        Stdout         string
        Stderr         string
        Err            chan error
        StartResponse  chan StartResponse
    }

（2）w.Start()

含义：

    启动10个独立的goroutine，每个goroutine运行循环，负责启动新的容器。

路径：

    github.com/docker/containerd/supervisor/worker.go

定义：

    func (w *worker) Start() {
        //相当于wg.Add(-1)，表示goroutine执行完毕
        defer w.wg.Done()

        //循环从startTasks管道里面读对象，只要实现了runtime.Container接口的结构体都可以放到这个管道中，这是一个带缓存区的管道，缓冲区为空就阻塞了，所以说该goroutine不退出。
        for t := range w.s.startTasks {
            started := time.Now()

            //启动容器，执行docker-containerd-shim
            process, err := t.Container.Start(t.CheckpointPath, runtime.NewStdio(t.Stdin, t.Stdout, t.Stderr))
            if err != nil {
                logrus.WithFields(logrus.Fields{
                    "error": err,
                    "id":    t.Container.ID(),
                }).Error("containerd: start container")
                t.Err <- err
                evt := &DeleteTask{
                    ID:      t.Container.ID(),
                    NoEvent: true,
                    Process: process,
                }
                w.s.SendTask(evt)
                continue
            }

            //监控容器的OOM事件
            if err := w.s.monitor.MonitorOOM(t.Container); err != nil && err != runtime.ErrContainerExited {
                if process.State() != runtime.Stopped {
                    logrus.WithField("error", err).Error("containerd: notify OOM events")
                }
            }

            //监控进程状态
            if err := w.s.monitorProcess(process); err != nil {
                logrus.WithField("error", err).Error("containerd: add process to monitor")
                t.Err <- err
                evt := &DeleteTask{
                    ID:      t.Container.ID(),
                    NoEvent: true,
                    Process: process,
                }
                w.s.SendTask(evt)
                continue
            }
            // only call process start if we aren't restoring from a checkpoint
            // if we have restored from a checkpoint then the process is already started
            if t.CheckpointPath == "" {
                if err := process.Start(); err != nil {
                    logrus.WithField("error", err).Error("containerd: start init process")
                    t.Err <- err
                    evt := &DeleteTask{
                        ID:      t.Container.ID(),
                        NoEvent: true,
                        Process: process,
                    }
                    w.s.SendTask(evt)
                    continue
                }
            }
            ContainerStartTimer.UpdateSince(started)
            t.Err <- nil
            t.StartResponse <- StartResponse{
                Container: t.Container,
            }
            w.s.notifySubscribers(Event{
                Timestamp: time.Now(),
                ID:        t.Container.ID(),
                Type:      StateStart,
            })
        }
    }

    //github.com/docker/containerd/runtime/container.go
    func (c *container) Start(checkpointPath string, s Stdio) (Process, error) {
        processRoot := filepath.Join(c.root, c.id, InitProcessID)
        if err := os.Mkdir(processRoot, 0755); err != nil {
            return nil, err
        }
        //命令：docker-containerd-shim，参数有3个，前面已经介绍过了。
        cmd := exec.Command(c.shim,
            c.id, c.bundle, c.runtime,
        )
        cmd.Dir = processRoot
        cmd.SysProcAttr = &syscall.SysProcAttr{
            Setpgid: true,
        }
        spec, err := c.readSpec()
        if err != nil {
            return nil, err
        }
        config := &processConfig{
            checkpoint:  checkpointPath,
            root:        processRoot,
            id:          InitProcessID,
            c:           c,
            stdio:       s,
            spec:        spec,
            processSpec: specs.ProcessSpec(spec.Process),
        }
        p, err := newProcess(config)
        if err != nil {
            return nil, err
        }
        if err := c.createCmd(InitProcessID, cmd, p); err != nil {
            return nil, err
        }
        return p, nil
    }

（3）s.Start()

含义：

    运行supervisor，监控容器进程和执行新容器。

路径：

    github.com/docker/containerd/supervisor/supervisor.go

定义：

    func (s *Supervisor) Start() error {
        logrus.WithFields(logrus.Fields{
            "stateDir":    s.stateDir,
            "runtime":     s.runtime,
            "runtimeArgs": s.runtimeArgs,
            "memory":      s.machine.Memory,
            "cpus":        s.machine.Cpus,
        }).Debug("containerd: supervisor running")
        go func() {
            for i := range s.tasks {
                s.handleTask(i)
            }
        }()
        return nil
    }

    //github.com/docker/containerd/supervisor/supervisor.go
    func (s *Supervisor) handleTask(i Task) {
        var err error
        switch t := i.(type) {
        case *AddProcessTask:
            err = s.addProcess(t)
        case *CreateCheckpointTask:
            err = s.createCheckpoint(t)
        case *DeleteCheckpointTask:
            err = s.deleteCheckpoint(t)
        case *StartTask:
            err = s.start(t)
        case *DeleteTask:
            err = s.delete(t)
        case *ExitTask:
            err = s.exit(t)
        case *GetContainersTask:
            err = s.getContainers(t)
        case *SignalTask:
            err = s.signal(t)
        case *StatsTask:
            err = s.stats(t)
        case *UpdateTask:
            err = s.updateContainer(t)
        case *UpdateProcessTask:
            err = s.updateProcess(t)
        case *OOMTask:
            err = s.oom(t)
        default:
            err = ErrUnknownTask
        }
        if err != errDeferredResponse {
            i.ErrorCh() <- err
            close(i.ErrorCh())
        }
    }

（4）startServer

路径：

    github.com/docker/containerd/contaienrd/main.go

定义：

    func startServer(protocol, address string, sv *supervisor.Supervisor) (*grpc.Server, error) {
        // TODO: We should use TLS.
        // TODO: Add an option for the SocketGroup.
        sockets, err := listeners.Init(protocol, address, "", nil)
        if err != nil {
            return nil, err
        }
        if len(sockets) != 1 {
            return nil, fmt.Errorf("incorrect number of listeners")
        }
        l := sockets[0]
        s := grpc.NewServer()
        types.RegisterAPIServer(s, server.NewServer(sv))
        healthServer := health.NewHealthServer()
        grpc_health_v1.RegisterHealthServer(s, healthServer)

        go func() {
            logrus.Debugf("containerd: grpc api on %s", address)
            if err := s.Serve(l); err != nil {                                     //启动http server
                logrus.WithField("error", err).Fatal("containerd: serve grpc")
            }
        }()
        return s, nil
    }