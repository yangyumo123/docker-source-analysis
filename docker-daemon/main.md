main入口
==============================================================
## 简介
docker daemon的入口位置：github.com/docker/docker/cmd/dockerd.docker.go。

## main
定义：

    func main() {
        //如果进程已经执行（已经初始化），则退出。
        if reexec.Init() {
            return
        }

        // 设置仿真终端，这个依赖于平台。
        _, stdout, stderr := term.StdStreams()

        //设置日志输出，stderr。
        logrus.SetOutput(stderr)

        //flag参数合并，flag.CommandLine是目的，是docker自己写的一个包：github.com/docker/docker/pkg/mflg。将前面的FlagSet合并到该包的FlagSet中。
        flag.Merge(flag.CommandLine, daemonCli.commonFlags.FlagSet)

        //flag参数使用说明
        flag.Usage = func() {
            fmt.Fprint(stdout, "Usage: dockerd [OPTIONS]\n\n")
            fmt.Fprint(stdout, "A self-sufficient runtime for containers.\n\nOptions:\n")

            flag.CommandLine.SetOutput(stdout)
            flag.PrintDefaults()
        }
        //flag参数使用说明的简短描述
        flag.CommandLine.ShortUsage = func() {
            fmt.Fprint(stderr, "\nUsage:\tdockerd [OPTIONS]\n")
        }

        //解析命令行，解析传入的命令行参数，并创建对应的Flag，添加到FlagSet的actual中，同时更新FlagSet的formal。这个在apiserver已经详细介绍过了，这里就不介绍了。
        if err := flag.CommandLine.ParseFlags(os.Args[1:], false); err != nil {
            os.Exit(1)
        }

        //如果命令行参数是"--version"，则打印版本并退出。
        if *flVersion {
            showVersion()
            return
        }

        //如果命令行参数是"--help"，则打印帮助并退出。
        if *flHelp {
            // if global flag --help is present, regardless of what other options and commands there are,
            // just print the usage.
            flag.Usage()
            return
        }

        // linux返回值stop=false。所以会执行后面的daemonCli.start()
        stop, err := initService()
        if err != nil {
            logrus.Fatal(err)
        }

        if !stop {
            //这里是daemonCli启动的核心代码。在下一节中详细分析。
            err = daemonCli.start()
            notifyShutdown(err)
            if err != nil {
                logrus.Fatal(err)
            }
        }
    }




_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 
