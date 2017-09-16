Linux内核namespace的相关知识
===============================================================
## 简介
本文介绍Docker容器隔离技术中使用的Linux内核知识：Namespace。本文档分析linux 3.10.107版本。

基本上一个容器所需要6项隔离，即Linux内核提供的6种namespace隔离：

|Namespace|系统调用参数|隔离内容|
|----------|------------|------------------|
|UTS|CLONE_NEWUTS|主机名与域名|
|IPC|CLONE_NEWIPC|信号量、消息队列和共享内存|
|PID|CLONE_NEWPID|进程号|
|Network|CLONE_NEWNET|network devices、stacks、ports等|
|Mount|CLONE_NEWNS|挂载点（文件系统）|
|User|CLONE_NEWUSER|用户和用户组|

**补充**：还有一种namespace叫cgroups，它是在Linux内核4.6版本才出现，docker目前并未使用它。

## 目录
1. [Linux内核中的namesapce数据结构](./struct.md)
2. [调用namespace的API](./api.md)


_______________________________________________________________________
[[返回docker-linux.md]](../docker-linux.md) 