创建containerd daemon
==================================================
## 简介
前面介绍了启动过程，当设置了"--containerd"参数时，将创建并启动containerd daemon。

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

    创建containerd daemon。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

定义：

    func (r *remote) runContainerdDaemon() error {
        pidFilename := filepath.Join(r.stateDir, containerdPidFilename)
        f, err := os.OpenFile(pidFilename, os.O_RDWR|os.O_CREATE, 0600)
        defer f.Close()
        if err != nil {
            return err
        }

        // File exist, check if the daemon is alive
        b := make([]byte, 8)
        n, err := f.Read(b)
        if err != nil && err != io.EOF {
            return err
        }

        if n > 0 {
            pid, err := strconv.ParseUint(string(b[:n]), 10, 64)
            if err != nil {
                return err
            }
            if utils.IsProcessAlive(int(pid)) {
                logrus.Infof("libcontainerd: previous instance of containerd still alive (%d)", pid)
                r.daemonPid = int(pid)
                return nil
            }
        }

        // rewind the file
        _, err = f.Seek(0, os.SEEK_SET)
        if err != nil {
            return err
        }

        // Truncate it
        err = f.Truncate(0)
        if err != nil {
            return err
        }

        // Start a new instance
        args := []string{
            "-l", fmt.Sprintf("unix://%s", r.rpcAddr),
            "--shim", "docker-containerd-shim",
            "--metrics-interval=0",
            "--start-timeout", "2m",
            "--state-dir", filepath.Join(r.stateDir, containerdStateDir),
        }
        if r.runtime != "" {
            args = append(args, "--runtime")
            args = append(args, r.runtime)
        }
        if r.debugLog {
            args = append(args, "--debug")
        }
        if len(r.runtimeArgs) > 0 {
            for _, v := range r.runtimeArgs {
                args = append(args, "--runtime-args")
                args = append(args, v)
            }
            logrus.Debugf("libcontainerd: runContainerdDaemon: runtimeArgs: %s", args)
        }

        cmd := exec.Command(containerdBinary, args...)
        // redirect containerd logs to docker logs
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stderr
        cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true, Pdeathsig: syscall.SIGKILL}
        cmd.Env = nil
        // clear the NOTIFY_SOCKET from the env when starting containerd
        for _, e := range os.Environ() {
            if !strings.HasPrefix(e, "NOTIFY_SOCKET") {
                cmd.Env = append(cmd.Env, e)
            }
        }
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


_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 

