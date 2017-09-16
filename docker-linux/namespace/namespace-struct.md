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
    };

补充：这里仅列出与namespace相关的参数，如需了解全部参数，请看参考文献：[[进程结构体task_struct的完整定义]](../../reference/task.md)


## nsproxy
含义：

    包含各种namespace的定义。

路径：

    include/linux/nsproxy.h

定义：

    struct nsproxy {
        atomic_t             count;      // 是nsproxy所指向的namespace的数量，而不是task的数量。
        struct uts_namespace *uts_ns;    // uts namespace
        struct ipc_namespace *ipc_ns;    // ipc namespace
        struct mnt_namespace *mnt_ns;    // mnt namespace
        struct pid_namespace *pid_ns;    // pid namespace
        struct net           *net_ns;    // net namespace
    };

下面详细介绍每种namespace的定义：

### 1. uts_namespace
含义：

    主机名和域名的namespace隔离。

路径：

    include/linux/utsname.h

定义：

    struct uts_namespace {
        struct kref kref;
        struct new_utsname name;
        struct user_namespace *user_ns;
        unsigned int proc_inum;
    };

### 2. ipc_namespace
含义：

    进程间通信的namespace隔离，主要是消息队列。

路径：

    include/linux/ipc_namespace.h

定义：

    struct ipc_namespace {
        atomic_t count;
        struct ipc_ids ids[3];
        int sem_ctls[4];
        int used_sems;
        unsigned int msg_ctlmax;
        unsigned int msg_ctlmnb;
        unsigned int msg_ctlmni;
        atomic_t msg_bytes;
        atomic_t msg_hdrs;
        int auto_msgmni;
        size_t shm_ctlmax;
        size_t shm_ctlall;
        unsigned long shm_tot;
        int shm_ctlmni;
        /*
         * Defines whether IPC_RMID is forced for _all_ shm segments regardless
         * of shmctl()
         */
        int             shm_rmid_forced;

        struct notifier_block ipcns_nb;

        /* The kern_mount of the mqueuefs sb.  We take a ref on it */
        struct vfsmount *mq_mnt;

        /* # queues in this ns, protected by mq_lock */
        unsigned int    mq_queues_count;

        /* next fields are set through sysctl */
        unsigned int    mq_queues_max;   /* initialized to DFLT_QUEUESMAX */
        unsigned int    mq_msg_max;      /* initialized to DFLT_MSGMAX */
        unsigned int    mq_msgsize_max;  /* initialized to DFLT_MSGSIZEMAX */
        unsigned int    mq_msg_default;
        unsigned int    mq_msgsize_default;

        /* user_ns which owns the ipc ns */
        struct user_namespace *user_ns;

        unsigned int    proc_inum;
    };

### 3. mnt_namespace
含义：

    文件系统的namespace隔离。mnt namespace是第一种出现的namespace，因此它的参数命名比较特殊，CLONE_NEWNS。

路径：

    fs/mount.h

定义：

    struct mnt_namespace {
        atomic_t count;
        unsigned int proc_inum;
        struct mount * root;
        struct list_head list;
        struct user_namespace *user_ns;
        u64 seq; /* Sequence number to revent loops */
        wait_queue_head_t poll;
        int event;
    };

### 4. pid_namespace
含义：

    进程号的namespace隔离。

路径：

    include/linux/pid_namespace.h

定义：

    struct pid_namespace {
        struct kref kref;
        struct pidmap pidmap[PIDMAP_ENTRIES];
        int last_pid;
        unsigned int nr_hashed;
        struct task_struct *child_reaper;
        struct kmem_cache *pid_cachep;
        unsigned int level;
        struct pid_namespace *parent;
    #ifdef CONFIG_PROC_FS
        struct vfsmount *proc_mnt;
        struct dentry *proc_self;
    #endif
    #ifdef CONFIG_BSD_PROCESS_ACCT
        struct bsd_acct_struct *bacct;
    #endif
        struct user_namespace *user_ns;
        struct work_struct proc_work;
        kgid_t pid_gid;
        int hide_pid;
        int reboot;     /* group exit code if this pidns was rebooted */
        unsigned int proc_inum;
    };

### 5. net
含义：

    网络的namespace隔离，包括：devices、stacks、ports等。

路径：

    net/net_namespace.h

定义：

    struct net {
        atomic_t		passive;	/* To decided when the network
                            * namespace should be freed.
                            */
        atomic_t		count;		/* To decided when the network
                            *  namespace should be shut down.
                            */
    #ifdef NETNS_REFCNT_DEBUG
        atomic_t		use_count;	/* To track references we
                            * destroy on demand
                            */
    #endif
        spinlock_t		rules_mod_lock;

        struct list_head	list;		/* list of network namespaces */
        struct list_head	cleanup_list;	/* namespaces on death row */
        struct list_head	exit_list;	/* Use only net_mutex */

        struct user_namespace   *user_ns;	/* Owning user namespace */

        unsigned int		proc_inum;

        struct proc_dir_entry 	*proc_net;
        struct proc_dir_entry 	*proc_net_stat;

    #ifdef CONFIG_SYSCTL
        struct ctl_table_set	sysctls;
    #endif

        struct sock 		*rtnl;			/* rtnetlink socket */
        struct sock		*genl_sock;

        struct list_head 	dev_base_head;
        struct hlist_head 	*dev_name_head;
        struct hlist_head	*dev_index_head;
        unsigned int		dev_base_seq;	/* protected by rtnl_mutex */
        int			ifindex;

        /* core fib_rules */
        struct list_head	rules_ops;


        struct net_device       *loopback_dev;          /* The loopback */
        struct netns_core	core;
        struct netns_mib	mib;
        struct netns_packet	packet;
        struct netns_unix	unx;
        struct netns_ipv4	ipv4;
    #if IS_ENABLED(CONFIG_IPV6)
        struct netns_ipv6	ipv6;
    #endif
    #if defined(CONFIG_IP_SCTP) || defined(CONFIG_IP_SCTP_MODULE)
        struct netns_sctp	sctp;
    #endif
    #if defined(CONFIG_IP_DCCP) || defined(CONFIG_IP_DCCP_MODULE)
        struct netns_dccp	dccp;
    #endif
    #ifdef CONFIG_NETFILTER
        struct netns_nf		nf;
        struct netns_xt		xt;
    #if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
        struct netns_ct		ct;
    #endif
    #if IS_ENABLED(CONFIG_NF_DEFRAG_IPV6)
        struct netns_nf_frag	nf_frag;
    #endif
        struct sock		*nfnl;
        struct sock		*nfnl_stash;
    #endif
    #ifdef CONFIG_WEXT_CORE
        struct sk_buff_head	wext_nlevents;
    #endif
        struct net_generic __rcu	*gen;

        /* Note : following structs are cache line aligned */
    #ifdef CONFIG_XFRM
        struct netns_xfrm	xfrm;
    #endif
        struct netns_ipvs	*ipvs;
        struct sock		*diag_nlsk;
        atomic_t		rt_genid;
    };

### 6. user_namespace
含义：

    user namespace隔离是在Linux内核3.8中才完成的，docker1.10中才使用，而且不是默认开启的。user_namespace的定义也不是在nsproxy中。

路径：

    include/linux/user_namespace.h

定义：

    struct user_namespace {
	    struct uid_gid_map	    uid_map;
	    struct uid_gid_map	    gid_map;
	    struct uid_gid_map	    projid_map;
	    atomic_t		        count;
	    struct user_namespace	*parent;
	    int			            level;
	    kuid_t			        owner;
	    kgid_t			        group;
	    unsigned int		    proc_inum;
	    unsigned long		    flags;
	    bool			        may_mount_sysfs;
	    bool			        may_mount_proc;
    };








__________________________________________________________________________


## 参考文献
* [[进程结构体task_struct的完整定义]](../../reference/task.md)


_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 

