一个真实的docker daemon启动参数配置
=============================================================
## 简介
在讲解docker源码之前，我们先来看一下一个真实的docker服务的配置情况，便于我们了解应该为docker daemon配置什么样的启动参数？

下面以centos7为例，讲解通过yum方式安装并启动docker.service时的参数配置情况。

## 安装docker.service

    # yum install -y docker            //目前生产系统用的是docker 1.12.6版本
    
docker daemon进程绑定的是unix socket，而不是tcp端口。这个套接字默认的属主是root，其他用户可以使用sudo命令来访问这个套接字文件。因为这个原因，docker进程都必须以root身份运行。

    [root@localhost docker]# ls -l /var/run/docker.sock   
    srw-rw----. 1 root root 0 Sep 22 20:53 /var/run/docker.sock

为了其他用户不使用sudo也能使用docker，可以创建一个docker用户组，并把用户添加到这个组里。当docker进程启动的时候，会设置套接字可以被这个组的用户读写。

    # usermod -aG docker your_username

    //退出root，使用your_username用户登录
    $ docker run hello-world

## docker.service文件
使用yum install安装完docker后，会在/usr/lib/systemd/system/docker.service中设置docker.service。

    [Unit]
    Description=Docker Application Container Engine
    Documentation=http://docs.docker.com
    After=network.target
    Wants=docker-storage-setup.service
    Requires=docker-cleanup.timer

    [Service]
    Type=notify
    NotifyAccess=all
    KillMode=process
    EnvironmentFile=-/etc/sysconfig/docker
    EnvironmentFile=-/etc/sysconfig/docker-storage
    EnvironmentFile=-/etc/sysconfig/docker-network
    Environment=GOTRACEBACK=crash
    Environment=DOCKER_HTTP_HOST_COMPAT=1
    Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
    ExecStart=/usr/bin/dockerd-current \
            --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
            --default-runtime=docker-runc \
            --exec-opt native.cgroupdriver=systemd \
            --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
            $OPTIONS \
            $DOCKER_STORAGE_OPTIONS \
            $DOCKER_NETWORK_OPTIONS \
            $ADD_REGISTRY \
            $BLOCK_REGISTRY \
            $INSECURE_REGISTRY
    ExecReload=/bin/kill -s HUP $MAINPID
    LimitNOFILE=1048576
    LimitNPROC=1048576
    LimitCORE=infinity
    TimeoutStartSec=0
    Restart=on-abnormal
    MountFlags=slave

    [Install]
    WantedBy=multi-user.target

我们需要关注环境变量：EnvironmentFile，和docker执行命令：ExecStart。

## 环境变量：

    /etc/sysconfig/docker
    /etc/sysconfig/docker-storage
    /etc/sysconfig/docker-network
    GOTRACEBACK=crash
    DOCKER_HTTP_HOST_COMPAT=1
    PATH=/usr/libexec/docker:/usr/bin:/usr/sbin

### 1. /etc/sysconfig/docker
docker daemon的主配置文件，下面列出默认的配置：

    # /etc/sysconfig/docker

    # Modify these options if you want to change the way the docker daemon runs
    OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
    if [ -z "${DOCKER_CERT_PATH}" ]; then
        DOCKER_CERT_PATH=/etc/docker
    fi

    # If you want to add your own registry to be used for docker search and docker
    # pull use the ADD_REGISTRY option to list a set of registries, each prepended
    # with --add-registry flag. The first registry added will be the first registry
    # searched.
    #ADD_REGISTRY='--add-registry registry.access.redhat.com'

    # If you want to block registries from being used, uncomment the BLOCK_REGISTRY
    # option and give it a set of registries, each prepended with --block-registry
    # flag. For example adding docker.io will stop users from downloading images
    # from docker.io
    # BLOCK_REGISTRY='--block-registry'

    # If you have a registry secured with https but do not have proper certs
    # distributed, you can tell docker to not look for full authorization by
    # adding the registry to the INSECURE_REGISTRY line and uncommenting it.
    # INSECURE_REGISTRY='--insecure-registry'

    # On an SELinux system, if you remove the --selinux-enabled option, you
    # also need to turn on the docker_transition_unconfined boolean.
    # setsebool -P docker_transition_unconfined 1

    # Location used for temporary files, such as those created by
    # docker load and build operations. Default is /var/lib/docker/tmp
    # Can be overriden by setting the following environment variable.
    # DOCKER_TMPDIR=/var/tmp

    # Controls the /etc/cron.daily/docker-logrotate cron job status.
    # To disable, uncomment the line below.
    # LOGROTATE=false
    #

    # docker-latest daemon can be used by starting the docker-latest unitfile.
    # To use docker-latest client, uncomment below lines
    #DOCKERBINARY=/usr/bin/docker-latest
    #DOCKERDBINARY=/usr/bin/dockerd-latest
    #DOCKER_CONTAINERD_BINARY=/usr/bin/docker-containerd-latest
    #DOCKER_CONTAINERD_SHIM_BINARY=/usr/bin/docker-containerd-shim-latest

我们重点关心OPTIONS、ADD_REGISTRY和INSECURE_REGISTRY。

OPTIONS：

    选项，设置各种flag命令行参数。包括：--selinux-enabled，--log-driver=journald（journald表示日志输出到journald控制台上，json-file表示日志输出到文件），等等。

ADD_REGISTRY：

    默认docker registry是docker.io，即默认拉取image是从docker.io上拉取。ADD_REGISTRY参数可以增加你自己的docker registry。例如，增加红帽的registry：--add-registry registry.access.redhat.com。
    该值可以添加多个registry，格式：ADD_REGISTRY='--add-registry XXX --add-registry YYY --add-registry ZZZ ...'
    增加registry的flags参数也可以放在OPTIONS中。

INSECURE_REGISTRY：

    非安全端口访问registry。如果你的docker registry是使用非安全端口，那么需要添加这个参数，例如：INSECURE_REGISTRY='--insecure-registry XXX --insecure-registry YYY ...'
    该flags参数也可以放在OPTIONS中。


### 2. /etc/sysconfig/docker-storage
docker存储相关的配置：

    # This file may be automatically generated by an installation program.
    # Please DO NOT edit this file directly. Instead edit
    # /etc/sysconfig/docker-storage-setup and/or refer to
    # "man docker-storage-setup".

    # By default, Docker uses a loopback-mounted sparse file in
    # /var/lib/docker.  The loopback makes it slower, and there are some
    # restrictive defaults, such as 100GB max storage.

    DOCKER_STORAGE_OPTIONS=


### 3. /etc/sysconfig/docker-network
docker网络相关的配置：

    # /etc/sysconfig/docker-network
    DOCKER_NETWORK_OPTIONS=


## docker执行命令：
docker daemon执行命令：/usr/bin/dockerd-current

参数：

    --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current
    --default-runtime=docker-runc                                       //默认runtime
    --exec-opt native.cgroupdriver=systemd                              //cgroup驱动方式
    --userland-proxy-path=/usr/libexec/docker/docker-proxy-current
    $OPTIONS                                                            //在/etc/sysconfig/docker中的参数，前面已经介绍了。
    $DOCKER_STORAGE_OPTIONS                                             //在/etc/sysconfig/docker-storage中的参数，前面已经介绍了。
    $DOCKER_NETWORK_OPTIONS                                             //在/etc/sysconfig/docker-network中的参数，前面已经介绍了。
    $ADD_REGISTRY                                                       //在/etc/sysconfig/docker中的参数，前面已经介绍了。
    $BLOCK_REGISTRY                                                     //在/etc/sysconfig/docker中的参数，前面已经介绍了。
    $INSECURE_REGISTRY                                                  //在/etc/sysconfig/docker中的参数，前面已经介绍了。

完整执行语句：

    ExecStart=/usr/bin/dockerd-current \
            --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
            --default-runtime=docker-runc \
            --exec-opt native.cgroupdriver=systemd \
            --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
            $OPTIONS \
            $DOCKER_STORAGE_OPTIONS \
            $DOCKER_NETWORK_OPTIONS \
            $ADD_REGISTRY \
            $BLOCK_REGISTRY \
            $INSECURE_REGISTRY


