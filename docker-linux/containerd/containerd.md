containerd原理
=========================================================
## 简介
Linux基金会于2015年6月成立了OCI（Open Container Initiative）组织，得到了谷歌、微软、亚马逊等云计算厂商的支持。Docker贡献出了runC，runC是按照OCF（Open Container Format）制定的一种具体实现。

实际上，docker从1.11将docker daemon进行了拆解，把容器创建、容器生命周期管理的部分拆解出来作为containerd进程。

containerd也是一个daemon，底层是通过runC来具体实现容器创建、容器生命周期管理的。

runC实际上是调用libcontainer来实现容器创建、容器生命周期管理的。

实际上，你可以不用docker daemon，而只用containerd来创建容器，这就是容器标准化的好处。下面将介绍使用containerd创建容器的方法：

## 1. 准备rootfs
不管是使用libcontainer还是使用containerd，都需要准备rootfs。有很多操作系统都可以作为rootfs，这里使用busybox作为rootfs。

    #cd ~
    #mkdir my_container
    #cd my_container
    #mkdir rootfs
    #docker export $(docker create busybox) | tar -C rootfs -xvf -

    此时rootfs如下：
    [root@localhost my_container]# ls -l rootfs/
    total 16
    drwxr-xr-x. 2 root  root  12288 Aug 23 08:00 bin
    drwxr-xr-x. 4 adm   sys      43 Sep 19 20:13 dev
    drwxr-xr-x. 2 root  root    124 Sep 19 20:13 etc
    drwxr-xr-x. 2 65534 65534     6 Aug 23 08:00 home
    drwxr-xr-x. 2 root  root      6 Sep 19 20:13 proc
    drwxr-xr-x. 2 root  root      6 Aug 23 08:00 root
    drwxr-xr-x. 2 root  root      6 Sep 19 20:13 sys
    drwxrwxrwt. 2 root  root      6 Aug 23 08:00 tmp
    drwxr-xr-x. 3 root  root     18 Aug 23 08:00 usr
    drwxr-xr-x. 4 root  root     30 Aug 23 08:00 var

## 2. 创建配置文件
使用libcontainer创建容器时，我们是创建了container.json文件，这里我们创建config.json，和container.json类似，包含容器所需的所有配置信息。config.json和rootfs目录在同一层级。

    #runc spec
    此时会生成一个名为config.json的配置文件。如果没有runc命令，可以使用yum install runc安装。

    config.json标准配置如下：
    {
        "ociVersion": "1.0.0-rc5",
        "platform": {
                "os": "linux",
                "arch": "amd64"
        },
        "process": {
                "terminal": true,
                "consoleSize": {
                        "height": 0,
                        "width": 0
                },
                "user": {
                        "uid": 0,
                        "gid": 0
                },
                "args": [
                        "sh"
                ],
                "env": [
                        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                        "TERM=xterm"
                ],
                "cwd": "/",
                "capabilities": {
                        "bounding": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "effective": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "inheritable": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "permitted": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ],
                        "ambient": [
                                "CAP_AUDIT_WRITE",
                                "CAP_KILL",
                                "CAP_NET_BIND_SERVICE"
                        ]
                },
                    "rlimits": [
                        {
                                "type": "RLIMIT_NOFILE",
                                "hard": 1024,
                                "soft": 1024
                        }
                ],
                "noNewPrivileges": true
        },
        "root": {
                "path": "rootfs",
                "readonly": true
        },
        "hostname": "runc",
        "mounts": [
                {
                        "destination": "/proc",
                        "type": "proc",
                        "source": "proc"
                },
                {
                        "destination": "/dev",
                        "type": "tmpfs",
                        "source": "tmpfs",
                        "options": [
                                "nosuid",
                                "strictatime",
                                "mode=755",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/pts",
                        "type": "devpts",
                        "source": "devpts",
                        "options": [
                                "nosuid",
                                "noexec",
                                "newinstance",
                                "ptmxmode=0666",
                                "mode=0620",
                                "gid=5"
                        ]
                },
                {
                        "destination": "/dev/shm",
                        "type": "tmpfs",
                        "source": "shm",
                        "options": [
                                "nosuid",
                                    "noexec",
                                "nodev",
                                "mode=1777",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/mqueue",
                        "type": "mqueue",
                        "source": "mqueue",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev"
                        ]
                },
                {
                        "destination": "/sys",
                        "type": "sysfs",
                        "source": "sysfs",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "ro"
                        ]
                },
                {
                        "destination": "/sys/fs/cgroup",
                        "type": "cgroup",
                        "source": "cgroup",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "relatime",
                                "ro"
                        ]
                }
        ],
        "linux": {
                "resources": {
                        "devices": [
                                {
                                        "allow": false,
                                        "access": "rwm"
                                }
                        ]
                },
                    "namespaces": [
                        {
                                "type": "pid"
                        },
                        {
                                "type": "network"
                        },
                        {
                                "type": "ipc"
                        },
                        {
                                "type": "uts"
                        },
                        {
                                "type": "mount"
                        }
                ],
                "maskedPaths": [
                        "/proc/kcore",
                        "/proc/latency_stats",
                        "/proc/timer_list",
                        "/proc/timer_stats",
                        "/proc/sched_debug",
                        "/sys/firmware"
                ],
                "readonlyPaths": [
                        "/proc/asound",
                        "/proc/bus",
                        "/proc/fs",
                        "/proc/irq",
                        "/proc/sys",
                        "/proc/sysrq-trigger"
                ]
        }
    }

## 3. 启动容器
可以使用docker-runc启动，也可以使用runc命令启动。

    [root@localhost my_container]# runc run busybox
    / # ps aux
    PID   USER     TIME   COMMAND
        1 root       0:00 sh
        4 root       0:00 ps aux
    / # 

    注意：runc必须使用root权限启动。
