runc源码
================================================================
## 简介
docker为了容器标准化，安装OCF格式创建了runc，可以使用runc直接创建容器。在docker中，名称为docker-runc。

## 入口
路径：

    github.com/opencontainers/runc/main.go

定义：

    func main() {
        //创建App对象，跟前面docker-containerd中一样。
        app := cli.NewApp()
        app.Name = "runc"
        app.Usage = usage

        var v []string
        if version != "" {
            v = append(v, version)
        }
        if gitCommit != "" {
            v = append(v, fmt.Sprintf("commit: %s", gitCommit))
        }
        v = append(v, fmt.Sprintf("spec: %s", specs.Version))
        app.Version = strings.Join(v, "\n")

        //设置flag参数
        app.Flags = []cli.Flag{
            cli.BoolFlag{
                Name:  "debug",
                Usage: "enable debug output for logging",
            },
            cli.StringFlag{
                Name:  "log",
                Value: "/dev/null",
                Usage: "set the log file path where internal debug information is written",
            },
            cli.StringFlag{
                Name:  "log-format",
                Value: "text",
                Usage: "set the format used by logs ('text' (default), or 'json')",
            },
            cli.StringFlag{
                Name:  "root",
                Value: "/run/runc",                                                                              //会被docker-containerd-shim传入的flag参数覆盖
                Usage: "root directory for storage of container state (this should be located in tmpfs)",
            },
            cli.StringFlag{
                Name:  "criu",                                                                                   //支持检测点创建和恢复
                Value: "criu",
                Usage: "path to the criu binary used for checkpoint and restore",
            },
            cli.BoolFlag{
                Name:  "systemd-cgroup",
                Usage: "enable systemd cgroup support, expects cgroupsPath to be of form \"slice:prefix:name\" for e.g. \"system.slice:runc:434234\"",
            },
        }

        //各种子命令
        app.Commands = []cli.Command{
            checkpointCommand,
            createCommand,
            deleteCommand,
            eventsCommand,
            execCommand,
            initCommand,
            killCommand,
            listCommand,
            pauseCommand,
            psCommand,
            restoreCommand,
            resumeCommand,
            runCommand,
            specCommand,
            startCommand,
            stateCommand,
            updateCommand,
        }

        //运行前执行的函数，这里注册
        app.Before = func(context *cli.Context) error {
            if context.GlobalBool("debug") {
                logrus.SetLevel(logrus.DebugLevel)
            }
            if path := context.GlobalString("log"); path != "" {
                f, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_APPEND|os.O_SYNC, 0666)
                if err != nil {
                    return err
                }
                logrus.SetOutput(f)
            }
            switch context.GlobalString("log-format") {
            case "text":
                // retain logrus's default.
            case "json":
                logrus.SetFormatter(new(logrus.JSONFormatter))
            default:
                return fmt.Errorf("unknown log-format %q", context.GlobalString("log-format"))
            }
            return nil
        }

        // If the command returns an error, cli takes upon itself to print
        // the error on cli.ErrWriter and exit.
        // Use our own writer here to ensure the log gets sent to the right location.
        cli.ErrWriter = &FatalWriter{cli.ErrWriter}
        if err := app.Run(os.Args); err != nil {
            fatal(err)
        }
    }

### 1. app.Run
含义：

    运行runc。

路径：

    github.com/opencontainers/runc/Godeps/_workspace/src/github.com/urfave/cli/app.go

定义：

    func (a *App) Run(arguments []string) (err error) {
        a.Setup()

        // parse flags
        set := flagSet(a.Name, a.Flags)
        set.SetOutput(ioutil.Discard)
        err = set.Parse(arguments[1:])         //解析flag参数
        nerr := normalizeFlags(a.Flags, set)
        context := NewContext(a, set, nil)
        if nerr != nil {
            fmt.Fprintln(a.Writer, nerr)
            ShowAppHelp(context)
            return nerr
        }

        if checkCompletions(context) {
            return nil
        }

        if err != nil {
            if a.OnUsageError != nil {
                err := a.OnUsageError(context, err, false)
                HandleExitCoder(err)
                return err
            }
            fmt.Fprintf(a.Writer, "%s\n\n", "Incorrect Usage.")
            ShowAppHelp(context)
            return err
        }

        if !a.HideHelp && checkHelp(context) {
            ShowAppHelp(context)
            return nil
        }

        if !a.HideVersion && checkVersion(context) {
            ShowVersion(context)
            return nil
        }

        if a.After != nil {
            defer func() {
                if afterErr := a.After(context); afterErr != nil {
                    if err != nil {
                        err = NewMultiError(err, afterErr)
                    } else {
                        err = afterErr
                    }
                }
            }()
        }

        if a.Before != nil {
            beforeErr := a.Before(context)
            if beforeErr != nil {
                fmt.Fprintf(a.Writer, "%v\n\n", beforeErr)
                ShowAppHelp(context)
                HandleExitCoder(beforeErr)
                err = beforeErr
                return err
            }
        }

        args := context.Args()
        if args.Present() {
            name := args.First()
            c := a.Command(name)
            if c != nil {
                return c.Run(context)
            }
        }

        // Run default Action
        err = HandleAction(a.Action, context)

        HandleExitCoder(err)
        return err
    }

### 2. createCommand
含义：

    创建容器子命令。

路径：

    github.com/opencontainers/runc/create.go

定义：

    var createCommand = cli.Command{
        Name:  "create",
        Usage: "create a container",
        ArgsUsage: `<container-id>

    Where "<container-id>" is your name for the instance of the container that you
    are starting. The name you provide for the container instance must be unique on
    your host.`,
        Description: `The create command creates an instance of a container for a bundle. The bundle
    is a directory with a specification file named "` + specConfig + `" and a root
    filesystem.

    The specification file includes an args parameter. The args parameter is used
    to specify command(s) that get run when the container is started. To change the
    command(s) that get executed on start, edit the args parameter of the spec. See
    "runc spec --help" for more explanation.`,
        Flags: []cli.Flag{
            cli.StringFlag{
                Name:  "bundle, b",
                Value: "",
                Usage: `path to the root of the bundle directory, defaults to the current directory`,
            },
            cli.StringFlag{
                Name:  "console",
                Value: "",
                Usage: "specify the pty slave path for use with the container",
            },
            cli.StringFlag{
                Name:  "pid-file",
                Value: "",
                Usage: "specify the file to write the process id to",
            },
            cli.BoolFlag{
                Name:  "no-pivot",
                Usage: "do not use pivot root to jail process inside rootfs.  This should be used whenever the rootfs is on top of a ramdisk",
            },
            cli.BoolFlag{
                Name:  "no-new-keyring",
                Usage: "do not create a new session keyring for the container.  This will cause the container to inherit the calling processes session key",
            },
        },
        Action: func(context *cli.Context) error {
            spec, err := setupSpec(context)
            if err != nil {
                return err
            }
            status, err := startContainer(context, spec, true)
            if err != nil {
                return err
            }
            // exit with the container's exit status so any external supervisor is
            // notified of the exit with the correct exit status.
            os.Exit(status)
            return nil
        },
    }

_______________________________________________________________________
[[返回README.md]](../README.md) 

