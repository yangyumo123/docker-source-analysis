Linux内核中的namesapce数据结构
===========================================================
## 简介
本文介绍Linux内核namespace相关的结构体定义。

## task_struct
含义：

    Linux中的进程也被称为task，进程结构体（task_struct）中包含一个指向namespace结构体（nsproxy）的指针*nsproxy。从定义中可以看出，一个进程是关联一组namespace集合的，即可以对一个进程实现各种类型的namesapce隔离。

路径：

    include/linux/sched.h

定义：

    struct task_struct {
        ...
        /* namespace */
        struct nsproxy *nsproxy;                 // nsproxy结构体包含各种namespace定义。
        ...
    }

补充：这里仅列出与namespace相关的参数，如需了解全部参数，请看参考文献：[[进程结构体task_struct的完整定义]](../../reference/task.md)


## nsproxy
含义：

路径：

定义：



__________________________________________________________________________


## 参考文献
* [[进程结构体task_struct的完整定义]](../../reference/task.md)

