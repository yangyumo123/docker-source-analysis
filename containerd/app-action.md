app.Action
=====================================================
## 简介
向命令行对象结构体App的Action参数注册一个函数，类型：func(context *cli.Context)。

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
            go w.Start()
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

（1）startServer

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