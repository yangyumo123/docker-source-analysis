cgroups术语
============================================================
## 简介
本文介绍cgroups相关的术语及其关系，并简单介绍了各种subsystem。

## cgroups的术语

    task：cgroups中，task表示系统的一个进程。
    cgroup：控制组，cgroups中的资源控制都以cgroup为单位实现的。
    subsystem：子系统，是资源控制器。
    hierarchy：层级树，由一系列cgroup组成的树状结构，每个hierarchy绑定对应的subsystem进行资源调度。整个系统可以有多个hierarchy。cgroups是由多个hierarchy构成的森林。

    这4个关系需要遵循下面4个规则：
    1. 同一个hierarchy可以附加一个或多个subsystem。例如，cpu和memory的subsystem可以附加到同一个hierarchy上。
    2. 一个subsystem可以附加到多个hierarchy，但是当hierarchy上已经有其他的subsystem时，就不能再附加到该hierarchy上了。
    3. 一个task不能存在于同一个hierarchy的不同cgroup中，但是一个task可以存在于不同hierarchy中的cgroup中。之所以不同存在于同一个hierarchy中的多个cgroup中，是因为防止资源分配出现矛盾。例如，cpu的subsystem为同一个hierarchy中的/cg1分配10%，为/cg2分配20%，这就出现了矛盾。
    4. 子task和父task在fork之后是处于同一个cgroup中，之后可以被移到其他cgroup中。

## subsystem子系统
目前，docker使用以下subsystem：

    blkio：设置块设备（磁盘、固态硬盘、USB等）设置IO限制。
    cpu：设置task对cpu的使用。
    cpuacct：生成cgroup中task对cpu资源使用情况的报告。
    cpuset：为cgroup中的task分配独立的cpu（多处理器系统）。
    devices：开启或关闭cgroup中task对设备的访问。
    freezer：挂起或恢复cgrou中的task。
    memory：设置cgroup中task对内存使用量的限额，并生成task对内存使用情况的报告。
    perf-event：对cgroup中task进行统一的性能测试。
    net_cls：对cgroup中数据包进行流量控制。

_______________________________________________________________________
[[返回cgroups.md]](./cgroups.md) 


