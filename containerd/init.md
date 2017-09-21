参数初始化
=====================================================
## 简介
在运行containerd主函数之前，先使用默认值初始化命令行flag参数，如果用户提供命令行flag参数，将覆盖默认值。

## flag
含义：

    默认值初始化flag参数，用户的命令行输入flag会覆盖默认值。dockerd在调用docker-containerd命令时，就是指定了flag，从而覆盖这里的默认值。
路径：

    github.com/docker/containerd/containerd/main.go

定义：

    const (
        usage               = `High performance container daemon`
        minRlimit           = 1024
        defaultStateDir     = "/run/containerd"                                     //containerd的根目录，docker daemon在调用docker-containerd命令时，传入flag参数：/var/run/docker/libcontainerd/containerd，覆盖了默认值。
        defaultGRPCEndpoint = "unix:///run/containerd/containerd.sock"              //containerd暴露给docker daemon的rpc监听地址。docker daemon在调用docker-containerd命令时，传入flag参数：unix:///var/run/docker/libcontainerd/docker-containerd.sock，覆盖了默认值。
    )

    //flag参数
    var daemonFlags = []cli.Flag{
        cli.BoolFlag{
            Name:  "debug",                                     //是否支持debug模式，docker-containerd也会设置debug。
            Usage: "enable debug output in the logs",
        },
        cli.StringFlag{
            Name:  "state-dir",                                 //docker daemon调用docker-containerd时，会用命令行参数："/var/run/docker/libcontainerd/containerd"覆盖该默认值
            Value: defaultStateDir,
            Usage: "runtime state directory",
        },
        cli.DurationFlag{
            Name:  "metrics-interval",                          //docker daemon调用docker-containerd时，会用命令行参数：0覆盖该默认值
            Value: 5 * time.Minute,
            Usage: "interval for flushing metrics to the store",
        },
        cli.StringFlag{
            Name:  "listen,l",                                  //docker daemon调用docker-containerd时，会用命令行参数："unix:///var/run/docker/libcontainerd/docker-containerd.sock"覆盖该默认值
            Value: defaultGRPCEndpoint,
            Usage: "proto://address on which the GRPC API will listen",
        },
        cli.StringFlag{
            Name:  "runtime,r",                                 //containerd使用的runtime类型。docker-containerd默认使用docker-runC。
            Value: "runc",
            Usage: "name or path of the OCI compliant runtime to use when executing containers",
        },
        cli.StringSliceFlag{
            Name:  "runtime-args",                              //可以指定containerd使用哪种runtime，docker-containerd默认使用docker-runC。用户可以指定自己的runtime。
            Value: &cli.StringSlice{},
            Usage: "specify additional runtime args",
        },
        cli.StringFlag{
            Name:  "shim",
            Value: "containerd-shim",                           //docker-containerd进程在启动后会创建docker-containerd-shim进程，该进程实现runC，实现容器创建，容器周期管理。docker daemon调用docker-containerd时，会用命令行参数："docker-containerd-shim"覆盖该默认值
            Usage: "Name or path of shim",
        },
        cli.StringFlag{
            Name:  "pprof-address",
            Usage: "http address to listen for pprof events",
        },
        cli.DurationFlag{
            Name:  "start-timeout",                             //docker daemon调用docker-containerd时，会用命令行参数："2m"覆盖该默认值
            Value: 15 * time.Second,
            Usage: "timeout duration for waiting on a container to start before it is killed",
        },
        cli.IntFlag{
            Name:  "retain-count",
            Value: 500,
            Usage: "number of past events to keep in the event log",
        },
        cli.StringFlag{
            Name:  "graphite-address",
            Usage: "Address of graphite server",
        },
    }

_______________________________________________________________________
[[返回containerd.md]](./containerd.md) 


