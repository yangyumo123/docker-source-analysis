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
            Version:      "0.0.0",                 //程序版本，后面会被重新设置。
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
        // The name of the program. Defaults to path.Base(os.Args[0])
        Name string
        // Full name of command for help, defaults to Name
        HelpName string
        // Description of the program.
        Usage string
        // Text to override the USAGE section of help
        UsageText string
        // Description of the program argument format.
        ArgsUsage string
        // Version of the program
        Version string
        // List of commands to execute
        Commands []Command
        // List of flags to parse
        Flags []Flag
        // Boolean to enable bash completion commands
        EnableBashCompletion bool
        // Boolean to hide built-in help command
        HideHelp bool
        // Boolean to hide built-in version flag and the VERSION section of help
        HideVersion bool
        // An action to execute when the bash-completion flag is set
        BashComplete func(context *Context)
        // An action to execute before any subcommands are run, but after the context is ready
        // If a non-nil error is returned, no subcommands are run
        Before func(context *Context) error
        // An action to execute after any subcommands are run, but after the subcommand has finished
        // It is run even if Action() panics
        After func(context *Context) error
        // The action to execute when no subcommands are specified
        Action func(context *Context)
        // Execute this function if the proper command cannot be found
        CommandNotFound func(context *Context, command string)
        // Execute this function, if an usage error occurs. This is useful for displaying customized usage error messages.
        // This function is able to replace the original error messages.
        // If this function is not set, the "Incorrect usage" is displayed and the execution is interrupted.
        OnUsageError func(context *Context, err error, isSubcommand bool) error
        // Compilation date
        Compiled time.Time
        // List of all authors who contributed
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

