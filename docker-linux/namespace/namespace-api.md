调用namespace的API
==========================================================
## 简介
namespace的API包括clone、setns以及unshare，还有/proc下的部分文件。

6个namespace隔离分别对应6个参数：CLONE_NEWUTS、CLONE_NEWIPC、CLONE_NEWNS（因为mnt namespace是最早的隔离，所以它的名字比较特殊）、CLONE_NEWPID、CLONE_NEWNET、CLONE_NEWUSER。

## 1. clone
含义：

    创建一个新进程并把它放到新的namespace中。clone相当于unix中的fork系统调用。
    Unix复制进程的系统调用是fork，但是，Linux中实现了3个：fork、vfork和clone。
        fork：创建的子进程和父进程拥有完全相同的副本，包括task_struct内容、内存的内容等等。
        vfork：创建的子进程与父进程共享数据段，而且由vfork（）创建的子进程将先于父进程运行。
        clone：Linux上创建线程一般用pthread库，然而Linux本身也给我们提供了创建线程的系统调用，就是clone。

定义：

    int clone(int (*child_func)(void*), void *child_stack, int flags, void *arg)

参数：

* child_func：子进程的main函数。
* child_stack：子进程运行的栈空间。
* flags：可以取上面的CLONE_NEW*（也可以取跟namespace无关的flag，例如：CLONE_VM、CLONE_FILES等，用|分隔），这样就会创建一个或多个不同类型的namespace，并把新创建的进程加入到新创建的这些namespace中。
* arg：用户传入的参数。

## 2. /proc/[pid]/ns
含义：

在/proc/[pid]/ns文件下可以看到指向不同namespace编号的文件，如下所示：

    lrwxrwxrwx. 1 901001 9010 0 Sep 15 17:33 ipc -> ipc:[4026533110]
    lrwxrwxrwx. 1 901001 9010 0 Sep 15 16:27 mnt -> mnt:[4026533206]
    lrwxrwxrwx. 1 901001 9010 0 Sep 15 17:33 net -> net:[4026533113]
    lrwxrwxrwx. 1 901001 9010 0 Sep 15 16:27 pid -> pid:[4026533208]
    lrwxrwxrwx. 1 901001 9010 0 Sep 15 17:33 user -> user:[4026531837]
    lrwxrwxrwx. 1 901001 9010 0 Sep 15 16:27 uts -> uts:[4026533207]

    如果两个进程的/proc/[pid]/ns下面的文件指向的namespace编号相同，则说明它们是在同一个namespace下。

/proc/[pid]/ns的另外一个作用是，一旦文件被打开，只要打开的文件描述符存在，那么就算PID所属的所有进程都已经结束，创建的namespace也会一直存在。可以把/proc/[pid]/ns挂载到一个目录中实现保留namespace的作用，为以后进程加入这个namespace做准备。

    # touch ~/net
    # mount --bind /proc/22222/ns/net ~/net     //保留net namespace

## 3. setns
含义：

    将进程加入一个新的namespace中。

定义：

    int setns(int fd, int nstype);

参数：

* fd：要加入的namespace的文件描述符，是指向/proc/[pid]/ns目录的文件描述符，可以通过打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。
* nstype：指定namespace的类型。如果当前进程不能根据fd得到它的类型，如fd由其他进程创建，并通过Unix domain socket传给当前进程，那么就需要通过nstype来指定fd指向的namespace类型。如果进程能根据fd得到namespace类型，比如这个fd是由当前进程打开的，那么nstype设置为0即可。

示例：

    fd = open(argv[1], O_RDONLY);  //获取namespace文件描述符
    setns(fd, 0);                  //将调用进程加入到新的namespace中
    execvp(argv[2], &argv[2]);     //执行用户命令

    假设编译后程序名称为setns：
    # ./setns ~/net /bin/bash      // 至此，可以在新的命令空间中执行shell命令了。

### 4. unshare
含义：

    unshare使当前进程退出指定类型的namespace，并加入到新创建的namespace。运行在原先的进程上，相当于跳出原先的namespace。

定义：

    int unshare(int flags);

参数：

* flags：上面的CLONE_NEW*。这样当前进程就退出了当前指定类型的namespace，并加入到新创建的namespace。



_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 


