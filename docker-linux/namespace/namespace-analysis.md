每种namespace的详细分析
===================================================
## 简介
这里介绍Go语言中namespace隔离的实现方法。我们需要下载一个库：go get github.com/Sirupsen/logrus。

## 1. uts namespace
含义：

    uts namespace提供了主机名和域名的隔离，这样每个容器就可以拥有独立的主机名和域名，在网络上可以被视为一个独立的节点而非宿主机上的一个进程。

示例：

（1）先创建一个uts隔离的新进程：

    package main

    import (
            "os"
            "os/exec"
            "syscall"

            "github.com/Sirupsen/logrus"
    )

    func main() {
            if len(os.Args) < 2 {
                    logrus.Errorf("missing commands")
                    return
            }
            switch os.Args[1] {
            case "uts":
                    uts()
            default:
                    logrus.Errorf("wrong command")
                    return
            }
    }
    func uts() {
            logrus.Infof("Running %v", os.Args[2:])
            cmd := exec.Command(os.Args[2], os.Args[3:]...)
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr

            cmd.SysProcAttr = &syscall.SysProcAttr{
                    Cloneflags: syscall.CLONE_NEWUTS,
            }
            check(cmd.Run())
    }
    func check(err error) {
            if err != nil {
                    logrus.Errorln(err)
            }
    }

（2）在Linux中执行：

    # go run main.go run sh
    INFO[0000] Running [sh]
    sh-4.2#                            //此时已经进入了新进程的sh

（3）此时，在新进程中执行sh命令，由于指定CLONE_NEWUTS，此时已经与之前的进程不在同一个uts namespace中了。可以在新sh和旧sh中分别执行ls -l /proc/$$/ns进行查看：

新进程sh：

    sh-4.2# ls -l /proc/$$/ns
    total 0
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 ipc -> ipc:[4026531839]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 net -> net:[4026531956]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 user -> user:[4026531837]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:33 uts -> uts:[4026532465]

旧进程sh：

    [root@localhost ~]# ls -l /proc/$$/ns
    total 0
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 ipc -> ipc:[4026531839]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 net -> net:[4026531956]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 user -> user:[4026531837]
    lrwxrwxrwx. 1 root root 0 Sep 16 18:35 uts -> uts:[4026531838]

结论：从上面结果可以看出只有uts namespace的编号发生了变化，新进程拥有了一个新的uts namespace，即和旧进程的uts namespace进行了隔离。

（4）下面验证新进程中修改hostname不会影响旧进程的hostname

为了在启动sh的同时修改hostname，下面将uts()函数拆分成uts()和hostname()，这样可以保证修改hostname的时候已经在新的namespace中了，避免修改宿主机的hostname。

    package main

    import (
            "os"
            "os/exec"
            "syscall"

            "github.com/Sirupsen/logrus"
    )

    func main() {
            if len(os.Args) < 2 {
                    logrus.Errorf("missing commands")
                    return
            }
            switch os.Args[1] {
            case "uts":
                    uts()
            case "hostname":
                    hostname()
            default:
                    logrus.Errorf("wrong command")
                    return
            }
    }

    func uts() {
            logrus.Info("Setting up...")
            cmd := exec.Command("/proc/self/exe", append([]string{"hostname"}, os.Args[2:]...)...)  // /proc/self/exe是当前正在执行的命令，即go run main.go
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
            cmd.SysProcAttr = &syscall.SysProcAttr{
                    Cloneflags: syscall.CLONE_NEWUTS,
            }
            check(cmd.Run())
    }

    func hostname() {
            logrus.Infof("Running %v", os.Args[2:])
            cmd := exec.Command(os.Args[2], os.Args[3:]...)
            cmd.Stdin = os.Stdin
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
            check(syscall.Sethostname([]byte("newhostname")))
            check(cmd.Run())
    }

    func check(err error) {
            if err != nil {
                    logrus.Errorln(err)
            }
    }

查看：

    [root@localhost demo]# go run main.go uts sh
    INFO[0000] Setting up...                                
    INFO[0000] Running [sh]                                 
    sh-4.2# hostname                                    //新进程sh中执行hostname
    newhostname
    sh-4.2# exit
    exit
    [root@localhost demo]# hostname                     //旧进程sh中执行hostname
    localhost.localdomain
    [root@localhost demo]# 

结论：

    从上面的结果可以看出，新进程的hostname确实改变了，和旧进程的hostname实现了uts namespace隔离。


_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 

