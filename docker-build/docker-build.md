编译docker源码
=======================================================
## 简介
本文介绍怎么编译docker源码，docker二进制文件的组成。

## 说明
文本使用docker版本为：1.12.6，commit=78d18021ecba00c00730dec9d56de6896f9e708d

## 编译方法
下载docker源码：

    # cd ~
    # git clone https://github.com/moby/moby.git                             //下载docker源码，docker现在改名为moby了
    # git checkout -q 78d18021ecba00c00730dec9d56de6896f9e708d               //切换到1.12.6版本

编译docker源码：

    # cd moby
    # make
    # make install                                                           //这个会将bundles目录中的二进制文件copy到/usr/local/bin下面

## 源码输出
下面是第二次执行make的结果，已经存在镜像的情况下，就会省去中间下载镜像，下载依赖等步骤。

第一次编译会生成完整的编译输出，包括下载镜像，下载依赖包等等，完整的输出请看参考文献：[[编译docker源码的完整输出]](../reference/docker-build.md)。

    docker build  -t "docker-dev:HEAD" -f "Dockerfile" .
    Sending build context to Docker daemon 187.1 MB
    Step 1 : FROM debian:jessie
    ---> 0a374bb127cf
    Step 2 : RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys E871F18B51E0147C77796AC81196BA81F6B0FC61  || apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys E871F18B51E0147C77796AC81196BA81F6B0FC61
    ---> Using cache
    ---> 3e59ba87dd8e
    Step 3 : RUN echo deb http://ppa.launchpad.net/zfs-native/stable/ubuntu trusty main > /etc/apt/sources.list.d/zfs.list
    ---> Using cache
    ---> 472d6757416a
    Step 4 : ARG APT_MIRROR=httpredir.debian.org
    ---> Using cache
    ---> 85c99b3f8215
    Step 5 : RUN sed -i s/httpredir.debian.org/$APT_MIRROR/g /etc/apt/sources.list
    ---> Using cache
    ---> 32b0ad525878
    Step 6 : RUN apt-get update && apt-get install -y       apparmor        apt-utils       aufs-tools      automake        bash-completion         binutils-mingw-w64      bsdmainutils    btrfs-tools  build-essential         clang   createrepo      curl    dpkg-sig        gcc-mingw-w64   git     iptables        jq      libapparmor-dev         libcap-dev      libltdl-dev     libsqlite3-dev       libsystemd-journal-dev  libtool         mercurial       net-tools       pkg-config      python-dev      python-mock     python-pip      python-websocket        ubuntu-zfs  xfsprogs         libzfs-dev      tar     zip     --no-install-recommends         && pip install awscli==1.10.15
    ---> Using cache
    ---> c635888e7505
    Step 7 : ENV LVM2_VERSION 2.02.103
    ---> Using cache
    ---> 32888190c2fd
    Step 8 : RUN mkdir -p /usr/local/lvm2   && curl -fsSL "https://mirrors.kernel.org/sourceware/lvm2/LVM2.${LVM2_VERSION}.tgz"             | tar -xzC /usr/local/lvm2 --strip-components=1
    ---> Using cache
    ---> cb83736b3c3f
    Step 9 : RUN cd /usr/local/lvm2         && ./configure          --build="$(gcc -print-multiarch)"               --enable-static_link    && make device-mapper   && make install_device-mapper
    ---> Using cache
    ---> 4f504c5c0c23
    Step 10 : ENV OSX_SDK MacOSX10.11.sdk
    ---> Using cache
    ---> c4ce6bcc304a
    Step 11 : ENV OSX_CROSS_COMMIT 8aa9b71a394905e6c5f4b59e2b97b87a004658a4
    ---> Using cache
    ---> f444dd7d1121
    Step 12 : RUN set -x    && export OSXCROSS_PATH="/osxcross"     && git clone https://github.com/tpoechtrager/osxcross.git $OSXCROSS_PATH        && ( cd $OSXCROSS_PATH && git checkout -q $OSX_CROSS_COMMIT)         && curl -sSL https://s3.dockerproject.org/darwin/v2/${OSX_SDK}.tar.xz -o "${OSXCROSS_PATH}/tarballs/${OSX_SDK}.tar.xz"  && UNATTENDED=yes OSX_VERSION_MIN=10.6 ${OSXCROSS_PATH}/build.sh
    ---> Using cache
    ---> 0d031f9f5daa
    Step 13 : ENV PATH /osxcross/target/bin:$PATH
    ---> Using cache
    ---> 538f139b24a5
    Step 14 : ENV SECCOMP_VERSION 2.3.1
    ---> Using cache
    ---> d907919e1b14
    Step 15 : RUN set -x    && export SECCOMP_PATH="$(mktemp -d)"   && curl -fsSL "https://github.com/seccomp/libseccomp/releases/download/v${SECCOMP_VERSION}/libseccomp-${SECCOMP_VERSION}.tar.gz"             | tar -xzC "$SECCOMP_PATH" --strip-components=1         && (            cd "$SECCOMP_PATH"              && ./configure --prefix=/usr/local              && make             && make install          && ldconfig     )       && rm -rf "$SECCOMP_PATH"
    ---> Using cache
    ---> cac39b293e3b
    Step 16 : ENV GO_VERSION 1.6.4
    ---> Using cache
    ---> b0a2a77a8438
    Step 17 : ENV DOCKER_CROSSPLATFORMS linux/386 linux/arm         darwin/amd64    freebsd/amd64 freebsd/386 freebsd/arm   windows/amd64 windows/386
    ---> Using cache
    ---> 094c6a4fd8b9
    Step 18 : RUN curl -fsSL "https://storage.googleapis.com/golang/go1.4.3.linux-amd64.tar.gz"     | tar -xzC /root &&     mv /root/go /root/go1.4 &&      cd /usr/local &&        curl -fsSL "https://storage.googleapis.com/golang/go$GO_VERSION.src.tar.gz"  | tar -xzC /usr/local &&        cd go &&        printf 'diff --git a/src/runtime/sys_darwin_amd64.s b/src/runtime/sys_darwin_amd64.s\nindex e09b906..fa8ff2f 100644\n--- a/src/runtime/sys_darwin_amd64.s\n+++ b/src/runtime/sys_darwin_amd64.s\n@@ -157,6 +157,7 @@ systime:\n\t// Fall back to system call (usually first call in this thread).\n\tMOVQ\tSP, DI\n\tMOVQ\t$0, SI\n+\tMOVQ\t$0, DX  // required as of Sierra; Issue 16570\n\tMOVL\t$(0x2000000+116), AX\n\tSYSCALL\n\tCMPQ\tAX, $0\n' | patch -p1 &&  cd src &&        ./make.bash
    ---> Using cache
    ---> d909cb4d6bad
    Step 19 : ENV PATH /go/bin:/usr/local/go/bin:$PATH
    ---> Using cache
    ---> 5c02944056fc
    Step 20 : ENV GOPATH /go:/go/src/github.com/docker/docker/vendor
    ---> Using cache
    ---> c6ab415baf7c
    Step 21 : ENV GO_TOOLS_COMMIT 823804e1ae08dbb14eb807afc7db9993bc9e3cc3
    ---> Using cache
    ---> 78ec16a69e3f
    Step 22 : RUN git clone https://github.com/golang/tools.git /go/src/golang.org/x/tools  && (cd /go/src/golang.org/x/tools && git checkout -q $GO_TOOLS_COMMIT)  && go install -v golang.org/x/tools/cmd/cover        && go install -v golang.org/x/tools/cmd/vet
    ---> Using cache
    ---> 3a7df0595619
    Step 23 : ENV GO_LINT_COMMIT 32a87160691b3c96046c0c678fe57c5bef761456
    ---> Using cache
    ---> 4028bc141680
    Step 24 : RUN git clone https://github.com/golang/lint.git /go/src/github.com/golang/lint       && (cd /go/src/github.com/golang/lint && git checkout -q $GO_LINT_COMMIT)       && go install -v github.com/golang/lint/golint
    ---> Using cache
    ---> 1e1d9937b667
    Step 25 : ENV REGISTRY_COMMIT_SCHEMA1 ec87e9b6971d831f0eff752ddb54fb64693e51cd
    ---> Using cache
    ---> 08c053cefe06
    Step 26 : ENV REGISTRY_COMMIT 47a064d4195a9b56133891bbb13620c3ac83a827
    ---> Using cache
    ---> 028d0b4b7330
    Step 27 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone https://github.com/docker/distribution.git "$GOPATH/src/github.com/docker/distribution"    && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT")   && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH"                go build -o /usr/local/bin/registry-v2 github.com/docker/distribution/cmd/registry   && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT_SCHEMA1")      && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH"             go build -o /usr/local/bin/registry-v2-schema1 github.com/docker/distribution/cmd/registry      && rm -rf "$GOPATH"
    ---> Using cache
    ---> 5f36045540e0
    Step 28 : ENV NOTARY_VERSION v0.3.0
    ---> Using cache
    ---> 545611a8967c
    Step 29 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone https://github.com/docker/notary.git "$GOPATH/src/github.com/docker/notary"        && (cd "$GOPATH/src/github.com/docker/notary" && git checkout -q "$NOTARY_VERSION")  && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH"                 go build -o /usr/local/bin/notary-server github.com/docker/notary/cmd/notary-server  && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH"                 go build -o /usr/local/bin/notary github.com/docker/notary/cmd/notary   && rm -rf "$GOPATH"
    ---> Using cache
    ---> 8d546a25b436
    Step 30 : ENV DOCKER_PY_COMMIT 7befe694bd21e3c54bb1d7825270ea4bd6864c13
    ---> Using cache
    ---> e3a454e22700
    Step 31 : RUN git clone https://github.com/docker/docker-py.git /docker-py      && cd /docker-py        && git checkout -q $DOCKER_PY_COMMIT    && pip install -r test-requirements.txt
    ---> Using cache
    ---> b17051a185ff
    Step 32 : RUN git config --global user.email 'docker-dummy@example.com'
    ---> Using cache
    ---> 94349703f1e3
    Step 33 : RUN groupadd -r docker
    ---> Using cache
    ---> 602008a26ffd
    Step 34 : RUN useradd --create-home --gid docker unprivilegeduser
    ---> Using cache
    ---> 688acec113e4
    Step 35 : VOLUME /var/lib/docker
    ---> Using cache
    ---> 795b8ce9e885
    Step 36 : WORKDIR /go/src/github.com/docker/docker
    ---> Using cache
    ---> ad7b6a6a0799
    Step 37 : ENV DOCKER_BUILDTAGS apparmor pkcs11 seccomp selinux
    ---> Using cache
    ---> aff06851b145
    Step 38 : RUN ln -sfv $PWD/.bashrc ~/.bashrc
    ---> Using cache
    ---> bdb76d5bba77
    Step 39 : RUN ln -sv $PWD/contrib/completion/bash/docker /etc/bash_completion.d/docker
    ---> Using cache
    ---> e3d6b9540b0a
    Step 40 : COPY contrib/download-frozen-image-v2.sh /go/src/github.com/docker/docker/contrib/
    ---> Using cache
    ---> 2f3423bba416
    Step 41 : RUN ./contrib/download-frozen-image-v2.sh /docker-frozen-images       buildpack-deps:jessie@sha256:25785f89240fbcdd8a74bdaf30dd5599a9523882c6dfc567f2e9ef7cf6f79db6   busybox:latest@sha256:e4f93f6ed15a0cdd342f5aae387886fba0ab98af0a102da6276eaf24d6e6ade0       debian:jessie@sha256:f968f10b4b523737e253a97eac59b0d1420b5c19b69928d35801a6373ffe330e   hello-world:latest@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
    ---> Using cache
    ---> 97c8ca78a92a
    Step 42 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone --depth 1 -b v1.0.5 https://github.com/cpuguy83/go-md2man.git "$GOPATH/src/github.com/cpuguy83/go-md2man"  && git clone --depth 1 -b v1.4 https://github.com/russross/blackfriday.git "$GOPATH/src/github.com/russross/blackfriday"     && go get -v -d github.com/cpuguy83/go-md2man   && go build -v -o /usr/local/bin/go-md2man github.com/cpuguy83/go-md2man     && rm -rf "$GOPATH"
    ---> Using cache
    ---> 2cf87100244a
    Step 43 : ENV TOMLV_COMMIT 9baf8a8a9f2ed20a8e54160840c492f937eeaf9a
    ---> Using cache
    ---> 87dbfb694b00
    Step 44 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone https://github.com/BurntSushi/toml.git "$GOPATH/src/github.com/BurntSushi/toml"    && (cd "$GOPATH/src/github.com/BurntSushi/toml" && git checkout -q "$TOMLV_COMMIT")  && go build -v -o /usr/local/bin/tomlv github.com/BurntSushi/toml/cmd/tomlv     && rm -rf "$GOPATH"
    ---> Using cache
    ---> 370655d33bb5
    Step 45 : ENV RUNC_COMMIT 50a19c6ff828c58e5dab13830bd3dacde268afe5
    ---> Using cache
    ---> 76e4d4228797
    Step 46 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone https://github.com/docker/runc.git "$GOPATH/src/github.com/opencontainers/runc"    && cd "$GOPATH/src/github.com/opencontainers/runc"   && git checkout -q "$RUNC_COMMIT"       && make static BUILDTAGS="seccomp apparmor selinux"     && cp runc /usr/local/bin/docker-runc   && rm -rf "$GOPATH"
    ---> Using cache
    ---> ad699af60388
    Step 47 : ENV CONTAINERD_COMMIT 2a5e70cbf65457815ee76b7e5dd2a01292d9eca8
    ---> Using cache
    ---> bbd8706101a7
    Step 48 : RUN set -x    && export GOPATH="$(mktemp -d)"         && git clone https://github.com/docker/containerd.git "$GOPATH/src/github.com/docker/containerd"        && cd "$GOPATH/src/github.com/docker/containerd"     && git checkout -q "$CONTAINERD_COMMIT"         && make static  && cp bin/containerd /usr/local/bin/docker-containerd   && cp bin/containerd-shim /usr/local/bin/docker-containerd-shim      && cp bin/ctr /usr/local/bin/docker-containerd-ctr      && rm -rf "$GOPATH"
    ---> Using cache
    ---> d19687c335c2
    Step 49 : ENTRYPOINT hack/dind
    ---> Using cache
    ---> 4d12450eee21
    Step 50 : COPY . /go/src/github.com/docker/docker
    ---> Using cache
    ---> 23ebda74733a
    Successfully built 23ebda74733a
    docker run --rm -i --privileged -e BUILDFLAGS -e KEEPBUNDLE -e DOCKER_BUILD_GOGC -e DOCKER_BUILD_PKGS -e DOCKER_DEBUG -e DOCKER_EXPERIMENTAL -e DOCKER_GITCOMMIT -e DOCKER_GRAPHDRIVER=devicemapper -e DOCKER_INCREMENTAL_BINARY -e DOCKER_REMAP_ROOT -e DOCKER_STORAGE_OPTS -e DOCKER_USERLANDPROXY -e TESTDIRS -e TESTFLAGS -e TIMEOUT -v "/root/mygo/src/github.com/docker/docker/bundles:/go/src/github.com/docker/docker/bundles" -t "docker-dev:HEAD" hack/make.sh binary

    bundles/1.12.6 already exists. Removing.

    ---> Making bundle: binary (in bundles/1.12.6/binary)
    Building: bundles/1.12.6/binary-client/docker-1.12.6
    Created binary: bundles/1.12.6/binary-client/docker-1.12.6
    Building: bundles/1.12.6/binary-daemon/dockerd-1.12.6
    Created binary: bundles/1.12.6/binary-daemon/dockerd-1.12.6
    Building: bundles/1.12.6/binary-daemon/docker-proxy-1.12.6
    Created binary: bundles/1.12.6/binary-daemon/docker-proxy-1.12.6
    Copying nested executables into bundles/1.12.6/binary-daemon

## 分析编译过程
### 1. 创建bundles目录
第一次编译会在当前路径创建bundles目录，用于存储编译后的二进制文件。详细请看参考文献的完整版：[[编译docker源码的完整输出]](../reference/docker-build.md)

    mkdir bundles

### 2. 创建docker image
执行docker build指令，创建docker镜像："docker-dev:HEAD"。

    docker build  -t "docker-dev:HEAD" -f "Dockerfile" .

下面介绍docker image的创建过程：

### 3. 执行FROM
基础镜像：debian:jessie。第一次编译需要下载该基础镜像，当基础镜像下载到本地后，以后的编译就不需要下载了。

    Step 1 : FROM debian:jessie

### 4. 下载zfs ppa key
apt-key adv指令在debian linux中用于下载key（密钥）。

apt-key adv参数：

    --keyserver：指从哪个密钥服务器下载密钥，这里是从"hkp://p80.pool.sks-keyservers.net:80"或"hkp://pgp.mit.edu:80"下载密钥。
    --recv-keys：指接收密钥。
    E871F18B51E0147C77796AC81196BA81F6B0FC61：指key的指纹（fingerprint）

Dockerfile指令：

    RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys E871F18B51E0147C77796AC81196BA81F6B0FC61  || apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys E871F18B51E0147C77796AC81196BA81F6B0FC61

### 5. 创建zfs.list文件
把deb http://ppa.launchpad.net/zfs-native/stable/ubuntu trusty main内容拷贝到/etc/apt/sources.list.d/zfs.list目录下：

    RUN echo deb http://ppa.launchpad.net/zfs-native/stable/ubuntu trusty main > /etc/apt/sources.list.d/zfs.list

### 6. Dockerfile定义APT_MIRROR变量

    ARG APT_MIRROR=httpredir.debian.org

### 7. 替换/etc/app/sources.list文件中的"httpredir.debian.org"为变量APT_MIRROR

    RUN sed -i s/httpredir.debian.org/$APT_MIRROR/g /etc/apt/sources.list

### 8. 下载工具包

    RUN apt-get update && apt-get install -y       apparmor        apt-utils       aufs-tools      automake        bash-completion         binutils-mingw-w64      bsdmainutils    btrfs-tools  build-essential         clang   createrepo      curl    dpkg-sig        gcc-mingw-w64   git     iptables        jq      libapparmor-dev         libcap-dev      libltdl-dev     libsqlite3-dev       libsystemd-journal-dev  libtool         mercurial       net-tools       pkg-config      python-dev      python-mock     python-pip      python-websocket        ubuntu-zfs  xfsprogs         libzfs-dev      tar     zip     --no-install-recommends         && pip install awscli==1.10.15

### 9. 获得LVM2源码，用于静态编译。
设置LVM2的版本环境变量：

    ENV LVM2_VERSION 2.02.103

创建/usr/local/lvm2目录，下载LVM2.2.02.103.tgz，并解压到/usr/local/lvm2目录中

    RUN mkdir -p /usr/local/lvm2   && curl -fsSL "https://mirrors.kernel.org/sourceware/lvm2/LVM2.${LVM2_VERSION}.tgz"             | tar -xzC /usr/local/lvm2 --strip-components=1

### 10. 配置lvm2，编译并安装device-mapper

    RUN cd /usr/local/lvm2         && ./configure          --build="$(gcc -print-multiarch)"               --enable-static_link    && make device-mapper   && make install_device-mapper

### 11. 为OSX交叉编译配置容器的环境变量
设置环境变量：

    ENV OSX_SDK MacOSX10.11.sdk
    ENV OSX_CROSS_COMMIT 8aa9b71a394905e6c5f4b59e2b97b87a004658a4

导入OSXCROSS_PATH环境变量，并下载osxcross.git到"/osxcross"目录中，切换到 "OSX_CROSS_COMMIT"版本。下载MacOSX10.11.sdk.tar.gz到/osxcross/tarballs/MacOSX10.11.sdk.tar.gz中。设置OSX_VERSION_MIN环境变量，执行/osxcross/build.sh脚本编译osxcross。

    RUN set -x \
        && export OSXCROSS_PATH="/osxcross" \
        && git clone https://github.com/tpoechtrager/osxcross.git $OSXCROSS_PATH \
        && ( cd $OSXCROSS_PATH && git checkout -q $OSX_CROSS_COMMIT) \
        && curl -sSL https://s3.dockerproject.org/darwin/v2/${OSX_SDK}.tar.xz -o "${OSXCROSS_PATH}/tarballs/${OSX_SDK}.tar.xz" \
        && UNATTENDED=yes OSX_VERSION_MIN=10.6 ${OSXCROSS_PATH}/build.sh

设置PATH环境变量，加入osxcross编译后的二进制：

    ENV PATH /osxcross/target/bin:$PATH

### 12. 安装2.3.1版本的seccomp
seccomp（secure computing）是Linux内核支持的一种sandboxing机制。它能使进程进入一种“安全”运行模式。seccomp是Andrea Arcangeli在2005年设计的，目的是解决grid computing中的安全问题，比如你打算出租你的CPU资源，但又担心不可信会破坏你的系统。那么，seccomp可以为“不可信的纯计算型代码”提供一个安全的运行环境，以保护你的系统和应用程序的正常运行不受“不可信代码”的干扰。

    ENV SECCOMP_VERSION 2.3.1
    RUN set -x \
        && export SECCOMP_PATH="$(mktemp -d)" \
        && curl -fsSL "https://github.com/seccomp/libseccomp/releases/download/v${SECCOMP_VERSION}/libseccomp-${SECCOMP_VERSION}.tar.gz" \
            | tar -xzC "$SECCOMP_PATH" --strip-components=1 \
        && ( \
            cd "$SECCOMP_PATH" \
            && ./configure --prefix=/usr/local \
            && make \
            && make install \
            && ldconfig \
        ) \
        && rm -rf "$SECCOMP_PATH"

### 13. 安装Go编译环境
设置Golang版本

    ENV GO_VERSION 1.6.4

编译Go用于交叉编译

    ENV DOCKER_CROSSPLATFORMS \
        linux/386 linux/arm \
        darwin/amd64 \
        freebsd/amd64 freebsd/386 freebsd/arm \
        windows/amd64 windows/386

    RUN curl -fsSL "https://storage.googleapis.com/golang/go1.4.3.linux-amd64.tar.gz" \
        | tar -xzC /root && \
        mv /root/go /root/go1.4 && \
        cd /usr/local && \
        curl -fsSL "https://storage.googleapis.com/golang/go$GO_VERSION.src.tar.gz" \
        | tar -xzC /usr/local && \
        cd go && \
        printf 'diff --git a/src/runtime/sys_darwin_amd64.s b/src/runtime/sys_darwin_amd64.s\nindex e09b906..fa8ff2f 100644\n--- a/src/runtime/sys_darwin_amd64.s\n+++ b/src/runtime/sys_darwin_amd64.s\n@@ -157,6 +157,7 @@ systime:\n\t// Fall back to system call (usually first call in this thread).\n\tMOVQ\tSP, DI\n\tMOVQ\t$0, SI\n+\tMOVQ\t$0, DX  // required as of Sierra; Issue 16570\n\tMOVL\t$(0x2000000+116), AX\n\tSYSCALL\n\tCMPQ\tAX, $0\n' | patch -p1 && \
        cd src && \
        ./make.bash

设置PATH和GOPATH：

    ENV PATH /go/bin:/usr/local/go/bin:$PATH
    ENV GOPATH /go:/go/src/github.com/docker/docker/vendor

安装Go 工具包：

    //go cover and vet
    ENV GO_TOOLS_COMMIT 823804e1ae08dbb14eb807afc7db9993bc9e3cc3
    RUN git clone https://github.com/golang/tools.git /go/src/golang.org/x/tools \
	&& (cd /go/src/golang.org/x/tools && git checkout -q $GO_TOOLS_COMMIT) \
	&& go install -v golang.org/x/tools/cmd/cover \
	&& go install -v golang.org/x/tools/cmd/vet

    //go lint
    ENV GO_LINT_COMMIT 32a87160691b3c96046c0c678fe57c5bef761456
    RUN git clone https://github.com/golang/lint.git /go/src/github.com/golang/lint \
        && (cd /go/src/github.com/golang/lint && git checkout -q $GO_LINT_COMMIT) \
        && go install -v github.com/golang/lint/golint

### 14. 安装registry
安装2个版本的registry，下载distribution到临时目录，编译并安装二进制到容器中/usr/local/bin下面：

    ENV REGISTRY_COMMIT_SCHEMA1 ec87e9b6971d831f0eff752ddb54fb64693e51cd
    ENV REGISTRY_COMMIT 47a064d4195a9b56133891bbb13620c3ac83a827
    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone https://github.com/docker/distribution.git "$GOPATH/src/github.com/docker/distribution" \
        && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT") \
        && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH" \
            go build -o /usr/local/bin/registry-v2 github.com/docker/distribution/cmd/registry \
        && (cd "$GOPATH/src/github.com/docker/distribution" && git checkout -q "$REGISTRY_COMMIT_SCHEMA1") \
        && GOPATH="$GOPATH/src/github.com/docker/distribution/Godeps/_workspace:$GOPATH" \
            go build -o /usr/local/bin/registry-v2-schema1 github.com/docker/distribution/cmd/registry \
        && rm -rf "$GOPATH"

### 15. 安装notary和notary-server
notary的目标是保证server和client之间交互使用可信任的连接，解决互联网的内容发布的安全性。docker使用notary用于镜像源认证、镜像完整性等安全需求。

    ENV NOTARY_VERSION v0.3.0
    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone https://github.com/docker/notary.git "$GOPATH/src/github.com/docker/notary" \
        && (cd "$GOPATH/src/github.com/docker/notary" && git checkout -q "$NOTARY_VERSION") \
        && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH" \
            go build -o /usr/local/bin/notary-server github.com/docker/notary/cmd/notary-server \
        && GOPATH="$GOPATH/src/github.com/docker/notary/vendor:$GOPATH" \
            go build -o /usr/local/bin/notary github.com/docker/notary/cmd/notary \
        && rm -rf "$GOPATH"

### 16. 获取"docker-py"源码，用于运行它们的集成测试

    ENV DOCKER_PY_COMMIT 7befe694bd21e3c54bb1d7825270ea4bd6864c13
    RUN git clone https://github.com/docker/docker-py.git /docker-py \
        && cd /docker-py \
        && git checkout -q $DOCKER_PY_COMMIT \
        && pip install -r test-requirements.txt

### 17. 设置git的user.email

    RUN git config --global user.email 'docker-dummy@example.com'

### 18. 创建用户和组，用于测试
创建用户组docker，创建用户unprivilegeduser，创建home目录。

    RUN groupadd -r docker
    RUN useradd --create-home --gid docker unprivilegeduser

Dockerfile的VOLUME指令在镜像中创建挂载点/var/lib/docker。WORKDIR设置当前工作路径。ENV设置环境变量DOCKER_BUILDTAGS。

    VOLUME /var/lib/docker
    WORKDIR /go/src/github.com/docker/docker
    ENV DOCKER_BUILDTAGS apparmor pkcs11 seccomp selinux

### 19. 使用.bashrc文件
在当前路径中创建软连接：$PWD/.bashrc，连接到~/.bashrc

    RUN ln -sfv $PWD/.bashrc ~/.bashrc

### 20. 设置Docker的bash completion（命令补全）
docker的bash completion是指在docker命令后可以通过tab键来补全剩余的子命令或参数。

在当前路径中创建docker bash completion软连接：$PWD/contrib/completion/bash/docker，连接到/etc/bash_completion.d/docker

    RUN ln -sv $PWD/contrib/completion/bash/docker /etc/bash_completion.d/docker

### 21. 获取Hub images，我们就可以在本地使用"docker load"指令加载tar文件到docker中，而不需要使用docker pull拉去image了。

    COPY contrib/download-frozen-image-v2.sh /go/src/github.com/docker/docker/contrib/
    RUN ./contrib/download-frozen-image-v2.sh /docker-frozen-images \
        buildpack-deps:jessie@sha256:25785f89240fbcdd8a74bdaf30dd5599a9523882c6dfc567f2e9ef7cf6f79db6 \
        busybox:latest@sha256:e4f93f6ed15a0cdd342f5aae387886fba0ab98af0a102da6276eaf24d6e6ade0 \
        debian:jessie@sha256:f968f10b4b523737e253a97eac59b0d1420b5c19b69928d35801a6373ffe330e \
        hello-world:latest@sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7

### 22. 下载man page generator
下载go-md2man源码，编译并安装到/usr/local/bin中。

    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone --depth 1 -b v1.0.5 https://github.com/cpuguy83/go-md2man.git "$GOPATH/src/github.com/cpuguy83/go-md2man" \
        && git clone --depth 1 -b v1.4 https://github.com/russross/blackfriday.git "$GOPATH/src/github.com/russross/blackfriday" \
        && go get -v -d github.com/cpuguy83/go-md2man \
        && go build -v -o /usr/local/bin/go-md2man github.com/cpuguy83/go-md2man \
        && rm -rf "$GOPATH"

### 23. 下载toml validator
toml用于Go语言中的反射接口的解析和编解码，类似于Go原生的json和xml库。

下载toml源码，编译并安装到/usr/local/bin中。

    ENV TOMLV_COMMIT 9baf8a8a9f2ed20a8e54160840c492f937eeaf9a
    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone https://github.com/BurntSushi/toml.git "$GOPATH/src/github.com/BurntSushi/toml" \
        && (cd "$GOPATH/src/github.com/BurntSushi/toml" && git checkout -q "$TOMLV_COMMIT") \
        && go build -v -o /usr/local/bin/tomlv github.com/BurntSushi/toml/cmd/tomlv \
        && rm -rf "$GOPATH"

### 24. 安装runc
runc是容器标准化的产物，用于实际创建容器，管理容器生命周期。后面会详细介绍runc。

下载runc源码，编译并安装到/usr/local/bin/docker-runc

    ENV RUNC_COMMIT 50a19c6ff828c58e5dab13830bd3dacde268afe5
    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone https://github.com/docker/runc.git "$GOPATH/src/github.com/opencontainers/runc" \
        && cd "$GOPATH/src/github.com/opencontainers/runc" \
        && git checkout -q "$RUNC_COMMIT" \
        && make static BUILDTAGS="seccomp apparmor selinux" \
        && cp runc /usr/local/bin/docker-runc \
        && rm -rf "$GOPATH"

### 25. 安装containerd
containerd也是docker为了容器标准化而拆分出来的，是独立的二进制文件，用于管理容器创建的deamon程序。后面会详细介绍containerd。

下载containerd源码，编译并安装到/usr/local/bin/docker-containerd（daemon）和/usr/local/bin/docker-containerd-ctr（命令行）

    ENV CONTAINERD_COMMIT 2a5e70cbf65457815ee76b7e5dd2a01292d9eca8
    RUN set -x \
        && export GOPATH="$(mktemp -d)" \
        && git clone https://github.com/docker/containerd.git "$GOPATH/src/github.com/docker/containerd" \
        && cd "$GOPATH/src/github.com/docker/containerd" \
        && git checkout -q "$CONTAINERD_COMMIT" \
        && make static \
        && cp bin/containerd /usr/local/bin/docker-containerd \
        && cp bin/containerd-shim /usr/local/bin/docker-containerd-shim \
        && cp bin/ctr /usr/local/bin/docker-containerd-ctr \
        && rm -rf "$GOPATH"

### 26. 包装所有命令到hack/dind脚本中，允许内嵌的容器

    ENTRYPOINT ["hack/dind"]

### 27. 更新docker源码
把本地docker源码复制到容器的/go/src/github.com/docker/docker中。

    COPY . /go/src/github.com/docker/docker

**至此，docker image编译完成：**

    Successfully built 23ebda74733a

**下面开始运行docker image，执行hack/make.sh binary，生成二进制文件**

### 28. 生成docker相关的二进制文件
执行docker run，运行容器，参数包括：特权容器，各种环境变量，挂载卷（将宿主机上的"/root/mygo/src/github.com/docker/docker/bundles"目录挂载到容器的"/go/src/github.com/docker/docker/bundles"目录，这样编译生成的二进制文件就会在宿主机的"/root/mygo/src/github.com/docker/docker/bundles"目录中找到），运行之前docker build出来的镜像"docker-dev:HEAD"，执行hack/make.sh binary脚本。

    docker run --rm -i --privileged -e BUILDFLAGS -e KEEPBUNDLE -e DOCKER_BUILD_GOGC -e DOCKER_BUILD_PKGS -e DOCKER_DEBUG -e DOCKER_EXPERIMENTAL -e DOCKER_GITCOMMIT -e DOCKER_GRAPHDRIVER=devicemapper -e DOCKER_INCREMENTAL_BINARY -e DOCKER_REMAP_ROOT -e DOCKER_STORAGE_OPTS -e DOCKER_USERLANDPROXY -e TESTDIRS -e TESTFLAGS -e TIMEOUT -v "/root/mygo/src/github.com/docker/docker/bundles:/go/src/github.com/docker/docker/bundles" -t "docker-dev:HEAD" hack/make.sh binary

### 29. docker相关的二进制文件
docker编译后，会在/root/mygo/src/github.com/docker/docker/bundles下面生成1.12.6子目录，并且一个lastest软连接指向该子目录。

1.12.6子目录下面有bianry-client子目录（client）和binary-daemon子目录（daemon）。

    //注意，我是将docker源码放在/root/mygo/src/github.com/docker/docker目录中
    [root@localhost bundles]# ls -l /root/mygo/src/github.com/docker/docker/bundles/
    total 0
    drwxr-xr-x. 4 root root 48 Sep 23 08:57 1.12.6
    lrwxrwxrwx. 1 root root  6 Sep 23 08:55 latest -> 1.12.6

    [root@localhost bundles]# cd 1.12.6/
    [root@localhost 1.12.6]# ll
    total 4
    drwxr-xr-x. 2 root root   94 Sep 23 08:55 binary-client
    drwxr-xr-x. 2 root root 4096 Sep 23 08:57 binary-daemon

binary-client的内容：

    //可以看出有1个二进制文件：docker。
    [root@localhost 1.12.6]# cd binary-client/
    [root@localhost binary-client]# ll
    total 15320
    lrwxrwxrwx. 1 root root       13 Sep 23 08:55 docker -> docker-1.12.6
    -rwxr-xr-x. 1 root root 15675672 Sep 23 08:55 docker-1.12.6
    -rw-r--r--. 1 root root       48 Sep 23 08:55 docker-1.12.6.md5
    -rw-r--r--. 1 root root       80 Sep 23 08:55 docker-1.12.6.sha256

binary-daemon的内容：

    //可以看出有几个二进制文件：dockerd、docker-containerd、docker-containerd-ctr、docker-containerd-shim、docker-runc、docker-proxy。
    [root@localhost binary-client]# cd ../binary-daemon/
    [root@localhost binary-daemon]# ll
    total 80992
    -rwxr-xr-x. 1 root root 11291144 Sep 23 08:57 docker-containerd
    -rwxr-xr-x. 1 root root 10537472 Sep 23 08:57 docker-containerd-ctr
    -rw-r--r--. 1 root root       56 Sep 23 08:57 docker-containerd-ctr.md5
    -rw-r--r--. 1 root root       88 Sep 23 08:57 docker-containerd-ctr.sha256
    -rw-r--r--. 1 root root       52 Sep 23 08:57 docker-containerd.md5
    -rw-r--r--. 1 root root       84 Sep 23 08:57 docker-containerd.sha256
    -rwxr-xr-x. 1 root root  3831592 Sep 23 08:57 docker-containerd-shim
    -rw-r--r--. 1 root root       57 Sep 23 08:57 docker-containerd-shim.md5
    -rw-r--r--. 1 root root       89 Sep 23 08:57 docker-containerd-shim.sha256
    lrwxrwxrwx. 1 root root       14 Sep 23 08:57 dockerd -> dockerd-1.12.6
    -rwxr-xr-x. 1 root root 45575312 Sep 23 08:57 dockerd-1.12.6
    -rw-r--r--. 1 root root       49 Sep 23 08:57 dockerd-1.12.6.md5
    -rw-r--r--. 1 root root       81 Sep 23 08:57 dockerd-1.12.6.sha256
    lrwxrwxrwx. 1 root root       19 Sep 23 08:57 docker-proxy -> docker-proxy-1.12.6
    -rwxr-xr-x. 1 root root  2879456 Sep 23 08:57 docker-proxy-1.12.6
    -rw-r--r--. 1 root root       54 Sep 23 08:57 docker-proxy-1.12.6.md5
    -rw-r--r--. 1 root root       86 Sep 23 08:57 docker-proxy-1.12.6.sha256
    -rwxr-xr-x. 1 root root  8764120 Sep 23 08:57 docker-runc
    -rw-r--r--. 1 root root       46 Sep 23 08:57 docker-runc.md5
    -rw-r--r--. 1 root root       78 Sep 23 08:57 docker-runc.sha256


## 参考文献
* [[编译docker源码的完整输出]](../reference/docker-build.md)

_______________________________________________________________________
[[返回README.md]](../README.md) 

