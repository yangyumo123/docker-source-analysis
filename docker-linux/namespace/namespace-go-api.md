Go语言实现的namespace的API
==========================================================
## 简介
Go语言的库：github.com/Sirupsen/logrus，实现了namespace隔离的高层实现。syscall库是namespace的系统调用底层实现。

6个namespace隔离分别对应6个参数：CLONE_NEWUTS、CLONE_NEWIPC、CLONE_NEWNS（因为mnt namespace是最早的隔离，所以它的名字比较特殊）、CLONE_NEWPID、CLONE_NEWNET、CLONE_NEWUSER。

## 1. syscall.SysProcAttr
含义：

    该结构体描述系统属性，包括namespace隔离相关的参数。

路径：

    src/syscall/exec_linux.go

定义：

    type SysProcAttr struct {
        Chroot       string         // Chroot.
        Credential   *Credential    // Credential.
        Ptrace       bool           // Enable tracing.
        Setsid       bool           // Create session.
        Setpgid      bool           // Set process group ID to Pgid, or, if Pgid == 0, to new pid.
        Setctty      bool           // Set controlling terminal to fd Ctty (only meaningful if Setsid is set)
        Noctty       bool           // Detach fd 0 from controlling terminal
        Ctty         int            // Controlling TTY fd
        Foreground   bool           // Place child's process group in foreground. (Implies Setpgid. Uses Ctty as fd of controlling TTY)
        Pgid         int            // Child's process group ID if Setpgid.
        Pdeathsig    Signal         // Signal that the process will get when its parent dies (Linux only)
        Cloneflags   uintptr        // Flags for clone calls (Linux only).    CLONE_NEW*，用于clone调用。
        Unshareflags uintptr        // Flags for unshare calls (Linux only)
        UidMappings  []SysProcIDMap // User ID mappings for user namespaces.
        GidMappings  []SysProcIDMap // Group ID mappings for user namespaces.
        // GidMappingsEnableSetgroups enabling setgroups syscall.
        // If false, then setgroups syscall will be disabled for the child process.
        // This parameter is no-op if GidMappings == nil. Otherwise for unprivileged
        // users this should be set to false for mappings work.
        GidMappingsEnableSetgroups bool
    }

Cloneflags可以取CLONE_NEW*和其他一些CLONE_*。其中CLONE_NEW*定义在：src/syscall/zerrors_linux_amd64.go（根据自己的OSARCH选择合适的.go）中。

	CLONE_NEWIPC                     = 0x8000000
	CLONE_NEWNET                     = 0x40000000
	CLONE_NEWNS                      = 0x20000
	CLONE_NEWPID                     = 0x20000000
	CLONE_NEWUSER                    = 0x10000000
	CLONE_NEWUTS                     = 0x4000000


## 2. /proc/[pid]/ns
含义：

    和C语言的一样。这里就不介绍了。




_______________________________________________________________________
[[返回namespace.md]](./namespace.md) 


