containerd-shim
======================================================
## 简介
下面介绍github.com/docker/containerd/containerd-shim进程

## 1.入口
路径：

    github.com/docker/containerd/containerd-shim/main.go

参数：

    arg0：container id
    arg1：bundle path
    arg2：runtime binary
    例子：docker-containerd-shim 9decb150527a3bxxxxxxxxxx /var/run/docker/libcontainerd/9decb150527a3bxxxxxxxxxx docker-runnc

定义：

    func main() {
        //flag解析，前面已经介绍过flag的相关原理和操作了，这里就是解析命令后面的参数。
        flag.Parse()

        //获得当前工作路径
        cwd, err := os.Getwd()
        if err != nil {
            panic(err)
        }

        //创建当前目录下的shim-log.json文件，用于记录日志。
        f, err := os.OpenFile(filepath.Join(cwd, "shim-log.json"), os.O_CREATE|os.O_WRONLY|os.O_APPEND|os.O_SYNC, 0666)
        if err != nil {
            panic(err)
        }

        //启动程序
        if err := start(f); err != nil {
            // this means that the runtime failed starting the container and will have the
            // proper error messages in the runtime log so we should to treat this as a
            // shim failure because the sim executed properly
            if err == errRuntime {
                f.Close()
                return
            }
            // log the error instead of writing to stderr because the shim will have
            // /dev/null as it's stdio because it is supposed to be reparented to system
            // init and will not have anyone to read from it
            writeMessage(f, "error", err)
            f.Close()
            os.Exit(1)
        }
    }


### 1.1start
含义：

    启动containerd-shim

路径：

    github.com/docker/containerd/containerd-shim/main.go

定义：

    func start(log *os.File) error {
        //信号处理，在正常运行前或runtime退出前执行
        signals := make(chan os.Signal, 2048)
        signal.Notify(signals)

        //把shim设置为subreaper进程，用于收集所有container创建的孤儿进程，下面会介绍subreaper的概念。
        if err := osutils.SetSubreaper(1); err != nil {
            return err
        }

        // 打开exit管道
        f, err := os.OpenFile("exit", syscall.O_WRONLY, 0)
        if err != nil {
            return err
        }
        defer f.Close()
        //打开control管道
        control, err := os.OpenFile("control", syscall.O_RDWR, 0)
        if err != nil {
            return err
        }
        defer control.Close()

        //创建Process，参数：arg0，arg1，arg2
        p, err := newProcess(flag.Arg(0), flag.Arg(1), flag.Arg(2))
        if err != nil {
            return err
        }
        defer func() {
            if err := p.Close(); err != nil {
                writeMessage(log, "warn", err)
            }
        }()

        //执行命令
        if err := p.create(); err != nil {
            p.delete()
            return err
        }
        msgC := make(chan controlMessage, 32)
        go func() {
            for {
                var m controlMessage
                if _, err := fmt.Fscanf(control, "%d %d %d\n", &m.Type, &m.Width, &m.Height); err != nil {
                    continue
                }
                msgC <- m
            }
        }()
        var exitShim bool
        for {
            select {
            case s := <-signals:
                switch s {
                case syscall.SIGCHLD:
                    exits, _ := osutils.Reap(false)
                    for _, e := range exits {
                        // check to see if runtime is one of the processes that has exited
                        if e.Pid == p.pid() {
                            exitShim = true
                            writeInt("exitStatus", e.Status)
                        }
                    }
                }
                // runtime has exited so the shim can also exit
                if exitShim {
                    // Let containerd take care of calling the runtime
                    // delete.
                    // This is needed to be done first in order to ensure
                    // that the call to Reap does not block until all
                    // children of the container have died if init was not
                    // started in its own PID namespace.
                    f.Close()
                    p.Wait()
                    return nil
                }
            case msg := <-msgC:
                switch msg.Type {
                case 0:
                    // close stdin
                    if p.stdinCloser != nil {
                        p.stdinCloser.Close()
                    }
                case 1:
                    if p.console == nil {
                        continue
                    }
                    ws := term.Winsize{
                        Width:  uint16(msg.Width),
                        Height: uint16(msg.Height),
                    }
                    term.SetWinsize(p.console.Fd(), &ws)
                }
            }
        }
        return nil
    }

（1）subreaper

    在Linux中，当一个进程的父进程死亡之后，它就变成了孤儿进程，由init进程收养，并且管理它的死亡。
    但是，Linux内核3.4引入了PR_SET_CHILD_SUBREAPER概念，即允许设置当前进程为subreaper，即相当于init进程，收养孤儿进程，负责孤儿进程的死亡。
    在docker-container-shim中，就是把docker-container-shim设置为subreaper进程，收养由container创建的孤儿进程。

（2）newProcess

含义：

     创建process结构体。

路径：

     github.com/docker/containerd/contaienrd-shim/process.go

定义：

    func newProcess(id, bundle, runtimeName string) (*process, error) {
        p := &process{
            id:      id,              // container id
            bundle:  bundle,          // bundle path
            runtime: runtimeName,     // runtime binary
        }
        //设置进程状态
        s, err := loadProcess()
        if err != nil {
            return nil, err
        }
        p.state = s

        //加载检查点和检查点路径
        if s.CheckpointPath != "" {
            cpt, err := loadCheckpoint(s.CheckpointPath)  
            if err != nil {
                return nil, err
            }
            p.checkpoint = cpt
            p.checkpointPath = s.CheckpointPath
        }

        //打开 pre-created fifo用于容器读写，以便如果另一侧停止监听，它们仍然保存打开状态。
        if err := p.openIO(); err != nil {
            return nil, err
        }
        return p, nil
    }
    type process struct {
        sync.WaitGroup
        id string
        bundle string
        stdio *stdio
        exec bool
        containerPid int
        checkpoint *checkpoint
        checkpointpath string
        shimIO *IO
        stdinCloser io.Closer
        console *os.File
        consolePath string
        state *processState
        runtime string
    }
    //读取process.json文件，获取进程状态
    func loadProcess() (*processState, error) {
        f, err := os.Open("process.json")
        if err != nil {
            return nil, err
        }
        defer f.Close()
        var s processState
        if err := json.NewDecoder(f).Decode(&s); err != nil {
            return nil, err
        }
        return &s, nil
    }

（3）create

路径：

    github.com/docker/containerd/containerd-shim/process.go

定义：

    func (p *process) create() error {
        //获取当前目录
        cwd, err := os.Getwd()
        if err != nil {
            return err
        }
        //当前目录下的log.json
        logPath := filepath.Join(cwd, "log.json")
        //args设置--log，--log-format，还有p.state.RuntimeArgs，
        args := append([]string{
            "--log", logPath,
            "--log-format", "json",
        }, p.state.RuntimeArgs...)

        //下面是设置各种类型args
        if p.state.Exec {
            //exec
            args = append(args, "exec",
                "-d",
                "--process", filepath.Join(cwd, "process.json"),
                "--console", p.consolePath,
            )
        } else if p.checkpoint != nil {
            //restore
            args = append(args, "restore",
                "-d",
                "--image-path", p.checkpointPath,
                "--work-path", filepath.Join(p.checkpointPath, "criu.work", "restore-"+time.Now().Format(time.RFC3339)),
            )
            add := func(flags ...string) {
                args = append(args, flags...)
            }
            if p.checkpoint.Shell {
                add("--shell-job")
            }
            if p.checkpoint.TCP {
                add("--tcp-established")
            }
            if p.checkpoint.UnixSockets {
                add("--ext-unix-sk")
            }
            if p.state.NoPivotRoot {
                add("--no-pivot")
            }
            for _, ns := range p.checkpoint.EmptyNS {
                add("--empty-ns", ns)
            }

        } else {
            //create容器，
            args = append(args, "create",
                "--bundle", p.bundle,
                "--console", p.consolePath,
            )
            if p.state.NoPivotRoot {
                args = append(args, "--no-pivot")
            }
        }

        //设置--pid-file
        args = append(args,
            "--pid-file", filepath.Join(cwd, "pid"),
            p.id,
        )

        //创建cmd，p.runtime是runc
        cmd := exec.Command(p.runtime, args...)
        cmd.Dir = p.bundle        //准备的rootfs
        cmd.Stdin = p.stdio.stdin
        cmd.Stdout = p.stdio.stdout
        cmd.Stderr = p.stdio.stderr
        // 设置SysProcAttr
        cmd.SysProcAttr = setPDeathSig()

        //启动容器
        if err := cmd.Start(); err != nil {
            if exErr, ok := err.(*exec.Error); ok {
                if exErr.Err == exec.ErrNotFound || exErr.Err == os.ErrNotExist {
                    return fmt.Errorf("%s not installed on system", p.runtime)
                }
            }
            return err
        }
        p.stdio.stdout.Close()
        p.stdio.stderr.Close()
        if err := cmd.Wait(); err != nil {
            if _, ok := err.(*exec.ExitError); ok {
                return errRuntime
            }
            return err
        }
        data, err := ioutil.ReadFile("pid")
        if err != nil {
            return err
        }
        pid, err := strconv.Atoi(string(data))
        if err != nil {
            return err
        }
        p.containerPid = pid
        return nil
    }

（3.1）setPDeathSig

路径：

    github.com/docker/containerd/containerd-shim/process_pdeathsig.go

定义：

    //设置父进程SIGKILL信号，以便如果shim死了，容器进程也会被杀死。
    func setDeathSig() *syscall.SysProcAttr {
        return &syscall.SysProcAttr{
            Pdeathsig: syscall.SIGKILL,
        }
    }

（3.2）cmd.Start

路径：

    src/os/exec/exec.go

定义：

    func (c *Cmd) Start() error {
        if c.lookPathErr != nil {
            c.closeDescriptors(c.closeAfterStart)
            c.closeDescriptors(c.closeAfterWait)
            return c.lookPathErr
        }
        if runtime.GOOS == "windows" {
            lp, err := lookExtensions(c.Path, c.Dir)
            if err != nil {
                c.closeDescriptors(c.closeAfterStart)
                c.closeDescriptors(c.closeAfterWait)
                return err
            }
            c.Path = lp
        }
        if c.Process != nil {
            return errors.New("exec: already started")
        }
        if c.ctx != nil {
            select {
            case <-c.ctx.Done():
                c.closeDescriptors(c.closeAfterStart)
                c.closeDescriptors(c.closeAfterWait)
                return c.ctx.Err()
            default:
            }
        }

        type F func(*Cmd) (*os.File, error)
        for _, setupFd := range []F{(*Cmd).stdin, (*Cmd).stdout, (*Cmd).stderr} {
            fd, err := setupFd(c)
            if err != nil {
                c.closeDescriptors(c.closeAfterStart)
                c.closeDescriptors(c.closeAfterWait)
                return err
            }
            c.childFiles = append(c.childFiles, fd)
        }
        c.childFiles = append(c.childFiles, c.ExtraFiles...)

        var err error
        c.Process, err = os.StartProcess(c.Path, c.argv(), &os.ProcAttr{
            Dir:   c.Dir,
            Files: c.childFiles,
            Env:   c.envv(),
            Sys:   c.SysProcAttr,
        })
        if err != nil {
            c.closeDescriptors(c.closeAfterStart)
            c.closeDescriptors(c.closeAfterWait)
            return err
        }

        c.closeDescriptors(c.closeAfterStart)

        c.errch = make(chan error, len(c.goroutine))
        for _, fn := range c.goroutine {
            go func(fn func() error) {
                c.errch <- fn()
            }(fn)
        }

        if c.ctx != nil {
            c.waitDone = make(chan struct{})
            go func() {
                select {
                case <-c.ctx.Done():
                    c.Process.Kill()
                case <-c.waitDone:
                }
            }()
        }

        return nil
    }

_______________________________________________________________________
[[返回README.md]](../README.md) 

