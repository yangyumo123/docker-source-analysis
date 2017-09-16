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


## 2. ipc namespace
含义：

    实现信号量、消息队列、共享内存的隔离。Docker本身使用tcp通信，没有使用ipc。

## 3. pid namespace
含义：

    进程号隔离。

示例：

修改上面的程序，在Cloneflags的后面添加syscall.CLONE_NEWPID参数。

    func uts(){
        ...
        cmd.SysProcAttr = &syscall.SysProcAttr{
            Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID
        }
        ...
    }

执行：

    [root@localhost demo]# go run main.go uts sh
    INFO[0000] Setting up...                                
    INFO[0000] Running [sh]                                 
    sh-4.2# ps
       PID TTY          TIME CMD
      2125 pts/0    00:00:00 bash
      2592 pts/0    00:00:00 go
      2612 pts/0    00:00:00 main
      2615 pts/0    00:00:00 exe
      2618 pts/0    00:00:00 sh
      2619 pts/0    00:00:00 ps
    sh-4.2# 

结论：发现pid号并没有隔离，还是在原namespace中。因为ps指令总是查看/proc，如果要进行隔离，需要修改root目录。

下面介绍修改root目录的方法。

首先，获得一个linux文件系统，可以从docker image中导出一个，例如：从image：busybox:latest中导出一个文件系统.

    [root@localhost demo]# docker pull busybox                                                                            //下载的image：docker.io/busybox:latest
    [root@localhost demo]# docker run -d docker.io/busybox:latest top -b                                                  //获得container id
    fb8ee35d83240d56288155d013ec345ee8a4993a913c535d9a710c557080b408
    [root@localhost demo]# docker export -o busybox.tar fb8ee35d83240d56288155d013ec345ee8a4993a913c535d9a710c557080b408  //打包
    [root@localhost demo]# mkdir busybox
    [root@localhost demo]# tar -xf busybox.tar -C busybox/                                                                //解包
    [root@localhost demo]# ll
    total 16
    drwxr-xr-x. 2 root  root  12288 Aug 23 08:00 bin
    drwxr-xr-x. 4 adm   sys      43 Sep 16 20:00 dev
    drwxr-xr-x. 2 root  root    124 Sep 16 20:00 etc
    drwxr-xr-x. 2 65534 65534     6 Aug 23 08:00 home
    drwxr-xr-x. 2 root  root      6 Sep 16 20:00 proc
    drwxr-xr-x. 2 root  root      6 Aug 23 08:00 root
    drwxr-xr-x. 3 root  root     21 Sep 16 20:00 run
    drwxr-xr-x. 2 root  root      6 Sep 16 20:00 sys
    drwxrwxrwt. 2 root  root      6 Aug 23 08:00 tmp
    drwxr-xr-x. 3 root  root     18 Aug 23 08:00 usr
    drwxr-xr-x. 4 root  root     30 Aug 23 08:00 var

然后，修改hostname()函数：

    func child() {
        ...
        check(syscall.Sethostname([]byte("newhostname")))
        check(syscall.Chroot("/root/mygo/src/github.com/yangyumo123/demo/busybox"))
        check(os.Chdir("/"))
        // func Mount(source string, target string, fstype string, flags uintptr, data string) (err error)
        // 前三个参数分别是文件系统的名字，挂载到的路径，文件系统的类型
        check(syscall.Mount("proc", "proc", "proc", 0, ""))
        check(cmd.Run())
        check(syscall.Unmount("proc", 0))
    }

执行：

    [root@localhost demo]# go run main.go uts /bin/sh
    INFO[0000] Setting up...                                
    INFO[0000] Running [/bin/sh]                            
    / # /bin/ps
    PID   USER     TIME   COMMAND
        1 root       0:00 /proc/self/exe hostname /bin/sh    //新进程pid在pid namespace中编号从1开始。
        4 root       0:00 /bin/sh
        5 root       0:00 /bin/ps
    / # 

结论：从上面结果来看，新进程的pid namespace确实进行了隔离。

### 4. mnt namespace
含义：

    文件系统隔离。

示例：

如果在hostname()中再挂载一个tmpfs：

    check(syscall.Mount("proc","proc","proc",0,""))
    check(syscall.Mount("tempdir","temp","tmpfs",0,""))
    check(cmd.Run())
    check(syscall.Unmount("proc",0))
    check(syscall.Unmount("temp",0))

执行：

    [root@localhost demo]# go run main.go uts /bin/sh
    INFO[0000] Setting up...                                
    INFO[0000] Running [/bin/sh]                            
    / # /bin/mount
    ...
    proc on /proc type proc (rw,relatime)
    tmp on /tmp type tmpfs (rw,seclabel,relatime)

    [root@localhost tmp]# ll /root/mygo/src/github.com/yangyumo123/demo/busybox/tmp   //在宿主机上查看，仍然能看到在新进程中挂载的abc文件，并没有被隔离。
    total 0
    -rw-r--r--. 1 root root 0 Sep 16 21:01 abc

结论：从上面的结果来看，新进程执行mount，在宿主机上仍然能看见mount的文件，并没有mnt隔离。原因是没有执行CLONE_NEWNS。修改一下代码，添加CLONE_NEWNS参数：

    func uts() {
        ...
        cmd.SysProcAttr = &syscall.SysProcAttr{
            Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,   //添加syscall.CLONE_NEWNS，进行mnt namespace隔离。
        }
        ...
    }

再执行：

    [root@localhost demo]# go run main.go uts /bin/sh
    INFO[0000] Setting up...                                
    INFO[0000] Running [/bin/sh]                            
    / # /bin/mount
    ...
    proc on /proc type proc (rw,relatime)
    tmp on /tmp type tmpfs (rw,seclabel,relatime)

    //在宿主机上执行
    [root@localhost tmp]# ll /root/mygo/src/github.com/yangyumo123/demo/busybox/tmp     //宿主机看不到新进程挂载的文件，因此，mnt namespace进行了隔离。
    total 0
    [root@localhost tmp]# 


_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 

