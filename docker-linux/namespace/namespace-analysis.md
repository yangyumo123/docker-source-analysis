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

    进程号隔离，即两个不同的namespace中的进程可以有相同的pid。内核为所有的pid namespace维护了一个树状结构，最顶层的是系统初始时创建的，称为root namespace。
    创建的新的pid namespace称为child namespace（子节点），而原先的pid namespace称为parent namespace（父节点）。父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点中的任何内容。
    pid namespace具有以下特性：
    * 每个pid namespace中的第一个进程的pid为1，就像Linux中的init进程一样拥有特权。
    * 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他namespace中的pid在这个namespace中没有任何意义。
    * 如果在新的pid namespace中重新挂载/proc文件系统，会发现其下只显示同属一个pid namespace中的其他进程。
    * 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。其中一个应用是：在外部监控Docker中运行的进程，即监控Docker daemon所在的pid namespace下所有进程及其子进程。

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

结论：从上面结果来看，新进程的pid namespace确实进行了隔离。此外，与其他namespace不同的是，为了实现一个稳定安全的容器，pid namespace还需要进行一些额外的工作才能确保其中的进程顺利运行。

（1）pid namespace中的init进程

新创建pid namespace时，默认启动的进程pid为1，即init进程，特权进程。init进程为所有进程的父进程，维护一张进程表，不断检查进程的状态，一旦有某个子进程因为程序错误成为了“孤儿”进程，init就负责回收资源并介绍这个子进程。

Docker启动时，第一个进程实现了进程监控和资源回收，它就是dockerinit。

（2）信号与init进程

内核为init进程赋予一个特权--信号屏蔽。如果init进程中没有写处理某个信号的代码逻辑，那么与init在同一个pid namespace中的进程（即使有超级权限）发送给该init进程的信号都会被屏蔽。这个作用是防止init进程被误杀。

但是，父节点中的进程发送的信号中有2个信号：SIGKILL（销毁进程）或SIGSTOP（暂停进程）不会被屏蔽，其他信号同样会被屏蔽。如果父节点中的进程发送SIGKILL或SIGSTOP，子节点的init会强制执行（无法通过代码捕获进行处理）。

一旦init进程被销毁，同一个pid namespace中的其他进程也会收到SIGKILL信号而被销毁。理论上，该pid namespace也会销毁，但是前面讲到，可以通过/proc/[pid]/ns/pid挂载的方式保留pid namespace。然而，保留下来的pid namespace无法通过setns()或clone()创建新进程，所以实际上没有什么用。这就是为什么Docker一旦启动就必须有进程在运行，不存在不包含任何进程的docker。

（3）挂载proc文件系统

前面实验已经说明了，重新挂载/proc文件系统，才能查看到pid号发生变化。

（4）unshare和setns

这两个API在pid namespace中有些需要注意的地方。

unshare允许用户在原来进程中建立namespace进行隔离，但是创建了pid namespace后，原先unshare调用者进程并不进入新的pid namespace，接下来创建的子进程才会进入新的pid namespace中，这个子进程就是新的pid namespace中的init进程。

setns也是一样，在创建pid namespace时，调用者进程不会进入新的pid namespace中，子进程才会进入。

为什么创建其他namespace时unshare和setns的调用者会直接进入新的namespace中，而唯独pid namespace不是呢？因为调用getpid()函数得到的pid是根据调用者所在的pid namespace而决定返回哪个pid，进入新的pid namespace会导致pid产生变化。而进程的pid一般被当做常量看待，如果pid变化，则会引起进程崩溃。



### 4. mnt namespace
含义：

    文件系统隔离。可以通过/proc/[pid]/mounts查看所有挂载在当前mnt namespace中的文件系统，还可以通过/proc/[pid]/mountstats查看mnt namespace中文件设备的统计信息等。

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

深入分析：

进程在创建mnt namespace时，会把当前的文件结构复制给新的mnt namespace。新mnt namespace中的所有mount操作都只影响自身的文件系统。这样做严格地实现了mnt隔离，但是某些情况可能并不适用。例如：父节点namespace中的进程挂载一张CD-ROM，这时子节点namespace拷贝的目录结构就无法自动挂载上这张CD-ROM，因为这种操作会影响到父节点的文件系统。

2006年引入了挂载传播（mount propagation）解决了这个问题，挂载传播定义了挂载对象（mount object）之间的关系，系统用这些关系决定任何挂载对象中的挂载事件如何传播到其他挂载对象中。

一个挂载状态可能有以下几种：

    共享挂载（shared）
    从属挂载（slave）
    共享/从属挂载（shared and slave）
    私有挂载（private）
    不可绑定挂载（unbindable）

传播事件的挂载对象称为共享挂载（例如，/lib。共享挂载克隆的挂载对象也是共享挂载）；接收传播事件的挂载对象称为从属挂载（例如，/bin。一般用于只读场景。从属挂载克隆的挂载对象也是从属挂载。）；既不传播也不接收传播事件的挂载对象称为私有挂载（例如，/proc。默认所有挂载都是私有的）；不可绑定挂载（例如，/root）即不可以复制。

设置共享挂载的命令：

    # mount --make-shared <mount-object>

设置从属挂载的命令：

    # mount --make-slave <mount-object>

将一个从属挂载对象设置为共享/从属挂载，命令：

    # mount --make-shared <slave-mount-object>

设置私有挂载的命令：

    # mount --make-private <mount-object>

设置不可绑定挂载的命令：

    # mount --make-unbindable <mount-object>

### 5. net namespace
含义：

    net namespace提供网络资源的隔离，包括：网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口等等。一个物理物理设备最多存在于一个net namespace中，但可以通过veth pair在不同的net namespace间创建通道，来通信。

    一般情况下，物理网络设备最初分配在root namespace中。但是，可以被转移给新创建的net namespace。当新创建的net namespace释放时，该net namespace中的物理设备也会返回到root namespace中。

    一般来说，我们不需要真正的网络隔离，我们需要在不同net namespace之间做通信。为此，容器的经典做法是创建一个veth pair，一端放在新的net namespace中，通常命名为eth0，另一端放在原先的net namespace中连接物理网络设备，再通过网桥把别的设备连接进来或者进行路由转发，以实现网络通信的目的。

    在建立veth pair之前，新旧net namespace之间如何通信？它们通过管道（pipe）。
    以dockerinit为例，docker daemon在宿主机上负责创建这个veth pair，通过netlink调用，把一端绑定到docker0网桥上，一端连接进新创建的net namespace进程中。建立过程中，docker daemon和dockerinit就通过pipe进行通信，当docker daemon完成veth pair的创建之前，dockerinit在管道的另一端循环等待，直到管道另一端传来docker daemon关于veth pair的信息，并关闭管道。dockerinit才结束等待，并把它的etho启动起来。

示例：

    可以在创建的时候指定参数：CLONE_NEWNET。也可以使用ip命令创建net namespace。下面使用ip命令来模拟创建net namespace的过程。
    1.创建名称为my_ns的net namespace
        # ip netns add my_ns
        此时，ip程序做了两件事：创建一个默认的回环设备lo，并在/var/run/netns目录下绑定一个挂载点，保证新创建的net namespace中即使没有进程也不会被释放。
    2.查看新创建的net namespace中的设备
        # ip netns exec my_ns ip link list
        1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        此时，lo是DOWN状态，还没有启动。

        # ip netns exec my_ns ping 127.0.0.1
        connect: Network is unreachable
        此时，ping本地是不通的。
    3.启动lo
        # ip netns exec my_ns ip link set dev lo up
        # ip netns exec my_ns ip link list
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        此时，lo是UP状态，启动了。
        # ip netns exec my_ns ping 127.0.0.1
        PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
        64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.065 ms
        64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.080 ms
        ...
        此时，ping本地是通的。

    4.创建veth pair
        # ip link add veth0 type veth peer name veth1                    //在旧的net namespace中创建veth0，对端veth1
        # ip link set veth1 netns my_ns                                  //把veth1设置到新的net namespace中
        # ip netns exec my_ns ifconfig veth1 10.1.1.1/24 up              //为新的net namespace的veth1配置网络地址，并启动
        # ifconfig veth0 10.1.1.2/24                                     //为旧的net namespace的veth0配置网络地址，
    5.检查2个net namespace的veth pair是否相通
        # ping 10.1.1.1                                                  //从旧的net namespace中ping新的net namespace
        PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
        64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.097 ms
        64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.097 ms

        # ip netns exec my_ns ping 10.1.1.2                              //从新的net namespace中ping旧的net namespace
        PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
        64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.087 ms
        64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.071 ms
    6.还可以查看route和iptables
        # ip netns exec my_ns route
        Kernel IP routing table
        Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
        10.1.1.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1

        # ip netns exec my_ns iptables -nL
        Chain INPUT (policy ACCEPT)
        target     prot opt source               destination         

        Chain FORWARD (policy ACCEPT)
        target     prot opt source               destination         

        Chain OUTPUT (policy ACCEPT)
        target     prot opt source               destination 

    7.删除新的net namespace
        # ip netns delete my_ns
        会卸载之前的挂载目录。如果net namespace中还有进程在运行，则等进程结束后销毁。

_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 

