docker daemon 源码分析
==============================================================
## 简介
docker daemon是docker的服务端，对应的二进制文件是dockerd。提供REST API，供docker client调用。

docker服务启动时，会启动2个进程：dockerd和docker-containerd。其中，dockerd就是docker daemon，docker-containerd是创建和管理容器的daemon，是从原先的docker daemon中拆分出来的。如果启动容器，则还会启动一个进程：docker-containerd-shim，这个进程会调用docker-runc命令，实际创建容器和管理容器生命周期。

## 目录
1. [一个真实的docker daemon启动参数配置](./centos-docker.md)
2. [初始化](./init.md)
3. [入口](./main.md)
4. [启动](./start.md)
5. [创建docker-containerd](./containerd-daemon.md)
6. [创建dockerd](./newDaemon.md)

_______________________________________________________________________
[[返回README.md]](../README.md) 


