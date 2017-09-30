app.Run
==========================================================
## 简介
运行contaienrd。

## app.Run
含义：

    运行containerd。

路径：

    github.com/docker/containerd/vendor/src/github.com/codegangsta/cli/app.go

定义：

    func (a *App) Run(arguments []string) (err error) {
        if a.Author != "" || a.Email != "" {
            a.Authors = append(a.Authors, Author{Name: a.Author, Email: a.Email})
        }

        //设置command的HelpName
        newCmds := []Command{}
        for _, c := range a.Commands {
            if c.HelpName == "" {
                c.HelpName = fmt.Sprintf("%s %s", a.HelpName, c.Name)
            }
            newCmds = append(newCmds, c)
        }
        a.Commands = newCmds

        // 添加帮助到命令中
        if a.Command(helpCommand.Name) == nil && !a.HideHelp {
            a.Commands = append(a.Commands, helpCommand)
            if (HelpFlag != BoolFlag{}) {
                a.appendFlag(HelpFlag)
            }
        }

        // 添加 version/help flags
        if a.EnableBashCompletion {
            a.appendFlag(BashCompletionFlag)
        }

        if !a.HideVersion {
            a.appendFlag(VersionFlag)
        }

        // 解析flag，覆盖FlagSet中的默认值，重新context。
        set := flagSet(a.Name, a.Flags)
        set.SetOutput(ioutil.Discard)
        err = set.Parse(arguments[1:])
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
                return err
            } else {
                fmt.Fprintf(a.Writer, "%s\n\n", "Incorrect Usage.")
                ShowAppHelp(context)
                return err
            }
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

        //执行前面注册的Before函数。
        if a.Before != nil {
            err = a.Before(context)
            if err != nil {
                fmt.Fprintf(a.Writer, "%v\n\n", err)
                ShowAppHelp(context)
                return err
            }
        }

        //返回命令行的args参数
        args := context.Args()  //默认为空
        if args.Present() {     //如果存在args，则Present返回true
            name := args.First()
            c := a.Command(name)
            if c != nil {
                return c.Run(context)
            }
        }

        // Run default Action
        a.Action(context)
        return nil
    }


