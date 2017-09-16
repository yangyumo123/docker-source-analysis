daemonCli 启动
=========================================================
## 简介
daemon启动的核心代码。

## start综述
含义：

    docker daemon启动。

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
        if cli.commonFlags.TrustKey == "" {
            cli.commonFlags.TrustKey = filepath.Join(getDaemonConfDir(), cliflags.DefaultTrustKeyFile)
        }

        //加载daemon client config。configFile是flag参数："--config-file"。下面详细介绍。
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

        //进程退出后移除进程pid。
        if cli.Pidfile != "" {
            pf, err := pidfile.New(cli.Pidfile)
            if err != nil {
                return fmt.Errorf("Error starting daemon: %v", err)
            }
            defer func() {
                if err := pf.Remove(); err != nil {
                    logrus.Error(err)
                }
            }()
        }

        //设置api server的Config配置信息。
        serverConfig := &apiserver.Config{
            Logging:     true,
            SocketGroup: cli.Config.SocketGroup,   //socket组
            Version:     dockerversion.Version,    //docker版本
            EnableCors:  cli.Config.EnableCors,    //是否允许跨域资源共享
            CorsHeaders: cli.Config.CorsHeaders,   //跨域资源共享的资源列表。
        }

        //设置server的TLS。
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

        //客户端hosts
        if len(cli.Config.Hosts) == 0 {
            cli.Config.Hosts = make([]string, 1)
        }

        //根据serverconfig创建api server
        api := apiserver.New(serverConfig)
        cli.api = api

        //如果设置了Config.Hosts，则执行下面操作。
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

            //返回监听到的api server服务器
            api.Accept(protoAddrParts[1], ls...)
        }

        //迁移keyfile，将环境变量"DOCKER_CONFIG"指定目录下的key.json迁移到/etc/docker/key.json下面。
        if err := migrateKey(); err != nil {
            return err
        }

        //TrustKeyPath就是/etc/docker/key.json
        cli.TrustKeyPath = cli.commonFlags.TrustKey

        //参数ServiceOptions前面已经介绍过了，是关于registry的参数。创建一个默认service对象，准备被安装到引擎中。
        registryService := registry.NewService(cli.Config.ServiceOptions)

        //创建容器配置
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

    合并到config配置中，并验证config配置的参数合法性。

路径：

    github.com/docker/docker/cmd/dockerd/daemon.go

定义：

    func loadDaemonCliConfig(config *daemon.Config, flags *flag.FlagSet, commonConfig *cliflags.CommonFlags, configFile string) (*daemon.Config, error) {
        //将commonConfig中的参数添加到config中。
        config.Debug = commonConfig.Debug
        config.Hosts = commonConfig.Hosts
        config.LogLevel = commonConfig.LogLevel
        config.TLS = commonConfig.TLS
        config.TLSVerify = commonConfig.TLSVerify
        config.CommonTLSOptions = daemon.CommonTLSOptions{}

        if commonConfig.TLSOptions != nil {
            config.CommonTLSOptions.CAFile = commonConfig.TLSOptions.CAFile
            config.CommonTLSOptions.CertFile = commonConfig.TLSOptions.CertFile
            config.CommonTLSOptions.KeyFile = commonConfig.TLSOptions.KeyFile
        }

        //加载configFile的信息到config中。
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

        //验证配置config，包括：config.DNS（DNS是否是有效ip地址），config.DNSSearch（DNS搜索域是否有效），config.Labels（标签是否有效），config.MaxConcurrentDownloads（是否超过了最大pull并发数量），config.MaxConcurrentUploads（是否超过了最大push并发数量）
        if err := daemon.ValidateConfiguration(config); err != nil {
            return nil, err
        }

        // 验证是否设置TLSVerifyKey
        if config.IsValueSet(cliflags.TLSVerifyKey) {
            config.TLS = true
        }

        // 设置日志级别
        cliflags.SetDaemonLogLevel(config.LogLevel)

        return config, nil
    }

### 3. apiserver.New
含义：

    用指定config参数创建一个server对象，
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

路径：

    github.com/docker/docker/registry/service.go

定义：

    func NewService(options ServiceOptions) *DefaultService {
        return &DefaultService{
            config: newServiceConfig(options),
        }
    }

    //创建service config
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
        // 解析"--insecure-registry"到CIDR和registry指定的配置中。
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

        // 配置 public registry.
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

    创建libcontainerd实例。

路径：

    github.com/docker/docker/libcontainerd/remote_linux.go

参数：

    cli.getLibcontainerdRoot()      - /var/run/docker/libcontainerd
    cli.getPlatformRemoteOptions()  - libcontainer配置

定义：

    func New(stateDir string, options ...RemoteOption) (_ Remote, err error) {
        defer func() {
            if err != nil {
                err = fmt.Errorf("Failed to connect to containerd. Please make sure containerd is installed in your PATH or you have specificed the correct address. Got error: %v", err)
            }
        }()

        //创建remote
        r := &remote{
            stateDir:    stateDir,
            daemonPid:   -1,
            eventTsPath: filepath.Join(stateDir, eventTimestampFilename), // /var/run/docker/libcontainerd/event.ts
        }
        for _, option := range options {
            if err := option.Apply(r); err != nil {   //传进来的参数要实现RemoteOptions接口的Apply(Remote)error方法。设置remote的参数，包括：rpcAddr, runtime, startDaemon, liveRestore, oomScore
                return nil, err
            }
        }

        //创建/var/run/docker/libecontainerd目录，0700
        if err := sysinfo.MkdirAll(stateDir, 0700); err != nil {
            return nil, err
        }

        //设置RPC地址：/var/run/docker/libcontainerd/docker-containerd.sock
        if r.rpcAddr == "" {
            r.rpcAddr = filepath.Join(stateDir, containerdSockFilename)
        }

        //如果设置了startDaemon=true，则运行容器daemon。那么什么时候会设置startDaemon=true呢？在github.com/docker/docker/cmd/dockerd/daemon_unix.go文件中的DaemonCli.getPlatformRemoteOptions方法中有一个判断：如果DaemonCli.Config.ContainerAddr为空时，则会设置startDaemon=true，即libcontainer也是以容器运行。在前面参数中已经提到Config.ContainerAddr参数默认为空，除非你通过命令行flag："--containerd"传入false值。正常情况下，不会运行容器的libcontainer，所以这里暂时不深入分析了。
        if r.startDaemon {
            if err := r.runContainerdDaemon(); err != nil {
                return nil, err
            }
        }

        // 不输出grpc重连日志。grpc是google的一个rpc框架
        grpclog.SetLogger(log.New(ioutil.Discard, "", log.LstdFlags))
        dialOpts := append([]grpc.DialOption{grpc.WithInsecure()},
            grpc.WithDialer(func(addr string, timeout time.Duration) (net.Conn, error) {
                return net.DialTimeout("unix", addr, timeout)
            }),
        )
        //连接rpcAddr，返回连接
        conn, err := grpc.Dial(r.rpcAddr, dialOpts...)
        if err != nil {
            return nil, fmt.Errorf("error connecting to containerd: %v", err)
        }
        //根据返回的连接创建客户端。
        r.rpcConn = conn
        r.apiClient = containerd.NewAPIClient(conn)

        // 读/var/run/docker/event.ts文件中最后一个event的时间戳
        t := r.getLastEventTimestamp()
        //把时间戳转换为google protobuf时间戳格式
        tsp, err := ptypes.TimestampProto(t)
        if err != nil {
            logrus.Errorf("libcontainerd: failed to convert timestamp: %q", err)
        }
        r.restoreFromTimestamp = tsp

        //启动一个goroutine，
        go r.handleConnectionChange()

        //启动事件监控
        if err := r.startEventsMonitor(); err != nil {
            return nil, err
        }

        return r, nil
    }

（1）grpc

这里主要介绍一下grpc的原理和在docker中的使用。

GRPC是google开源的一个高性能RPC框架，基于HTTP2协议，基于protobuf 3.x（高性能数据传输格式），基于Netty 4.x+。从开发角度说，你需要做以下步骤（以java为例）：

a）编写.proto文件，protobuf在其他文章中已经介绍了，这里就不详细介绍了。

b）使用编译工具来编译.proto文件。

c）启动一个server端，监听一个端口，等待client连接，通常用Netty构建

d）启动一个client端，也是通过Netty构建，client与server建立TCP长连接，并发送请求。request和response都是HTTP2，通过Netty channel进行交互。

grpc在这里的用处是创建一个client

（2）handleConnectionChange

含义：

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

            //daemonPid!=-1时才执行下面操作，前面设置了remote.daemon=-1，因此，不执行下面操作。那么什么时候会设置！=-1呢？在运行容器化的libcontainerd时候，有可能会设置。所以这里暂时不详细分析了。
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

（3）startEventsMonitor

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

