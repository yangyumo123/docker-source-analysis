创建containerd daemon
==================================================
## 简介
创建并启动containerd daemon进程，二进制文件是docker-containerd。

## 入口
含义：

    创建containerd daemon的入口。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go
    
定义： 

    func New(stateDir string, options ...RemoteOption) (_ Remote, err error) {
        ...
        if r.startDaemon {
            if err := r.runContainerdDaemon(); err != nil {
                return nil, err
            }
        }
        ...
    }

## runContainerdDaemon()
含义：

    创建docker-containerd守护进程。docker-containerd主要用于创建容器，管理容器生命周期。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

定义：

    func (r *remote) runContainerdDaemon() error {
        pidFilename := filepath.Join(r.stateDir, containerdPidFilename)        //docker-containerd进程的pid文件路径，取值：/var/run/docker/libcontainerd/docker-containerd.pid
        f, err := os.OpenFile(pidFilename, os.O_RDWR|os.O_CREATE, 0600)        //打开docker-containerd进程的pid文件，如果不存在，则创建该文件，权限0600
        defer f.Close()
        if err != nil {
            return err
        }

        // 读取pid文件，如果存在数值，说明旧的daemon可能是存活的，下面进行判断。
        b := make([]byte, 8)
        n, err := f.Read(b)
        if err != nil && err != io.EOF {
            return err
        }

        //旧的daemon可能是存活的，因此要判断进程是否存活。
        if n > 0 {
            pid, err := strconv.ParseUint(string(b[:n]), 10, 64)
            if err != nil {
                return err
            }
            //如果旧daemon进程是存活的，那么设置remote结构体的daemonPid=pid文件中读取的值，并且返回。
            if utils.IsProcessAlive(int(pid)) {
                logrus.Infof("libcontainerd: previous instance of containerd still alive (%d)", pid)
                r.daemonPid = int(pid)
                return nil
            }
        }

        // 如果pid文件没有值，或者pid值代表的进程已经不是存活状态，则返回到pid文件，将seek标记到0位置，准备重写pid值。
        _, err = f.Seek(0, os.SEEK_SET)
        if err != nil {
            return err
        }

        // 把pid文件长度变为0，即清空pid文件。至此都是在创建docker-containerd进程的pid文件。
        err = f.Truncate(0)
        if err != nil {
            return err
        }

        // 设置docker-containerd进程使用的args参数。
        args := []string{
            "-l", fmt.Sprintf("unix://%s", r.rpcAddr),                           //docker-containerd进程的rpc调用地址，取值：/var/run/docker/libcontianerd/docker-containerd.sock
            "--shim", "docker-containerd-shim",                                  //docker-containerd进程在启动后会创建docker-containerd-shim进程，该进程实现runC，实现容器创建，容器周期管理。
            "--metrics-interval=0",
            "--start-timeout", "2m",                                             //docker-containerd进程启动超时时间，2m。
            "--state-dir", filepath.Join(r.stateDir, containerdStateDir),        //docker-containerd的根目录，取值：/var/run/docker/libcontainerd/containerd
        }

        //在上节中remote的runtime参数被设置为"docker-runc"，因此，执行下列语句，即添加args："--runtime docker-runc"，表示使用runC来创建容器，管理容器生命周期。
        if r.runtime != "" {
            args = append(args, "--runtime")
            args = append(args, r.runtime)
        }

        //是否支持debug模式
        if r.debugLog {
            args = append(args, "--debug")
        }

        //默认runtimeArgs为空
        if len(r.runtimeArgs) > 0 {
            for _, v := range r.runtimeArgs {
                args = append(args, "--runtime-args")
                args = append(args, v)
            }
            logrus.Debugf("libcontainerd: runContainerdDaemon: runtimeArgs: %s", args)
        }

        //至此，默认的args包括：
        // -l=unix:///var/run/docker/libcontianerd/docker-containerd.sock
        // --shim=docker-containerd-shim
        // --metrics-interval=0
        // --start-timeout=2m
        // --state-dir=/var/run/docker/libcontainerd/containerd
        // --runtime docker-runc

        //创建命令，即创建Cmd结构体，包含：命令名称（containerdBinary="docker-containerd"），参数（args）。为后面执行docker-containerd命令做准备。
        cmd := exec.Command(containerdBinary, args...)

        // 重定向containerd logs到docker logs
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stderr
        cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true, Pdeathsig: syscall.SIGKILL}  //dockerd是docker-containerd的父进程，这里允许父进程dockerd死了以后给子进程docker-containerd发送SIGKILL信号。
        cmd.Env = nil
        // clear the NOTIFY_SOCKET from the env when starting containerd
        for _, e := range os.Environ() {
            if !strings.HasPrefix(e, "NOTIFY_SOCKET") {
                cmd.Env = append(cmd.Env, e)
            }
        }

        //执行docker-containerd命令，创建docker-containerd进程。
        if err := cmd.Start(); err != nil {
            return err
        }

        logrus.Infof("libcontainerd: new containerd process, pid: %d", cmd.Process.Pid)
        if err := setOOMScore(cmd.Process.Pid, r.oomScore); err != nil {
            utils.KillProcess(cmd.Process.Pid)
            return err
        }
        if _, err := f.WriteString(fmt.Sprintf("%d", cmd.Process.Pid)); err != nil {
            utils.KillProcess(cmd.Process.Pid)
            return err
        }

        r.daemonWaitCh = make(chan struct{})
        go func() {
            cmd.Wait()
            close(r.daemonWaitCh)
        }() // Reap our child when needed
        r.daemonPid = cmd.Process.Pid
        return nil
    }

1. cmd.Start()

含义：

    执行docker-containerd命令。

路径：

    src/os/exec/exec.go

定义：

    // 启动docker-containerd进程
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

        //设置stdin、stdout、stderr文件描述符到childFiles中。
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
        //启动docker-containerd进程
        c.Process, err = os.StartProcess(c.Path, c.argv(), &os.ProcAttr{
            Dir:   c.Dir,
            Files: c.childFiles,
            Env:   c.envv(),
            Sys:   c.SysProcAttr,    //包含可以处理SIGKILL信号
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
[[返回docker-daemon.md]](./docker-daemon.md) 

