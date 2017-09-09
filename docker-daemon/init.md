初始化init
================================================
## 简介
Go语言特性：全局var，全局const和init函数在main之前执行。详细内容在kubernetes源码分析中已经介绍了，这里只介绍主要的全局var，全局const和init函数。

## 1. daemonCli
含义：

    创建一个默认值的daemon client，包含一些配置参数。

路径：

    github.com/docker/docker/cmd/dockerd/docker.go  -- 入口
    github.com/docker/docker/cmd/dockerd/daemon.go  -- 函数定义

定义：

    var daemonCli = NewDaemonCli()

    func NewDaemonCli() *DaemonCli {
        daemonConfig := new(daemon.Config)

        //日志配置，flag："--log-opts"。
        daemonConfig.LogConfig.Config = make(map[string]string)

        //集群存储配置，例如TLS配置。flag："--cluster-store-opts"。
        daemonConfig.ClusterOpts = make(map[string]string)

        if runtime.GOOS != "linux" {
            daemonConfig.V2Only = true
        }

        //添加flag参数。
        daemonConfig.InstallFlags(flag.CommandLine, presentInHelp)

        //添加docker配置文件，"--config-file"，默认值：空。
        configFile := flag.CommandLine.String([]string{daemonConfigFileFlag}, defaultDaemonConfigFile, "Daemon configuration file")

        //精确检查参数数量
        flag.CommandLine.Require(flag.Exact, 0)

        //DaemonCli有3个重要参数：Config，commonFlags和configFile。
        return &DaemonCli{
            Config:      daemonConfig,
            commonFlags: cliflags.InitCommonFlags(),
            configFile:  configFile,
        }
    }

（1）InstallFlags
含义：

    添加flag参数到Go原生的CommandLine中。随后调用flag.Parse解析命令行传入的flag参数。

路径：

    github.com/docker/docker/daemon/config_solaris.go

定义：

    func (config *Config) InstallFlags(cmd *flag.FlagSet, usageFn func(string) string) {
        // 添加跨平台flag参数到CommandLine中
        config.InstallCommonFlags(cmd, usageFn)

        // 添加平台相关的flag参数到CommandLine中
        config.attachExperimentalFlags(cmd, usageFn)
    }

    //跨平台参数，github.com/docker/docker/daemon/config.go
    func (config *Config) InstallCommonFlags(cmd *flag.FlagSet, usageFn func(string) string) {
        var maxConcurrentDownloads, maxConcurrentUploads int

        //ServiceOptions的参数，详细请看下面介绍。包含："--registry-mirror", "--insecure-registry", "--disable-legacy-registry"。
        config.ServiceOptions.InstallCliFlags(cmd, usageFn)

        //存储驱动参数，对应config.GraphOptions
        cmd.Var(opts.NewNamedListOptsRef("storage-opts", &config.GraphOptions, nil), []string{"-storage-opt"}, usageFn("Storage driver options"))

        //加载的授权插件，对应config.AuthorizationPlugins
        cmd.Var(opts.NewNamedListOptsRef("authorization-plugins", &config.AuthorizationPlugins, nil), []string{"-authorization-plugin"}, usageFn("Authorization plugins to load"))

        //运行时执行参数，对应config.ExecOptions
        cmd.Var(opts.NewNamedListOptsRef("exec-opts", &config.ExecOptions, nil), []string{"-exec-opt"}, usageFn("Runtime execution options"))

        //daemon PID文件的路径，对应config.Pidfile。默认值：defaultPidFile=/var/run/docker.pid
        cmd.StringVar(&config.Pidfile, []string{"p", "-pidfile"}, defaultPidFile, usageFn("Path to use for daemon PID file"))

        //docker运行根目录，对应config.Root。默认值：defaultGraph=/var/lib/docker
        cmd.StringVar(&config.Root, []string{"g", "-graph"}, defaultGraph, usageFn("Root of the Docker runtime"))

        //被废弃
        cmd.BoolVar(&config.AutoRestart, []string{"#r", "#-restart"}, true, usageFn("--restart on the daemon has been deprecated in favor of --restart policies on docker run"))

        //存储驱动，对应config.GraphDriver
        cmd.StringVar(&config.GraphDriver, []string{"s", "-storage-driver"}, "", usageFn("Storage driver to use"))

        //容器网络MTU，对应config.Mtu。默认值：0。
        cmd.IntVar(&config.Mtu, []string{"#mtu", "-mtu"}, 0, usageFn("Set the containers network MTU"))

        //原生日志信息，不带ANSI着色的完整时间戳。对应config.RawLogs。默认值：false。
        cmd.BoolVar(&config.RawLogs, []string{"-raw-logs"}, false, usageFn("Full timestamps without ANSI coloring"))

        // FIXME: why the inconsistency between "hosts" and "sockets"?
        //dns服务器，对应config.DNS。
        cmd.Var(opts.NewListOptsRef(&config.DNS, opts.ValidateIPAddress), []string{"#dns", "-dns"}, usageFn("DNS server to use"))

        //dns参数，对应config.DNSOptions。
        cmd.Var(opts.NewNamedListOptsRef("dns-opts", &config.DNSOptions, nil), []string{"-dns-opt"}, usageFn("DNS options to use"))

        //dns搜索域名，对应config.DNSSearch
        cmd.Var(opts.NewListOptsRef(&config.DNSSearch, opts.ValidateDNSSearch), []string{"-dns-search"}, usageFn("DNS search domains to use"))

        //为daemon设置的key=value标签，对应config.Labels。
        cmd.Var(opts.NewNamedListOptsRef("labels", &config.Labels, opts.ValidateLabel), []string{"-label"}, usageFn("Set key=value labels to the daemon"))

        //容器日志驱动，对应config.LogConfig.Type。默认值："json-file"，即写文件。如果你发现docker没有写日志文件，那么可以看看docker的这个参数配置是否正确。
        cmd.StringVar(&config.LogConfig.Type, []string{"-log-driver"}, "json-file", usageFn("Default driver for container logs"))

        //默认容器日志驱动的参数，对应config.LogConfig.Config。
        cmd.Var(opts.NewNamedMapOpts("log-opts", config.LogConfig.Config, nil), []string{"-log-opt"}, usageFn("Default log driver options for containers"))

        //集群广播ip地址，对应config.ClusterAdvertise.
        cmd.StringVar(&config.ClusterAdvertise, []string{"-cluster-advertise"}, "", usageFn("Address or interface name to advertise"))

        //分布式存储的URL，对应config.ClusterStore.
        cmd.StringVar(&config.ClusterStore, []string{"-cluster-store"}, "", usageFn("URL of the distributed storage backend"))

        //集群存储参数，对应config.ClusterOpts
        cmd.Var(opts.NewNamedMapOpts("cluster-store-opts", config.ClusterOpts, nil), []string{"-cluster-store-opt"}, usageFn("Set cluster store options"))

        //在远程API中设置CORS头（跨域资源共享），对应config.CorsHeaders.
        cmd.StringVar(&config.CorsHeaders, []string{"-api-cors-header"}, "", usageFn("Set CORS headers in the remote API"))

        //最大并发pull数量，默认值：3
        cmd.IntVar(&maxConcurrentDownloads, []string{"-max-concurrent-downloads"}, defaultMaxConcurrentDownloads, usageFn("Set the max concurrent downloads for each pull"))

        //最大并发push数量，默认值：5
        cmd.IntVar(&maxConcurrentUploads, []string{"-max-concurrent-uploads"}, defaultMaxConcurrentUploads, usageFn("Set the max concurrent uploads for each push"))

        //设置默认ip地址作为swarm发布地址。对应config.SwarmDefaultAdvertiseAddr。
        cmd.StringVar(&config.SwarmDefaultAdvertiseAddr, []string{"-swarm-default-advertise-addr"}, "", usageFn("Set default address or interface for swarm advertised address"))

        config.MaxConcurrentDownloads = &maxConcurrentDownloads  //3
        config.MaxConcurrentUploads = &maxConcurrentUploads      //5
    }

    //ServiceOptions.InstallCliFlags
    //github.com/docker/docker/registry/config.go
    func (options *ServiceOptions) InstallCliFlags(cmd *flag.FlagSet, usageFn func(string) string) {
        //优先的docker registry 镜像站点。
        mirrors := opts.NewNamedListOptsRef("registry-mirrors", &options.Mirrors, ValidateMirror)
        cmd.Var(mirrors, []string{"-registry-mirror"}, usageFn("Preferred Docker registry mirror"))

        //docker与docker registry之间通信使用insecure方式。
        insecureRegistries := opts.NewNamedListOptsRef("insecure-registries", &options.InsecureRegistries, ValidateIndexName)
        cmd.Var(insecureRegistries, []string{"-insecure-registry"}, usageFn("Enable insecure registry communication"))

        //不能连接旧版本registry。
        cmd.BoolVar(&options.V2Only, []string{"-disable-legacy-registry"}, false, usageFn("Disable contacting legacy registries"))
    }

（2）InitCommonFlags
含义：

    添加通用flag参数到docker daemon和client中。

路径：

    github.com/docker/docker/cli/flags/common.go

定义：

    func InitCommonFlags() *CommonFlags {
        var commonFlags = &CommonFlags{FlagSet: new(flag.FlagSet)}

        //dockerCertPath是从环境变量"DOCKER_CERT_PATH"中读的，如果没有，则创建一个目录，从环境变量"DOCKER_CONFIG"中读需要创建的目录。
        if dockerCertPath == "" {
            dockerCertPath = cliconfig.ConfigDir()
        }

        commonFlags.PostParse = func() { postParseCommon(commonFlags) }

        cmd := commonFlags.FlagSet

        //是否debug模式。
        cmd.BoolVar(&commonFlags.Debug, []string{"D", "-debug"}, false, "Enable debug mode")

        //设置日志级别
        cmd.StringVar(&commonFlags.LogLevel, []string{"l", "-log-level"}, "info", "Set the logging level")

        //是否设置TLS
        cmd.BoolVar(&commonFlags.TLS, []string{"-tls"}, false, "Use TLS; implied by --tlsverify")

        //是否使用TLS验证远程连接。
        cmd.BoolVar(&commonFlags.TLSVerify, []string{"-tlsverify"}, dockerTLSVerify, "Use TLS and verify the remote")

        // TODO use flag flag.String([]string{"i", "-identity"}, "", "Path to libtrust key file")

        var tlsOptions tlsconfig.Options
        commonFlags.TLSOptions = &tlsOptions

        //设置ca证书，默认值：dockerCertPath/ca.pem
        cmd.StringVar(&tlsOptions.CAFile, []string{"-tlscacert"}, filepath.Join(dockerCertPath, DefaultCaFile), "Trust certs signed only by this CA")

        //设置cert，默认值：dockerCertPath/cert.pem
        cmd.StringVar(&tlsOptions.CertFile, []string{"-tlscert"}, filepath.Join(dockerCertPath, DefaultCertFile), "Path to TLS certificate file")

        //设置key，默认值：dockerCertPath/key.pem
        cmd.StringVar(&tlsOptions.KeyFile, []string{"-tlskey"}, filepath.Join(dockerCertPath, DefaultKeyFile), "Path to TLS key file")

        //daemon连接的主机
        cmd.Var(opts.NewNamedListOptsRef("hosts", &commonFlags.Hosts, opts.ValidateHost), []string{"H", "-host"}, "Daemon socket(s) to connect to")
        return commonFlags
    }


## 其他参数
//github.com/docker/docker/cmd/dockerd/docker.go
var flHelp    = flag.Bool([]string{"h", "-help"}, false, "Print usage")
var flVersion = flag.Bool([]string{"v", "-version"}, false, "Print version information and quit")


_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 