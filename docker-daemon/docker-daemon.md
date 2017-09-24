docker daemon 源码分析
==============================================================
## 简介
docker daemon是docker的服务端，对应的二进制文件是dockerd。提供REST API，供docker client调用。

## 目录
1. [一个真实的docker daemon启动参数配置](./centos-docker.md)
2. [初始化](./init.md)
3. [入口](./main.md)
4. [启动](./start.md)
5. [创建docker-containerd](./containerd-daemon.md)
6. [创建dockerd](./newDaemon.md)

_______________________________________________________________________
[[返回README.md]](../README.md) 


