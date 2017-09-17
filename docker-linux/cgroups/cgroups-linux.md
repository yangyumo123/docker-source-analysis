cgroups内核结构及API
====================================================
## 简介
cgroups的API是一个伪文件系统，用户可以通过文件操作来实现cgroups的管理。

## cgroups内核实现
cgroups是内核给进程挂的钩子，当进程运行到资源调度时就会触发这些钩子上附带的subsystem进行检测，根据资源类别进行资源限制和管理。

在task_struct结构体中有2个字段与cgroups相关：css_set和list_head。

关于task_struct的详细定义，请看参考文献：[[进程结构体task_struct的完整定义]](../../reference/task.md)

    struct task_struct {
        ...
    #ifdef CONFIG_CGROUPS
        /* Control Group info protected by css_set_lock */
        struct css_set __rcu *cgroups;                                       //指向css_set结构体
        /* cg_list protected by css_set_lock and tsk->alloc_lock */
        struct list_head cg_list;                                            //链表头
    #endif
    ...
    }

css_set结构体：

    //include/linux/cgroup.h
    struct css_set {

        /* Reference count */
        atomic_t refcount;

        /*
        * List running through all cgroup groups in the same hash
        * slot. Protected by css_set_lock
        */
        struct hlist_node hlist;

        /*
        * List running through all tasks using this cgroup
        * group. Protected by css_set_lock
        */
        struct list_head tasks;                                                      //指向task_struct的list_head

        /*
        * List of cg_cgroup_link objects on link chains from
        * cgroups referenced from this css_set. Protected by
        * css_set_lock
        */
        struct list_head cg_links;                                                   //指向由cg_cgroup_link形成的链表。

        /*
        * Set of subsystem states, one for each subsystem. This array
        * is immutable after creation apart from the init_css_set
        * during subsystem registration (at boot time) and modular subsystem
        * loading/unloading.
        */
        struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];                    

        /* For RCU-protected deletion */
        struct rcu_head rcu_head;
    };

cg_cgroup_link结构体:

    //kernel/cgroup.c
    struct cg_cgroup_link {
        /*
        * List running through cg_cgroup_links associated with a
        * cgroup, anchored on cgroup->css_sets
        */
        struct list_head cgrp_link_list;                                             //cgroup结构体中的css_set指向了它
        struct cgroup *cgrp;                                                         //指向cgroup结构体
        /*
        * List running through cg_cgroup_links pointing at a
        * single css_set object, anchored on css_set->cg_links
        */
        struct list_head cg_link_list;                                               //css_set结构体中的 struct list_head cg_links;指向了它。
        struct css_set *cg;                                                          //指向每一个task的css_set。在task和cgroup之间形成了双向链表。
    };

cgroup结构体：

    //include/linux/cgroup.h
    struct cgroup {
        unsigned long flags;		/* "unsigned long" so bitops work */

        /*
        * count users of this cgroup. >0 means busy, but doesn't
        * necessarily indicate the number of tasks in the cgroup
        */
        atomic_t count;

        int id;				/* ida allocated in-hierarchy ID */

        /*
        * We link our 'sibling' struct into our parent's 'children'.
        * Our children link their 'sibling' into our 'children'.
        */
        struct list_head sibling;	/* my parent's children */
        struct list_head children;	/* my children */
        struct list_head files;		/* my files */

        struct cgroup *parent;		/* my parent */
        struct dentry *dentry;		/* cgroup fs entry, RCU protected */

        /*
        * This is a copy of dentry->d_name, and it's needed because
        * we can't use dentry->d_name in cgroup_path().
        *
        * You must acquire rcu_read_lock() to access cgrp->name, and
        * the only place that can change it is rename(), which is
        * protected by parent dir's i_mutex.
        *
        * Normally you should use cgroup_name() wrapper rather than
        * access it directly.
        */
        struct cgroup_name __rcu *name;

        /* Private pointers for each registered subsystem */
        struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

        struct cgroupfs_root *root;

        /*
        * List of cg_cgroup_links pointing at css_sets with
        * tasks in this cgroup. Protected by css_set_lock
        */
        struct list_head css_sets;                                                      //指向cg_cgroup_links

        struct list_head allcg_node;	/* cgroupfs_root->allcg_list */
        struct list_head cft_q_node;	/* used during cftype add/rm */

        /*
        * Linked list running through all cgroups that can
        * potentially be reaped by the release agent. Protected by
        * release_list_lock
        */
        struct list_head release_list;

        /*
        * list of pidlists, up to two for each namespace (one for procs, one
        * for tasks); created on demand.
        */
        struct list_head pidlists;
        struct mutex pidlist_mutex;

        /* For RCU-protected deletion */
        struct rcu_head rcu_head;
        struct work_struct free_work;

        /* List of events which userspace want to receive */
        struct list_head event_list;
        spinlock_t event_list_lock;

        /* directory xattrs */
        struct simple_xattrs xattrs;
    };

从内核源码来看，进程task_struct和cgroup之间是多对多关系，双向链表：

    从task_struct到cgroup方向：
        task_struct（css_set *cgroups） --> css_set（list_head cg_links） --> cg_cgroup_list（cgroup *cgrp） --> cgroup
    从cgroup到task_struct方向：
        cgroup（list_head css_sets） --> cg_cgroup_list（css_set *cg） --> css_set（list_head tasks） --> task_struct




## 参考文献
* [[进程结构体task_struct的完整定义]](../../reference/task.md)


_______________________________________________________________________
[[返回cgroups.md]](./cgroups.md) 

