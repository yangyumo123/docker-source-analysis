app.Before
========================================================
## 简介
向命令行对象结构体App的Before参数注册一个函数，类型：func(context *cli.Context)error。

Before表示在context准备好之后，在所有子命令运行之前执行的一个函数。如果返回非nil，说明没有子命令运行。

## app.Before
路径：

    github.com/docker/containerd/contaienrd/main.go

定义：

    func main() {
        ...
        app.Before = func(context *cli.Context) error {
            //dump runtime stack
            setupDumpStacksTrap()

            //debug
            if context.GlobalBool("debug") {
                logrus.SetLevel(logrus.DebugLevel)
                if context.GlobalDuration("metrics-interval") > 0 {
                    if err := debugMetrics(context.GlobalDuration("metrics-interval"), context.GlobalString("graphite-address")); err != nil {
                        return err
                    }
                }
            }

            //监听pprof事件的地址。前面设置了flag默认值为空。
            if p := context.GlobalString("pprof-address"); len(p) > 0 {
                pprof.Enable(p)
            }

            //检查系统资源限制，RLIMIT是内核的资源限制（resource limit），用户可以通过系统调用对各种资源限制，这里仅检查RLIMIT_NOFILE资源，表示比进程所打开的最大文件描述符大一的值。该值如果小于minRlimit(1024)，则设置RLIMIT_NOFILE的cur变成max。
            if err := checkLimits(); err != nil {
                return err
            }
            return nil
        }
        ...
    }

## Context
含义：

    Context是一个类型，传递给命令行对象中的action函数。Context用于获取context-specific args和解析后的命令行参数。

路径：

    github.com/docker/containerd/vendor/src/github.com/codegangsta/cli/context.go

定义：

    type Context struct {
        App            *App
        Command        Command
        flagSet        *flag.FlagSet
        setFlags       map[string]bool
        globalSetFlags map[string]bool
        parentContext  *Context
    }

