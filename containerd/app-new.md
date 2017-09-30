创建命令行对象
======================================================
## 简介
创建新的cli application。

## NewApp
含义：

    创建App结构体。APP是命令行对象的主要结构体。后续的运行containerd命令，需要使用该结构体的信息。

路径：

    github.com/docker/contaienrd/vendor/src/github.com/codegangsta/cli/app.go

定义：

    func NewApp() *App {
        return &App{
            Name:         path.Base(os.Args[0]),   //程序名称，此处由args参数传入，后面会被重新设置为"containerd"。
            HelpName:     path.Base(os.Args[0]),   //help命令时的名字。
            Usage:        "A new cli application", //使用说明，后面会被重新设置为"High performance container daemon"。
            UsageText:    "",                      //覆盖help的Usage段。
            Version:      "0.0.0",                 //程序版本，后面会被重新设置，设置为containerd的git版本和gitcommit号。
            BashComplete: DefaultAppComplete,      //打印子命令列表。
            Action:       helpCommand.Action,      //没有指定子命令时执行该函数，后面会注册该函数的运行逻辑。
            Compiled:     compileTime(),           //程序完成时间。
            Writer:       os.Stdout,
        }
    }

## App结构体
含义：

    命令行对象的结构体。

路径：

    github.com/docker/contaienrd/vendor/src/github.com/codegangsta/cli/app.go

定义：

    type App struct {
        // 程序的名字。默认是path.Base(os.Args[0])
        Name string
        // help命令的完整名字，默认等于Name
        HelpName string
        // 程序的描述
        Usage string
        // 覆盖help命令中USAGE段的文本
        UsageText string
        // argument格式的描述
        ArgsUsage string
        // 程序版本
        Version string
        // 执行的命令列表
        Commands []Command
        // flags列表
        Flags []Flag
        // 是否支持bash命令补全
        EnableBashCompletion bool
        // 是否隐藏内置的help命令
        HideHelp bool
        // 是否隐藏内置version flag和help中的VERSION段
        HideVersion bool
        // 当设置bash-completion flag时，执行的函数
        BashComplete func(context *Context)
        // 在subcommands运行之前，context准备好之后执行的函数。non-nil error表示没有运行subcommands。
        Before func(context *Context) error
        // 所有subcommands运行之后，finished之前执行的函数。
        After func(context *Context) error
        // 当没有指定subcommands时，执行的函数。
        Action func(context *Context)
        // 如果没有找到合适的命令，则执行这个函数。
        CommandNotFound func(context *Context, command string)
        // Execute this function, if an usage error occurs. This is useful for displaying customized usage error messages.
        // This function is able to replace the original error messages.
        // If this function is not set, the "Incorrect usage" is displayed and the execution is interrupted.
        OnUsageError func(context *Context, err error, isSubcommand bool) error
        // 编译时间
        Compiled time.Time
        // 贡献者列表
        Authors []Author
        // Copyright of the binary if any
        Copyright string
        // Name of Author (Note: Use App.Authors, this is deprecated)
        Author string
        // Email of Author (Note: Use App.Authors, this is deprecated)
        Email string
        // Writer writer to write output to
        Writer io.Writer
    }

