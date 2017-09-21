containerd主函数
==================================================
## 简介
containerd的main函数，创建容器，管理容器生命周期等等。

## main
路径：

    github.com/docker/containerd/containerd/main.go

定义：

    func main() {
        logrus.SetFormatter(&logrus.TextFormatter{TimestampFormat: time.RFC3339Nano})

        //创建命令行对象
        app := cli.NewApp()

        //设置程序的名称
        app.Name = "containerd"
        //app版本
        if containerd.GitCommit != "" {
            app.Version = fmt.Sprintf("%s commit: %s", containerd.Version, containerd.GitCommit)
        } else {
            app.Version = containerd.Version
        }
        //设置app的简介，即前面介绍的常量
        app.Usage = usage
        //设置app的flag参数，即前面介绍的flag参数
        app.Flags = daemonFlags

        //设置app.Before
        app.Before = func(context *cli.Context) error {
            setupDumpStacksTrap()
            if context.GlobalBool("debug") {
                logrus.SetLevel(logrus.DebugLevel)
                if context.GlobalDuration("metrics-interval") > 0 {
                    if err := debugMetrics(context.GlobalDuration("metrics-interval"), context.GlobalString("graphite-address")); err != nil {
                        return err
                    }
                }
            }
            if p := context.GlobalString("pprof-address"); len(p) > 0 {
                pprof.Enable(p)
            }
            if err := checkLimits(); err != nil {
                return err
            }
            return nil
        }

        //设置app.Action
        app.Action = func(context *cli.Context) {
            if err := daemon(context); err != nil {
                logrus.Fatal(err)
            }
        }

        //运行app.Run
        if err := app.Run(os.Args); err != nil {
            logrus.Fatal(err)
        }
    }

## 目录
1. [创建命令行对象](./app-new.md)
2. [app.Before](./app-before.md)
3. [app.Action](./app-action.md)
3. [app.Run](./app-run.md)

_______________________________________________________________________
[[返回containerd.md]](./containerd.md) 
