daemonCli 启动
=========================================================
## 简介
daemon启动的核心代码。

## start综述
含义：

    docker daemon启动。重点研究容器的创建。

路径：

    github.com/docker/docker/cmd/dockerd/daemon.go

定义：

    func (cli *DaemonCli) start() (err error) {
        //stopc是一个channel，作用是让进程不退出，因为daemon要作为后台进程，一直运行，当遇到信号时可以使用stopc退出进程。
        stopc := make(chan bool)
        defer close(stopc)

        // 当运行daemon时，来自uuid包的警告。
        uuid.Loggerf = logrus.Warnf

        flags := flag.CommandLine

        //在上节中介绍了commonFlags.PostParse = func() { postParseCommon(commonFlags) }，这里的PostParse就是前面注册的函数。主要用来设置日志级别和TLS参数。
        cli.commonFlags.PostParse()

        //之前没有设置过TrustKey，因此这里设置：/etc/docker/key.json
        //在前面的centos例子中，/etc/docker/key.json的内容如下：
        //{"crv":"P-256","d":"-y8NC73lL3eHjYWA5PkeuLYXF12tMa_MInhvxRtEass","kid":"LFO2:F34B:M2US:PL24:Y6YH:NRJX:UNW2:GYCN:JLEX:CXNI:N2EZ:K3UK","kty":"EC","x":"tWe7PMJ0fEo6AgCl8S74iXEPwM43923DrtJfZZD-Mz4","y":"C1qlV1Yoy7fZL5vdN3tIxqvzlEBmiio0musfJLN4I3I"}
        if cli.commonFlags.TrustKey == "" {
            cli.commonFlags.TrustKey = filepath.Join(getDaemonConfDir(), cliflags.DefaultTrustKeyFile)
        }

        //合并到config配置中，并验证config配置的参数合法性。Config、commonFlags和configFile都是前面介绍的daemonCli.Config的字段。
        cliConfig, err := loadDaemonCliConfig(cli.Config, flags, cli.commonFlags, *cli.configFile)
        if err != nil {
            return err
        }
        cli.Config = cliConfig

        //设置debug
        if cli.Config.Debug {
            utils.EnableDebug()
        }

        //运行实验性编译。
        if utils.ExperimentalBuild() {
            logrus.Warn("Running experimental build")
        }

        //设置日志文本格式：时间戳格式，着色。
        logrus.SetFormatter(&logrus.TextFormatter{
            TimestampFormat: jsonlog.RFC3339NanoFixed,
            DisableColors:   cli.Config.RawLogs,
        })

        //设置系统默认掩码：0022
        if err := setDefaultUmask(); err != nil {
            return fmt.Errorf("Failed to set umask: %v", err)
        }

        //日志配置了日志参数，则需要验证日志参数
        if len(cli.LogConfig.Config) > 0 {
            if err := logger.ValidateLogOpts(cli.LogConfig.Type, cli.LogConfig.Config); err != nil {
                return fmt.Errorf("Failed to set log opts: %v", err)
            }
        }

        //进程退出后移除进程pid。cli.Pidfile的默认值是：/var/run/docker.pid，用户可以通过命令行参数："-p"或"--pidfile"来修改该默认值。
        if cli.Pidfile != "" {
            //创建PIDFile，往/var/run/docker.pid中写pid。
            pf, err := pidfile.New(cli.Pidfile)
            if err != nil {
                return fmt.Errorf("Error starting daemon: %v", err)
            }
            defer func() {
                //daemon退出时删除PIDFile
                if err := pf.Remove(); err != nil {
                    logrus.Error(err)
                }
            }()
        }

        //设置apiserver的Config配置信息。
        serverConfig := &apiserver.Config{
            Logging:     true,
            SocketGroup: cli.Config.SocketGroup,   //socket组，默认值："docker"。
            Version:     dockerversion.Version,    //docker版本
            EnableCors:  cli.Config.EnableCors,    //是否允许跨域资源共享
            CorsHeaders: cli.Config.CorsHeaders,   //跨域资源共享的资源列表。
        }

        //设置apiserver的TLS。
        if cli.Config.TLS {
            tlsOptions := tlsconfig.Options{
                CAFile:   cli.Config.CommonTLSOptions.CAFile,
                CertFile: cli.Config.CommonTLSOptions.CertFile,
                KeyFile:  cli.Config.CommonTLSOptions.KeyFile,
            }

            if cli.Config.TLSVerify {
                // server requires and verifies client's certificate
                tlsOptions.ClientAuth = tls.RequireAndVerifyClientCert
            }
            tlsConfig, err := tlsconfig.Server(tlsOptions)
            if err != nil {
                return err
            }
            serverConfig.TLSConfig = tlsConfig
        }

        //设置socket绑定的hosts。
        if len(cli.Config.Hosts) == 0 {
            cli.Config.Hosts = make([]string, 1)
        }

        //根据serverconfig创建apiserver
        api := apiserver.New(serverConfig)
        cli.api = api

        //如果设置了Config.Hosts，则执行下面操作。默认Config.Hosts为空。
        for i := 0; i < len(cli.Config.Hosts); i++ {
            var err error
            if cli.Config.Hosts[i], err = opts.ParseHost(cli.Config.TLS, cli.Config.Hosts[i]); err != nil {
                return fmt.Errorf("error parsing -H %s : %v", cli.Config.Hosts[i], err)
            }

            //解析host，分解为协议、ip、port。
            protoAddr := cli.Config.Hosts[i]
            protoAddrParts := strings.SplitN(protoAddr, "://", 2)
            if len(protoAddrParts) != 2 {
                return fmt.Errorf("bad format %s, expected PROTO://ADDR", protoAddr)
            }

            proto := protoAddrParts[0]
            addr := protoAddrParts[1]

            // 如果要绑定TCP，那么记得一定要设置TLS。
            if proto == "tcp" && (serverConfig.TLSConfig == nil || serverConfig.TLSConfig.ClientAuth != tls.RequireAndVerifyClientCert) {
                logrus.Warn("[!] DON'T BIND ON ANY IP ADDRESS WITHOUT setting -tlsverify IF YOU DON'T KNOW WHAT YOU'RE DOING [!]")
            }

            //为api server创建listener，调用sockets.NewTCPSocket(addr, tlsConfig)
            ls, err := listeners.Init(proto, addr, serverConfig.SocketGroup, serverConfig.TLSConfig)
            if err != nil {
                return err
            }

            //还是返回listener
            ls = wrapListeners(proto, ls)

            // 如果绑定tcp端口，则要确保容器不能使用它。
            if proto == "tcp" {
                if err := allocateDaemonPort(addr); err != nil {
                    return err
                }
            }
            logrus.Debugf("Listener created for HTTP on %s (%s)", protoAddrParts[0], protoAddrParts[1])

            //返回监听的api server服务器
            api.Accept(protoAddrParts[1], ls...)
        }

        //迁移keyfile，将环境变量"DOCKER_CONFIG"指定目录下的key.json迁移到/etc/docker/key.json下面。
        if err := migrateKey(); err != nil {
            return err
        }

        //TrustKeyPath=/etc/docker/key.json
        cli.TrustKeyPath = cli.commonFlags.TrustKey

        //参数ServiceOptions前面已经介绍过了，是关于registry的参数。创建一个默认service对象，准备被安装到引擎中。
        registryService := registry.NewService(cli.Config.ServiceOptions)

        //创建containerd，containerd负责创建容器，管理容器生命周期，后面会详细介绍。
        containerdRemote, err := libcontainerd.New(cli.getLibcontainerdRoot(), cli.getPlatformRemoteOptions()...)
        if err != nil {
            return err
        }
        cli.api = api
        signal.Trap(func() {
            cli.stop()
            <-stopc //等着daemonCli.start() 返回
        })

        //启动daemon，这里是创建docker daemon的核心程序。在下一章节详细介绍创建daemon的过程。
        d, err := daemon.NewDaemon(cli.Config, registryService, containerdRemote)
        if err != nil {
            return fmt.Errorf("Error starting daemon: %v", err)
        }

        name, _ := os.Hostname()

        c, err := cluster.New(cluster.Config{
            Root:                   cli.Config.Root,
            Name:                   name,
            Backend:                d,
            NetworkSubnetsProvider: d,
            DefaultAdvertiseAddr:   cli.Config.SwarmDefaultAdvertiseAddr,
        })
        if err != nil {
            logrus.Fatalf("Error creating cluster component: %v", err)
        }

        logrus.Info("Daemon has completed initialization")

        logrus.WithFields(logrus.Fields{
            "version":     dockerversion.Version,
            "commit":      dockerversion.GitCommit,
            "graphdriver": d.GraphDriverName(),
        }).Info("Docker daemon")

        cli.initMiddlewares(api, serverConfig)
        initRouter(api, d, c)

        cli.d = d
        cli.setupConfigReloadTrap()

        // The serve API routine never exits unless an error occurs
        // We need to start it as a goroutine and wait on it so
        // daemon doesn't exit
        serveAPIWait := make(chan error)
        go api.Wait(serveAPIWait)

        // after the daemon is done setting up we can notify systemd api
        notifySystem()

        // Daemon is fully initialized and handling API traffic
        // Wait for serve API to complete
        errAPI := <-serveAPIWait
        c.Cleanup()
        shutdownDaemon(d, 15)
        containerdRemote.Cleanup()
        if errAPI != nil {
            return fmt.Errorf("Shutting down due to ServeAPI error: %v", errAPI)
        }

        return nil
    }

### 1. cli.commonFlags.PostParse
含义：

    在上节中介绍了commonFlags.PostParse = func() { postParseCommon(commonFlags) }，这里的PostParse就是前面注册的函数。主要用来设置日志级别和TLS参数。

路径：

    github.com/docker/docker/cli/flags/common.go

定义：

    func postParseCommon(commonFlags *CommonFlags) {
        cmd := commonFlags.FlagSet

        //设置daemon日志级别
        SetDaemonLogLevel(commonFlags.LogLevel)

        //如果设置了"--tlsverify=true"，则设置commonFlags.TLS=true，使用TLS验证连接。
        if cmd.IsSet("-"+TLSVerifyKey) || commonFlags.TLSVerify {
            commonFlags.TLS = true
        }

        if !commonFlags.TLS {
            commonFlags.TLSOptions = nil
        } else {
            tlsOptions := commonFlags.TLSOptions
            tlsOptions.InsecureSkipVerify = !commonFlags.TLSVerify

            // 如果没有设置"--tlscert"，则重新设置为空。
            if !cmd.IsSet("-tlscert") {
                if _, err := os.Stat(tlsOptions.CertFile); os.IsNotExist(err) {
                    tlsOptions.CertFile = ""
                }
            }
            // 如果没有设置"--tlskey"，则重新设置为空。
            if !cmd.IsSet("-tlskey") {
                if _, err := os.Stat(tlsOptions.KeyFile); os.IsNotExist(err) {
                    tlsOptions.KeyFile = ""
                }
            }
        }
    }

### 2. loadDaemonCliConfig
含义：

    合并到DaemonCli.Config配置中，并验证DaemonCli.Config配置的参数合法性。

路径：

    github.com/docker/docker/cmd/dockerd/daemon.go

定义：

    func loadDaemonCliConfig(config *daemon.Config, flags *flag.FlagSet, commonConfig *cliflags.CommonFlags, configFile string) (*daemon.Config, error) {
        //将DaemonCli.commonFlags中的参数添加到DaemonCli.Config中。
        config.Debug = commonConfig.Debug                                           //默认为false，用户可以通过命令行参数传入："--debug"来改变该默认值。
        config.Hosts = commonConfig.Hosts                                           //默认为空串，用户可以通过命令行参数传入："-H"或"--hosts"来改变该默认值。如果为空，则不会通过parseDockerDaemonHost来验证socket绑定的主机。
        config.LogLevel = commonConfig.LogLevel                                     //默认为info，用户可以通过命令行参数传入："-l"或"--log-level"来改变该默认值。
        config.TLS = commonConfig.TLS                                               //默认为false，用户可以通过命令行参数传入："--tls"来改变该默认值。
        config.TLSVerify = commonConfig.TLSVerify                                   //默认值由环境变量DOCKER_TLS_VERIFY来指定，用户可以通过命令行参数传入："--tlsverify"来改变该默认值。
        config.CommonTLSOptions = daemon.CommonTLSOptions{}

        if commonConfig.TLSOptions != nil {
            config.CommonTLSOptions.CAFile = commonConfig.TLSOptions.CAFile         //默认为dockerCertPath/ca.pem，用户可以通过命令行参数传入："--tlscacert"来改变该默认值。dockerCertPath是环境变量DOCKER_CERT_PATH的值或者是DOCKER_CONFIG的值。
            config.CommonTLSOptions.CertFile = commonConfig.TLSOptions.CertFile     //默认为dockerCertPath/cert.pem，用户可以通过命令行参数传入："--tlscert"来改变该默认值。
            config.CommonTLSOptions.KeyFile = commonConfig.TLSOptions.KeyFile       //默认为dockerCertPath/key.pem，用户可以通过命令行参数传入："--tlskey"来改变该默认值。
        }

        //加载DaemonCli.configFile的信息到DaemonCli.Config中。默认值为空，用户可以通过命令行参数传入："--config-file"来改变该默认值。
        if configFile != "" {
            c, err := daemon.MergeDaemonConfigurations(config, flags, configFile)
            if err != nil {
                if flags.IsSet(daemonConfigFileFlag) || !os.IsNotExist(err) {
                    return nil, fmt.Errorf("unable to configure the Docker daemon with file %s: %v\n", configFile, err)
                }
            }
            if c != nil {
                config = c
            }
        }

        //验证配置DaemonCli.Config，包括：config.DNS（DNS是否是有效ip地址），config.DNSSearch（DNS搜索域是否有效），config.Labels（标签是否有效），config.MaxConcurrentDownloads（是否超过了最大pull并发数量），config.MaxConcurrentUploads（是否超过了最大push并发数量）
        if err := daemon.ValidateConfiguration(config); err != nil {
            return nil, err
        }

        // 验证是否设置TLSVerifyKey，其实TLSVerifyKey的值就是"tlsverify"
        if config.IsValueSet(cliflags.TLSVerifyKey) {
            config.TLS = true
        }

        // 设置日志级别
        cliflags.SetDaemonLogLevel(config.LogLevel)

        return config, nil
    }

### 3. apiserver.New
含义：

    用指定的serverConfig参数创建一个apiserver对象。

路径：

    github.com/docker/docker/api/server/server.go

定义：

    func New(cfg *Config) *Server {
        return &Server{
            cfg: cfg,
        }
    }

### 4. registry.NewService
含义：

    创建一个registry service对象。

定义：

    //github.com/docker/docker/registry/service.go
    func NewService(options ServiceOptions) *DefaultService {
        return &DefaultService{
            config: newServiceConfig(options),
        }
    }

    //创建service config，主要是设置命令行参数"--insecure-registry"指定的registry和官方默认的registry："docker.io"。
    //github.com/docker/docker/registry/config.go
    func newServiceConfig(options ServiceOptions) *serviceConfig {
        // 本地默认是insecure registry
        options.InsecureRegistries = append(options.InsecureRegistries, "127.0.0.0/8")

        config := &serviceConfig{
            ServiceConfig: registrytypes.ServiceConfig{
                InsecureRegistryCIDRs: make([]*registrytypes.NetIPNet, 0),
                IndexConfigs:          make(map[string]*registrytypes.IndexInfo, 0),
                Mirrors: options.Mirrors,
            },
            V2Only: options.V2Only,
        }
        // 解析"--insecure-registry"到CIDR和registry指定的配置中。在前面介绍的centos例子中，在/etc/sysconfig/docker"配置文件中的OPTIONS中可以添加命令行参数："--insecure-registry"
        for _, r := range options.InsecureRegistries {
            // 解析CIDR
            _, ipnet, err := net.ParseCIDR(r)
            if err == nil {
                // 有效的CIDR
                config.InsecureRegistryCIDRs = append(config.InsecureRegistryCIDRs, (*registrytypes.NetIPNet)(ipnet))
            } else {
                config.IndexConfigs[r] = &registrytypes.IndexInfo{
                    Name:     r,
                    Mirrors:  make([]string, 0),
                    Secure:   false,
                    Official: false,
                }
            }
        }

        // 配置 public registry。IndexName="docker.io"，作为默认的registry。
        config.IndexConfigs[IndexName] = &registrytypes.IndexInfo{
            Name:     IndexName,
            Mirrors:  config.Mirrors,
            Secure:   true,
            Official: true,
        }

        return config
    }

### 5. libcontainerd.New
含义：

    创建libcontainerd，libcontainerd负责创建容器，管理容器生命周期，后面会详细说明。返回值也作为后面的daemon.NewDaemon的参数传入。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

参数：

    cli.getLibcontainerdRoot()      - /var/run/docker/libcontainerd
    cli.getPlatformRemoteOptions()  - libcontainer配置，下面会详细解释。

定义：

    //参数：stateDir="/var/run/docker/libcontainerd"， options=remote结构体。
    func New(stateDir string, options ...RemoteOption) (_ Remote, err error){
        defer func() {
            if err != nil {
                err = fmt.Errorf("Failed to connect to containerd. Please make sure containerd is installed in your PATH or you have specificed the correct address. Got error: %v", err)
            }
        }()

        //创建remote结构体
        r := &remote{
            stateDir:    stateDir,                                         // /var/run/docker/libcontainerd
            daemonPid:   -1,                            
            eventTsPath: filepath.Join(stateDir, eventTimestampFilename),  // /var/run/docker/libcontainerd/event.ts
        }
        //将参数赋给新创建的remote结构体，分别调用参数的Apply方法。
        for _, option := range options {
            if err := option.Apply(r); err != nil {
                return nil, err
            }
        }

        //创建目录：/var/run/docker/libcontainerd，权限0700
        if err := sysinfo.MkdirAll(stateDir, 0700); err != nil {
            return nil, err
        }

        //设置remote的rpcAddr参数，如果rpcAddr参数不存在，则设置为/var/run/docker/libcontainerd/docker-containerd.sock，该socket文件是docker-containerd进程的监听socket。默认是不存在。
        if r.rpcAddr == "" {
            r.rpcAddr = filepath.Join(stateDir, containerdSockFilename)
        }
        //默认设置startDaemon=true。这里创建docker-containerd进程，这是创建容器的核心程序，在下节中详细介绍。
        if r.startDaemon {
            if err := r.runContainerdDaemon(); err != nil {   
                return nil, err
            }
        }

        
        // 不输出grpc重连日志
        grpclog.SetLogger(log.New(ioutil.Discard, "", log.LstdFlags))
        dialOpts := append([]grpc.DialOption{grpc.WithInsecure()},
            grpc.WithDialer(func(addr string, timeout time.Duration) (net.Conn, error) {
                return net.DialTimeout("unix", addr, timeout)
            }),
        )

        //下面是dockerd对docker-containerd的rpc调用。Dial是对rpcAddr（/var/run/docker/libcontainerd/docker-containerd.sock）拨号，返回一个连接conn。
        conn, err := grpc.Dial(r.rpcAddr, dialOpts...)
        if err != nil {
            return nil, fmt.Errorf("error connecting to containerd: %v", err)
        }

        //后面做的事情都是在监听docker-containerd进程的事件，连接情况等。

        //设置remote的rpcConn参数
        r.rpcConn = conn

        //设置remote的apiClient参数
        r.apiClient = containerd.NewAPIClient(conn)

        // 获取上次事件的时间戳，读 /var/run/docker/libcontainerd/event.ts文件
        t := r.getLastEventTimestamp()
        //将上次事件时间戳写到protobuf结构中。
        tsp, err := ptypes.TimestampProto(t)
        if err != nil {
            logrus.Errorf("libcontainerd: failed to convert timestamp: %q", err)
        }
        //设置remote的restoreFromTimestamp
        r.restoreFromTimestamp = tsp

        //处理连接变化的goroutine
        go r.handleConnectionChange()

        //启动事件监控
        if err := r.startEventsMonitor(); err != nil {
            return nil, err
        }

        return r, nil
    }

### 5.1 grpc

这里主要介绍一下grpc的原理和在docker中的使用。

    GRPC是google开源的一个高性能RPC框架，基于HTTP2协议，基于protobuf 3.x（高性能数据传输格式），基于Netty 4.x+。从开发角度说，你需要做以下步骤（以java为例）：
        a）编写.proto文件，protobuf在其他文章中已经介绍了，这里就不详细介绍了。
        b）使用编译工具来编译.proto文件。
        c）启动一个server端，监听一个端口，等待client连接，通常用Netty构建
        d）启动一个client端，也是通过Netty构建，client与server建立TCP长连接，并发送请求。request和response都是HTTP2，通过Netty channel进行交互。



### 5.2 参数cli.getPlatformRemoteOptions()
含义：

    实现RemoteOptions接口的结构体数组，即实现了Apply方法，这些方法在后面会被调用。

路径：

    cmd/dockerd/daemon_unix.go

定义：

    func (cli *DaemonCli) getPlatformRemoteOptions() []libcontainerd.RemoteOption {
        opts := []libcontainerd.RemoteOption{
            libcontainerd.WithDebugLog(cli.Config.Debug),                                  //定义容器的debug log是否对daemon有效。根据cli.Config.Debug的值设置remote结构体的debugLog参数。
            libcontainerd.WithOOMScore(cli.Config.OOMScoreAdjust),
        }
        if cli.Config.ContainerdAddr != "" {                                               //默认为空
            opts = append(opts, libcontainerd.WithRemoteAddr(cli.Config.ContainerdAddr))   
        } else {
            opts = append(opts, libcontainerd.WithStartDaemon(true))
        }
        if daemon.UsingSystemd(cli.Config) {                                               //如果cli.Config.ExecOptions中包含"native.cgroupdriver=systemd"，则执行下面操作。在前面的centos例子中，设置了"native.cgroupdriver=systemd"。
            args := []string{"--systemd-cgroup=true"}
            opts = append(opts, libcontainerd.WithRuntimeArgs(args))
        }
        if cli.Config.LiveRestore {
            opts = append(opts, libcontainerd.WithLiveRestore(true))
        }
        opts = append(opts, libcontainerd.WithRuntimePath(daemon.DefaultRuntimeBinary))
        return opts
    }

（1）remote结构体

    //libcontainerd/remote_linux.go
    type remote struct {
        sync.RWMutex
        apiClient            containerd.APIClient
        daemonPid            int
        stateDir             string
        rpcAddr              string
        startDaemon          bool
        closeManually        bool
        debugLog             bool
        rpcConn              *grpc.ClientConn
        clients              []*client
        eventTsPath          string
        runtime              string
        runtimeArgs          []string
        daemonWaitCh         chan struct{}
        liveRestore          bool
        oomScore             int
        restoreFromTimestamp *timestamp.Timestamp
    }

（2）根据cli.Config.Debug的值设置remote结构体的debugLog参数

    func WithDebugLog(debug bool) RemoteOptions {
        return debugLog(debug)
    }
    //debugLog实现RemoteOptions接口
    type debugLog bool
    func (d debugLog) Apply(r Remote) error {
        if remote, ok := r.(*remote); ok {
            remote.debugLog = bool(d)                   //设置remote结构体的debugLog参数
            return nil
        }
        return fmt.Errorf("WithDebugLog options not supported for this remote")
    }

（3）根据cli.Config.OOMScoreAdjust的值设置remote结构体的oomScore参数

    func WithOOMScore(score int) RemoteOptions {
        return oomScore(score)
    }
    type oomScore int
    func (o oomScore) Apply(r Remote) error {
        if remote, ok:=r.(*remote); ok {
            remote.oomScore = int(o)
            return nil
        }
        return fmt.Errorf("WithOOMScore options not support for this remote")
    }

（4）当cli.Config.ContainerdAddr值存在，则根据cli.Config.ContainerdAddr的值设置remote结构体的rpcAddr参数

    func WithRemoteAddr(addr string) RemoteOptions {
        return rpcAddr(addr)
    }
    type rpcAddr string
    func (a rpcAddr) Apply(r Remote) error {
        if remote, ok:=r.(*remote); ok {
            remote.rpcAddr=string(a)
            return nil
        }
        return fmt.Errorf("WithRemoteAddr option not supported for this remote")
    }

    当cli.Config.ContainerdAddr值不存在，则设置remote的startDaemon参数为true。
    func WithStartDaemon(start bool) RemoteOptions {
        return startDaemon(start)
    }
    type startDaemon bool
    func (s startDaemon) Apply(r Remote) error {
        if remote, ok := r.(*remote) ok {
            remote.startDaemon = bool(s)
            return nil
        }
        return fmt.Errorf("WithStartDaemon options not supported for this remote")
    }

    注意：默认ContainerAddr为""，但是可以设置命令行flag："--containerd"来改变默认值。在前面的centos例子中，没有设置"--containerd"，即会设置startDaemon=true，为后面执行docker-containerd做准备。

（5）如果cli options包含native.cgroupdriver=systemd，则返回true

    //daemon/daemon_unix.go
    func UsingSystemd(config *Config) bool {
        return getCD(config) == cgroupSystemdDriver            //"systemd"
    }
    func getCD(config *Config) string {
        for _, option := range config.ExecOptions {
            key, val, err := parsers.ParseKeyValueOpt(option)
            if err != nil || !strings.EqualFold(key, "native.cgroupdriver") {
                continue
            }
            return val
        }
        return ""
    }

    如果UsingSystemd返回true，则使用"--systemd-cgroup=true"参数来设置remote的runtimeArgs参数
    func WithRuntimeArgs(args []string) RemoteOption {
        return runtimeArgs(args)
    }
    type runtimeArgs []string
    func (rt runtimeArgs) Apply(r Remote) error {
        if remote, ok:=r.(*remote); ok{
            remote.runtimeArgs = rt
            return nil
        }
        return fmt.Errorf("WithRuntimeArgs options not supported for this remote")
    }

（6）如果cli.Config.LiveRestore值为true，则设置remote的liveRestore参数为true。LiveRestore表示当容器正在运行时，docker是否允许在线恢复，默认是false。

    func WithLiveRestore(v bool) RemoteOption {
        return liveRestore(v)
    }
    type liveRestore bool
    func (l liveRestore) Apply(r Remote) error{
        if remote,ok:=r.(*remote);ok{
            remote.liveRestore=bool(l)
            for _,c:=range remote.clients{
                c.liveRestore=bool(l)
            }
            return nil
        }
        return fmt.Errorf("WithLiveRestore option not supported for this remote")
    }

（7）设置remote的runtime参数为"docker-runc"

    func WithRuntimePath(rt string) RemoteOptions {
        return runtimePath(rt)
    }
    type runtimePath string
    func (rt runtimePath) Apply(r Remote) error {
        if remote, ok:=r.(*remote); ok {
            remote.runtime=string(rt)
            return nil
        }
        return fmt.Errorf("WithRuntime option not supported for this remote")
    }


### 5.3 handleConnectionChange

含义：

    处理连接变化。这是实验性API。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

定义：

    func (r *remote) handleConnectionChange() {
        var transientFailureCount = 0
        state := grpc.Idle
        for {
            //这是实验性API，等待容器连接状态改变
            s, err := r.rpcConn.WaitForStateChange(context.Background(), state)
            if err != nil {
                break
            }
            state = s
            logrus.Debugf("libcontainerd: containerd connection state change: %v", s)

            //daemonPid!=-1时才执行下面操作，前面设置了remote.daemon=-1，但是在运行containerd的时候，有可能会设置-1。
            if r.daemonPid != -1 {
                switch state {
                case grpc.TransientFailure:
                    // Reset state to be notified of next failure
                    transientFailureCount++
                    if transientFailureCount >= maxConnectionRetryCount {
                        transientFailureCount = 0
                        if utils.IsProcessAlive(r.daemonPid) {
                            utils.KillProcess(r.daemonPid)
                        }
                        <-r.daemonWaitCh
                        if err := r.runContainerdDaemon(); err != nil { //FIXME: Handle error
                            logrus.Errorf("libcontainerd: error restarting containerd: %v", err)
                        }
                    } else {
                        state = grpc.Idle
                        time.Sleep(connectionRetryDelay)
                    }
                case grpc.Shutdown:
                    // Well, we asked for it to stop, just return
                    return
                }
            }
        }
    }

### 5.4 startEventsMonitor

含义：

    启动事件监控。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

定义：

    func (r *remote) startEventsMonitor() error {
        // 获得上一次事件时间戳
        t := r.getLastEventTimestamp()

        //转换为protobuf的时间戳
        tsp, err := ptypes.TimestampProto(t)
        if err != nil {
            logrus.Errorf("libcontainerd: failed to convert timestamp: %q", err)
        }
        er := &containerd.EventsRequest{
            Timestamp: tsp,
        }

        //获得事件
        events, err := r.apiClient.Events(context.Background(), er)
        if err != nil {
            return err
        }

        //处理事件
        go r.handleEventStream(events)
        return nil
    }



_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 

