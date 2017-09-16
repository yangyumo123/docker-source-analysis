Linux内核namespace的相关知识
===============================================================
## 简介
本文介绍Docker容器隔离技术中使用的Linux内核知识：Namespace。本文档分析linux 3.10.107版本。

每一个namespace是包装了一些全局系统资源的抽象集合。

基本上一个容器所需要6项隔离，即Linux内核提供的6种namespace隔离：

|Namespace|系统调用参数|隔离内容|
|----------|------------|------------------|
|UTS|CLONE_NEWUTS|主机名与域名|
|IPC|CLONE_NEWIPC|信号量、消息队列和共享内存|
|PID|CLONE_NEWPID|进程号|
|Network|CLONE_NEWNET|network devices、stacks、ports等|
|Mount|CLONE_NEWNS|挂载点（文件系统）|
|User|CLONE_NEWUSER|用户ID、用户组ID、root目录、key（密钥）和特殊权限|

**补充**：

还有一种namespace叫cgroups，它是在Linux内核4.6版本才出现，docker目前并未使用它。

user namespace是在Linux内核3.8才完成的。docker从1.10版本开始支持user namespace，并且不是默认开启的。

## 目录
1. [Linux内核中的namesapce数据结构](./namespace-struct.md)
2. [调用namespace的C语言API](./namespace-c-api.md)
3. [调用namespace的Go语言API](./namespace-go-api.md)
4. [每种namespace的详细分析](./namespace-analysis.md)


_______________________________________________________________________
[[返回docker-linux.md]](../docker-linux.md) 
