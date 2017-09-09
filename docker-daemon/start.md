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

        containerdRemote, err := libcontainerd.New(cli.getLibcontainerdRoot(), cli.getPlatformRemoteOptions()...)
        if err != nil {
            return err
        }
        cli.api = api
        signal.Trap(func() {
            cli.stop()
            <-stopc // wait for daemonCli.start() to return
        })

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
                // Hack: Bypass setting the mirrors to IndexConfigs since they are going away
                // and Mirrors are only for the official registry anyways.
                Mirrors: options.Mirrors,
            },
            V2Only: options.V2Only,
        }
        // Split --insecure-registry into CIDR and registry-specific settings.
        for _, r := range options.InsecureRegistries {
            // Check if CIDR was passed to --insecure-registry
            _, ipnet, err := net.ParseCIDR(r)
            if err == nil {
                // Valid CIDR.
                config.InsecureRegistryCIDRs = append(config.InsecureRegistryCIDRs, (*registrytypes.NetIPNet)(ipnet))
            } else {
                // Assume `host:port` if not CIDR.
                config.IndexConfigs[r] = &registrytypes.IndexInfo{
                    Name:     r,
                    Mirrors:  make([]string, 0),
                    Secure:   false,
                    Official: false,
                }
            }
        }

        // Configure public registry.
        config.IndexConfigs[IndexName] = &registrytypes.IndexInfo{
            Name:     IndexName,
            Mirrors:  config.Mirrors,
            Secure:   true,
            Official: true,
        }

        return config
    }

_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 

