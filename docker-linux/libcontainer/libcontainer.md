libcontainer原理
================================================================
## 简介
docker早期使用LXC实现容器创建和管理，后来将底层实现抽象为libcontainer接口。libcontainer实现了namespace、cgroups、Rootfs、capabilities、进程运行的环境变量等配置和管理。

## libcontainer特性
1. 文件系统

    docker文件系统分为两层：bootfs和rootfs。
    bootfs包含了bootloader和linux内核。用户是不能对这层作任何修改的。在内核启动之后，bootfs实际上会被unmount掉。rootfs包含了一般系统上的常见目录结构，类似于/dev、/proc、/sys等等以及一些基本的文件和命令。
    docker的文件系统是分层的，它的rootfs在mount之后会转为只读模式。docker在它上面添加一个新的文件系统，来达成它的只读。
    如果用户想要修改底层只读层上的文件，这个文件就会被先拷贝到上层，修改后驻留在上层，并屏蔽原有的下层文件。
    最上层可以添加可读写的容器，用户也可以用docker commit命令将一个容器压制为image，供后续使用。

2. 资源管理

    libcontainer支持原生cgroups资源管理。容器中运行的第一个进程init，必须在初始化开始前放置到指定的cgroup目录中，这样就能防止初始化完成后运行的其他用户指令逃逸出cgroups的控制。父子进程的同步则通过管道来完成。

3. 容器安全配置

    libcontainer支持selinux、apparmor、capabilities等。

4. 创建容器

    在容器创建过程中，父进程与容器的init进程通过管道进行同步通信。当init启动时，它会接收管道信息，完成初始化，建立uid/gid映射，并把新进程放进新建的cgroup，最后接收父进程的EOF，结束管道。

    如果容器打开了伪终端，就会通过dup2把console作为容器的输入输出（STDIN、STDOUT、STDERR）对象。
    除此之外，以下四个文件也会在容器运行时自动生成。
    /etc/hosts
    /etc/resolv.conf
    /etc/hostname
    /etc/localtime

5. 容器热迁移

    使用内核CRIU技术实现容器热迁移，CRIU是checkpoint/restore in userspace。检查点保存和恢复技术，最初在用户态中，2012年被合并到内核中。热迁移可以保存容器的状态，以便在其他机器上恢复容器。其中容器状态一般包括：保存进程树，保存进程的资源，隔离进程资源相关联的附加程序。

## 使用nsinit创建容器
使用nsinit可以创建libcontainer接口的容器。docker虽然没有调用nsinit命令，但是实现原理差不多。

可以参考 [https://github.com/docker/libcontainer/blob/master/nsinit/README.md](https://github.com/docker/libcontainer/blob/master/nsinit/README.md)

1. 下载libcontainer项目

     # cd ~
     # git clone https://github.com/docker/libcontainer.git

     //添加libcontainer/vendor到GOPATH中
     # export GOPATH=$GOPATH:/root/libcontainer/vendor

2. 编译nsinit

     # cd libcontainer/nsinit
     # go get
     # make

3. 准备rootfs

     我们使用busybox作为rootfs
     # mkdir /busybox
     # curl -sSL 'https://github.com/jpetazzo/docker-busybox/blob/master/tarmaker-buildroot/rootfs.tar' | tar -xC /busybox
     如果curl失败，可以直接下载 https://github.com/jpetazzo/docker-busybox/blob/master/tarmaker-buildroot/rootfs.tar然后解压到/busybox中即可。

4. 准备container.json文件

     container.json文件是容器的配置文件。
     # cp libcontainer/sample_configs/minimal.json /busybox/container.json
     # cd /busybox

    container.json文件内容：
    {
        "no_pivot_root": false,                            //表示用rootfs作为文件系统挂载点，不单独设置pivot_root。pivot_root用来改变进程根目录，如果rootfs是基于ramfs（不支持pivot_root），那么会在mount时使用MS_MOVE标志位加上chroot来顶替。
        "parent_death_signal": 0,                          //表示当容器父进程销毁时发送给容器进程的信号。
        "pivot_dir": "",                                   //在容器root目录中指定一个目录作为容器文件系统挂载点目录。
        "rootfs": "/root/libcontainer",                    //容器根目录位置。
        "readonlyfs": false,                               //设置容器根目录为只读
        "mounts": [                                        //设置额外的挂载，填充的信息包括原路径，容器内目的路径，文件系统类型，挂载标识位，挂载的数据大小和权限，最后设定共享挂载还是非共享挂载
            {
                "source": "shm",
                "destination": "/dev/shm",
                "device": "tmpfs",
                "flags": 14,
                "data": "mode=1777,size=65536k",
                "relabel": ""
            },
            {
                "source": "mqueue",
                "destination": "/dev/mqueue",
                "device": "mqueue",
                "flags": 14,
                "data": "",
                "relabel": ""
            },
            {
                "source": "sysfs",
                "destination": "/sys",
                "device": "sysfs",
                "flags": 15,
                "data": "",
                "relabel": ""
            }
        ],
        "devices": [                                       //设置在容器启动时要创建的设备，填充的信息包括设备类型、容器内设备路径、设备块号（major、minor）、cgroup文件权限、用户编号、用户组编号。
            {
                "type": 99,
                "path": "/dev/fuse",
                "major": 10,
                "minor": 229,
                "permissions": "rwm",
                "file_mode": 0,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/null",
                "major": 1,
                "minor": 3,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/zero",
                "major": 1,
                "minor": 5,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/full",
                "major": 1,
                "minor": 7,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/tty",
                "major": 5,
                "minor": 0,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/urandom",
                "major": 1,
                "minor": 9,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "type": 99,
                "path": "/dev/random",
                "major": 1,
                "minor": 8,
                "permissions": "rwm",
                "file_mode": 438,
                "uid": 0,
                "gid": 0
            }
        ],
        "mount_label": "",                                 //设定共享挂载还是非共享挂载
        "hostname": "nsinit",                              //设定主机名
        "namespaces": [                                    //设定要加入的namespace，每个不同种类的namespace都可以指定，默认与父进程在同一个namespace中。
            {
                "type": "NEWNS",
                "path": ""
            },
            {
                "type": "NEWUTS",
                "path": ""
            },
            {
                "type": "NEWIPC",
                "path": ""
            },
            {
                "type": "NEWPID",
                "path": ""
            },
            {
                "type": "NEWNET",
                "path": ""
            }
        ],
        "capabilities": [                                  //设定在容器内进程拥有的capabilities权限，所有没加入此配置项的capabilities会被移除，即容器内进程失去该权限。
            "CHOWN",
            "DAC_OVERRIDE",
            "FSETID",
            "FOWNER",
            "MKNOD",
            "NET_RAW",
            "SETGID",
            "SETUID",
            "SETFCAP",
            "SETPCAP",
            "NET_BIND_SERVICE",
            "SYS_CHROOT",
            "KILL",
            "AUDIT_WRITE"
        ],
        "networks": [                                      //初始化容器的网络配置，包括类型（loopback、veth）、名称、网桥、物理地址、IPV4地址及网关、IPV6地址及网关、mtu、传输缓冲长度txqueuelen、宿主机设备名称。
            {
                "type": "loopback",
                "name": "",
                "bridge": "",
                "mac_address": "",
                "address": "127.0.0.1/0",
                "gateway": "localhost",
                "ipv6_address": "",
                "ipv6_gateway": "",
                "mtu": 0,
                "txqueuelen": 0,
                "host_interface_name": ""
            }
        ],
        "routes": null,                                    //配置路由表
        "cgroups": {                                       //配置cgroups资源限制参数，使用的参数不多，注意包括允许的设备列表、内存、交换区用量、CPU用量、块设备访问优先级、应用启停等。
            "name": "libcontainer",
            "parent": "nsinit",
            "allow_all_devices": false,
            "allowed_devices": [
                {
                    "type": 99,
                    "path": "",
                    "major": -1,
                    "minor": -1,
                    "permissions": "m",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 98,
                    "path": "",
                    "major": -1,
                    "minor": -1,
                    "permissions": "m",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/console",
                    "major": 5,
                    "minor": 1,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/tty0",
                    "major": 4,
                    "minor": 0,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/tty1",
                    "major": 4,
                    "minor": 1,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "",
                    "major": 136,
                    "minor": -1,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "",
                    "major": 5,
                    "minor": 2,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "",
                    "major": 10,
                    "minor": 200,
                    "permissions": "rwm",
                    "file_mode": 0,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/null",
                    "major": 1,
                    "minor": 3,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/zero",
                    "major": 1,
                    "minor": 5,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/full",
                    "major": 1,
                    "minor": 7,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/tty",
                    "major": 5,
                    "minor": 0,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/urandom",
                    "major": 1,
                    "minor": 9,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                },
                {
                    "type": 99,
                    "path": "/dev/random",
                    "major": 1,
                    "minor": 8,
                    "permissions": "rwm",
                    "file_mode": 438,
                    "uid": 0,
                    "gid": 0
                }
            ],
            "memory": 0,
            "memory_reservation": 0,
            "memory_swap": 0,
            "cpu_shares": 0,
            "cpu_quota": 0,
            "cpu_period": 0,
            "cpuset_cpus": "",
            "cpuset_mems": "",
            "blkio_weight": 0,
            "freezer": "",
            "slice": ""
        },
        "apparmor_profile": "",                               //配置用于selinux的apparmor文件
        "process_label": "",                                  //用于selinux的配置
        "rlimits": [                                          //最大文件打开数量，默认与父进程相同。
            {
                "type": 7,
                "hard": 1024,
                "soft": 1024
            }
        ],
        "additional_groups": null,                            //设定gid，添加同一用户下的其他组
        "uid_mappings": null,                                 //用于User namespace的uid映射。
        "gid_mappings": null,                                 //用于User namespace的gid映射。
        "mask_paths": [                                       //配置不使用的设备，通过绑定/dev/null进行路径掩盖
            "/proc/kcore"
        ],
        "readonly_paths": [                                   //在容器内设定只读部分的文件路径。
            "/proc/sys",
            "/proc/sysrq-trigger",
            "/proc/irq",
            "/proc/bus"
        ]
    }

5. 启动容器并执行命令

     # nsinit exec --tty --config container.json /bin/bash

     nsinit还有很多其他命令：
     config：根据内置的默认参数和命令行参数，生成容器的标准配置文件。
     init：这是内置参数，用户不能直接使用。这个命令在容器内部执行，为容器进行namespace初始化，并在完成初始化后执行用户指令。所以在代码中，运行nsinit exec后，传入到容器中运行的实际上是nsinit init，把用户指令作为配置项传入。
     oom：展示容器的内存超限通知。
     pause/unpause：暂停/恢复容器中的进程。
     stats：显示容器中的统计信息，主要包括cgroup和网络。
     checkpoint：保存容器的检查点快照并结束容器进程。需要填--image-path参数，后面是检查点保存的快照文件路径。完整的命令示例如下：
          nsinit checkpoint --image-path=/tmp/criu
     restore：从容器检查点快照恢复容器进程的运行。


6. 查看状态
     # vi state.json

## libcontainer实现原理
在docker中，对容器管理的模块为execdriver，docker支持的容器管理方式有两种，一种是最初支持的LXC方式，另一种是native，即使用libcontainer进行容器管理。docker daemon在启动过程中会对execdriver进行初始化，会根据驱动的名称选择使用的容器管理方式，默认使用native方式。

libcontainer接口定义了一系列容器管理的操作：Factory（容器创建）、Container（容器生命周期管理）、Process（进程生命周期管理）。

1. Factory对象

    Factory对象提供了容器创建和初始化的抽象接口，包含4个方法：
        Create()：创建一个Container类，包含容器id、状态目录（在root目录下创建的以id命名的文件夹，存state.json容器状态文件）、容器配置参数、初始化路径和参数，以及管理cgroup的方式（包含直接通过文件操作管理和systemd管理两个选择，默认选cgroup文件系统管理）。
        Load()：当创建的id已经存在时，即已经create过，存在id文件目录，就会从id目录下直接读取state.json来载入容器。
        Type()：返回容器管理的类型，目前可能返回的有libcontainer和lxc，为未来支持更多容器接口做准备。
        StartInitialization()：容器内初始化函数。
        这部分代码是在容器内部执行的，当容器创建时，如果New不加任何参数，默认在容器进程中运行的第一条命令介绍nsinit init。在execdriver的初始化中，会向reexec注册初始化器，命名为native，然后在创建libcontainer以后把native作为执行参数传递到容器中执行，这个初始化器创建的libcontainer就是没有参数的。
        传入的参数是一个管道文件描述符，为了保证在初始化过程中，父子进程间状态同步和配置信息传递而建立。
        不管是纯粹新建的容器还是已经创建的容器执行新的命令，都是从这个入口做初始化。
        第一步，通过管道获取配置信息。
        第二步，从配置信息中获取环境变量并设置为容器内环境变量。
        若是已经存在的容器执行新命令，则只需要配置cgroup、namespace的capabilities以及AppArmor等信息，最后执行命令。
        若是纯粹新建的容器，则还需要初始化网络、路由、namespace、主机名、配置只读路径等等，最后执行命令。
    
    至此，容器就已经创建和初始化完毕了。

2. Container对象

    Container对象主要包含容器配置、控制、状态显示等功能，是对不同平台容器功能的抽象。每一个Container进程内部都是线程安全的。因为Container有可能被外部进程销毁，所以每个方法都会对容器是否存在进行检测。
    1.ID()：显示Container的ID，在Factory对象中已经说过，ID具有唯一性。
    2.Status()：返回容器内进程是运行状态还是停止状态。通过执行"SIG=0"的KILL命令对进程是否存在进行检测。
    3.State()：返回容器的状态，包括容器ID、配置信息、初始进程ID、进程启动时间、cgroup文件路径、namespace路径。通过调用Status()判断进程是否存在。
    4.Config()：返回容器的配置信息，可在"配置参数解析"部分查看有哪些方面的配置信息。
    5.Processes()：返回cgroup文件cgroup.procs中的值，cgroup.procs文件会罗列所有在该cgroup中的线程组ID（若有线程创建了子线程，则子线程的PID不包括在内）。由于容器不断在运行，所以返回的结果并不能保证完全存活，除非容器处于“PAUSED”状态。
    6.Stats()：返回容器的统计信息，包括容器的cgroups中的统计以及网卡设备的统计信息。cgroups中主要统计了cpu、memory和blkio这三个子系统的统计内容。网卡设备的统计则通过读取系统中，网络网卡文件的统计信息文件/sys/calss/net/<EthInterface>/statistics来实现。
    7.Set()：设置容器cgroup各子系统的文件路径。因为cgroups的配置是进程运行时也会生效的，所以我们可以通过这个方法在容器运行时改变cgroups文件从而改变资源分配。
    8.Start()：构建ParentProcess对象，用于处理启动容器进程的所有初始化工作，并作为父进程与新创建的子进程（容器）进行初始化通信。传入的Process对象可以帮助我们追踪进程的生命周期，Process对象将在后文详细介绍。
        启动的过程首先会调用Status()方法的具体实现得知进程是否存活。
        创建一个管道为后期父子进程通信做准备。
        配置子进程cmd命令模板，配置参数的值就是从factory.Create()传入进来的，包括命令执行的工作目录、命令参数、输入输出、根目录、子进程管道以及KILL信号的值。
        根据容器进程是否存在确定是在已有容器中执行命令还是创建新的容器执行命令。若存在，则把配置的命令构建成一个exec.Cmd对象、cgroup路径、父子进程管道及配置保留到ParentProcess对象中；若不存在，则创建容器进程及相应namespace，目前对user namespace有了一定的支持，若配置时加入user namespace，会针对配置项进行映射。默认映射到宿主机的root用户，最后同样构建出相应的配置内容保留到ParentProcess对象中。通过在cmd.Env写入环境变量_LIBCONTAINER_INITTYPE来告诉容器进程采用的哪种方式启动。
        执行ParentProcess中构建的exec.Cmd内容，即执行ParentProcess.start()，具体的执行过程在Process部分介绍。
        最后如果是新建的容器进程，还会执行状态更新函数，把state.json的内容刷新。
    9.Destroy()：首先使用cgroup的freezer子系统暂停所有运行的进程，然后给所有进程发送SIGKILL信号（如果没有使用pid namespace就不会进程处理）。最后把cgroup及其子系统卸载，删除cgroup文件夹。
    10.Pause()：使用cgroup的freezer子系统暂停所有运行的进程。
    11.Resume()：使用cgroup的freezer子系统恢复所有运行的进程。
    12.NotifyOOM()：为容器内存使用超界提供只读的通道，通过向cgroup.event_control写入eventfd（用作线程间通信的消息队列）和cgroup.oom_control（用于决定内存使用超限后的处理方式）来实现。
    13.Checkpoint()：保存容器进程检查点快照，为容器热迁移做准备。通过使用CRIU的SWRK模式来实现，这种模式是CRIU另外两种模式CLI和RPC的结合体，允许用户需要的时候使用命令行工具一样运行CRIU，并接受用户远程调用的请求，即传入的热迁移检查点保存请求，传入文件形式以google的protobuf协议保存。
    14.Restore()：恢复检查点快照并运行，完成容器热迁移。同样通过CRIU的SWRK模式实现，恢复的时候可以传入配置文件设置恢复挂载点、网络等配置信息。
    至此，Container对象中所有函数及相关功能已经介绍完毕，包含了容器生命周期的全部过程。

    TIPs:Docker初始化通信--管道
    libcontainer创建容器进程时需要做初始化工作，此时就涉及到使用了namespace隔离后的两个进程间通信。我们把负责创建容器的进程称为父进程，容器进程称为子进程。父进程clone出子进程以后，依旧是共享内存的。但是如何让子进程知道内存中写入了新数据依旧是一个问题，一般有四种方法：
        发送信号通知（signal）
        对内存轮询访问（poll memory）
        sockets通信（sockets）
        文件和文件描述符（files and file-descriptors）
    对于signal而言，本身包含的信息有限，需要额外记录，namespace带来的上下文变化使其不易理解，并不是最佳选择。显然通过轮询内存的方式来沟通是一个非常低效的做法。另外，因为docker会加入network namespace，实际上初始时网络栈也是完全隔离的，所以socket方式并不可行。
    docker最终选择的方式是打开的可读可写文件描述符--管道。
    Linux中，通过pipe（int fd[2]）系统调用就可以创建管道，参数是一个包含两个整型的数组。调用完成后，在fd[1]端写入的数据，就可以从fd[0]端读取。
    //需要加入头文件：
    #include <unistd.h>
    //全局变量
    int fd[2];
    //在父进程中进行初始化
    pipe(fd);
    //关闭管道文件描述符
    close(checkpoint[1]);
    调用pipe函数后，创建的子进程会内嵌这个打开的文件描述符，对fd[1]写入数据后可以在fd[0]端读取。通过管道，父子进程之间就可以通信。通信完毕的奥秘就在于EOF信号的传递。大家都知道，当打开的文件描述符都关闭时，才能读到EOF信号，所以libcontainer中父进程先关闭自己这一端的管道，然后等待子进程关闭另一端的管道文件描述符，传来EOF表示子进程已经完成了初始化的过程。

3. Process对象

    Process主要分为两类，一类在源码中就叫Process，用于容器内进程的配置和IO的管理；另一类在源码中叫ParentProcess，负责处理容器启动工作，与Container对象直接进行接触，启动完成后作为Process的一部分，执行等待、发信号、获得pid等管理工作。

    ParentProcess对象，主要包含以下六个函数，而根据“需要新建容器”和“在已经存在的容器中执行”的不同方式，具体的实现也有所不同。
        已有容器中执行命令
            1.pid()：启动容器进程后通过管道从容器进程中获得，因为容器已经存在，与docker daemon在不同的pid namespace中，从进程所在的namespace获得的进程号才有意义。
            2.start()：初始化容器中的执行进程。在已有容器中执行命令一般由docker exec调用，在execdriver包中，该函数会读取配置文件，使用setns()加入到相应的namespace，然后通过clone()在该namespace中生成一个子进程，并把子进程通过管道传递出去，使用setns()以后并没有pid namespace，所以还需要通过加上clone()系统调用。
            开始执行过程，首先会运行C代码，通过管道获得进程pid，最后等待C代码执行完毕。
            通过获得的pid把cmd中的Process替换成新生成的子进程。
            把子进程加入cgroup中。
            通过管道传配置文件给子进程。
            等待初始化完成或出错返回，结束。
        新建容器执行命令：
            1.pid()：启动容器进程后通过exec.Cmd自带的pid()函数即可获得。
            2.start()：初始化及执行容器命令。
                开始运行进程。
                把进程pid加入到cgroup中管理。
                初始化容器网络。
                通过管道发送配置文件给子进程。
                等待初始化完成或出错返回，结束。
        实现方式类型是一些函数：
            terminate(): 发送SIGKILL信号结束进程。
            startTime(): 获取进程的启动时间。
            signal()：发送信号给进程。
            wait()：等待程序执行结束，返回结束的程序状态。

    Process对象，主要描述了容器内进程的配置以及IO。包括参数Args，环境变量Env，用户User（用于uid、gid映射），工作目录cwd，标准输入输出及错误输入，控制终端路径consolePath，容器权限capabilities以及上述提到的ParentProcess对象ops。



