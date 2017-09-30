containerd源码分析
=================================================
## 简介
containerd是docker容器标准化后的产物，本身是一个daemon，控制runC。可以不使用docker daemon，而仅使用containerd来创建容器。

containerd的命令行工具是"ctr"，一些操作命令如下：

    # ctr containers start redis /contaienrs/redis
    # ctr containers list
    ID                  PATH                STATUS              PROCESSES
    redis               /containers/redis   running             14063

    其中，/containers/redis是OCI bundle的路径，bundle就是准备rootfs。

编译containerd和ctr

    可以单独编译containerd，使用"make"指令编译，在bin目录下生成3个二进制文件：containerd、containerd-shim和ctr，然后，使用"make install"指令将这3个二进制文件安装到/usr/local/bin下面。

    docker中使用containerd的编译方法如下：
    # git clone https://github.com/docker/containerd.git
    # git checkout -q 2a5e70cbf65457815ee76b7e5dd2a01292d9eca8             //注意，此版本和docker daemon相对应。在前面介绍的docker编译过程中，会下载containerd源码进行编译。
    # make static
    # cp bin/containerd /usr/local/bin/docker-containerd
	# cp bin/containerd-shim /usr/local/bin/docker-containerd-shim
	# cp bin/ctr /usr/local/bin/docker-containerd-ctr

## 说明
这里介绍docker-containerd源码，对应github项目为github.com/docker/containerd。

下面分析containerd的源码，讲解containerd是怎么创建容器和管理容器生命周期的。

## 目录
1. [参数初始化](./init.md)
2. [containerd主函数](./main.md)


_______________________________________________________________________
[[返回README.md]](../README.md) 



