创建daemon
==========================================================
## 简介
创建一个docker daemon，为client提供web服务。

## NewDaemon
含义：

    创建daemon的全部所需，为client提供web请求服务。

路径：

    github.com/docker/docker/daemon/daemon.go

定义：

    func NewDaemon(config *Config, registryService registry.Service, containerdRemote libcontainerd.Remote) (daemon *Daemon, err error) {
        //设置默认mtu，如果config.Mtu不为空，则使用该值，否则设置为默认mtu=1500
        setDefaultMtu(config)

        // 检查root key最大值。读/proc/sys/kernel/keys/root_maxkeys文件获得root key最大值。如果该值小于1000000，则把该值设置为1000000.
        if err := ModifyRootKeyLimit(); err != nil {
            logrus.Warnf("unable to modify root key limit, number of containers could be limitied by this quota: %v", err)
        }

        // 验证daemon配置信息是否有效。包括：
        //config.bridgeConfig.Iface和config.bridgeConfig.IP只能设置其中之一。
        //设置config.bridgeConfig.EnableIPTables=false，表示不允许用iptables，如果再设置config.bridgeConfig.InterContainerCommunication=false，表示使用iptables，那这就矛盾了，这种情况也不允许。
        //如果设置了config.bridgeConfig.EnableIPTables=false，则必须设置config.bridgeConfig.EnableIPMasq=false。
        //检查native.cgroupdriver，要么是空，要么是cgroupfs或systemd，其他不允许。
        //cgroup-parent对systemd的cgroup来说，必须是一个有效的slice名字，"xxx.slice"。
        //设置默认runtime，即config.DefaultRuntime=runc
        //如果用systemd，则设置--systemd-cgroup=true，添加到config.Runtimes中。
        if err := verifyDaemonSettings(config); err != nil {
            return nil, err
        }

        // 判断网桥是否可用，主要是看是否config.bridgeConfig.Iface==none，等于node，返回true，否则，返回false。
        config.DisableBridge = isBridgeNetworkDisabled(config)

        // 验证平台是否支持daemon，这里为true，表示支持。
        if !platformSupported {
            return nil, errSystemNotSupported
        }

        // 验证平台相关的属性是否满足要求。检查内存管理区zone，daemon必须运行在zone=0（即全局zone）的内存区中。检查内核版本是否符合要求。
        if err := checkSystem(); err != nil {
            return nil, err
        }

        // 设置SIGUSR1处理器，吹dump栈。
        setupDumpStackTrap()

        //获得uid和gid
        uidMaps, gidMaps, err := setupRemappedRoot(config)
        if err != nil {
            return nil, err
        }
        rootUID, rootGID, err := idtools.GetRootUIDGID(uidMaps, gidMaps)
        if err != nil {
            return nil, err
        }

        // 获得docker的root
        var realRoot string
        if _, err := os.Stat(config.Root); err != nil && os.IsNotExist(err) {
            realRoot = config.Root
        } else {
            realRoot, err = fileutils.ReadSymlinkedDirectory(config.Root)
            if err != nil {
                return nil, fmt.Errorf("Unable to get the full path to root (%s): %s", config.Root, err)
            }
        }

        //设置daemon root，这里没有做操作。
        if err := setupDaemonRoot(config, realRoot, rootUID, rootGID); err != nil {
            return nil, err
        }
        //设置daemon进程，这里也没有做操作。
        if err := setupDaemonProcess(config); err != nil {
            return nil, err
        }

        // 设置临时目录，先读环境变量"DOCK_TMPDIR"，如果存在，则使用该环境变量指示的路径；否则创建/var/lib/docker/tmp目录，权限0700.
        tmp, err := tempDir(config.Root, rootUID, rootGID)
        if err != nil {
            return nil, fmt.Errorf("Unable to get the TempDir under %s: %s", config.Root, err)
        }
        realTmp, err := fileutils.ReadSymlinkedDirectory(tmp)
        if err != nil {
            return nil, fmt.Errorf("Unable to get the full path to the TempDir (%s): %s", tmp, err)
        }
        //设置环境变量TMPDIR=前面设置的路径
        os.Setenv("TMPDIR", realTmp)

        d := &Daemon{configStore: config}
        // daemon失败了，则关闭它。
        defer func() {
            if err != nil {
                if err := d.Shutdown(); err != nil {
                    logrus.Error(err)
                }
            }
        }()

        // 设置默认隔离模式，仅用于windows
        if err := d.setDefaultIsolation(); err != nil {
            return nil, fmt.Errorf("error setting default isolation mode: %v", err)
        }

        logrus.Debugf("Using default logging driver %s", config.LogConfig.Type)

        //设置goroutine最大数量，一般为系统最大线程数的90%，即/proc/sys/kernel/threads-max
        if err := configureMaxThreads(config); err != nil {
            logrus.Warnf("Failed to configure golang's threads limit: %v", err)
        }

        //---------------------------------------这部分与apparmor相关------------------------------------------
        //初始化AppArmor，AppArmor是linux的强制安全访问控制系统。下面详细介绍一下apparmor。
        installDefaultAppArmorProfile()

        //---------------------------------------这部分与container相关-----------------------------------------
        //daemon仓库目录，默认是/var/lib/docker/containers
        daemonRepo := filepath.Join(config.Root, "containers")
        //创建daemon仓库目录。
        if err := idtools.MkdirAllAs(daemonRepo, 0700, rootUID, rootGID); err != nil && !os.IsExist(err) {
            return nil, err
        }

        //----------------------------------------这部分是与image文件系统有关------------------------------------
        //驱动名字，取自环境变量"DOCKER_DRIVER"的值
        driverName := os.Getenv("DOCKER_DRIVER")
        if driverName == "" {
            driverName = config.GraphDriver
        }

        //创建一个layer store对象。image是分层结构，关于image的文件系统的详细内容很多文章已经介绍过了，这里就不详细介绍了。
        d.layerStore, err = layer.NewStoreFromOptions(layer.StoreOptions{
            StorePath:                 config.Root,
            MetadataStorePathTemplate: filepath.Join(config.Root, "image", "%s", "layerdb"),
            GraphDriver:               driverName,
            GraphDriverOptions:        config.GraphOptions,
            UIDMaps:                   uidMaps,
            GIDMaps:                   gidMaps,
        })
        if err != nil {
            return nil, err
        }

        //获得graphdriver名字
        graphDriver := d.layerStore.DriverName()

        //镜像的根目录，默认：/var/lib/docker/image/devicemapper
        imageRoot := filepath.Join(config.Root, "image", graphDriver)

        // 配置和验证内核是否支持安全性配置。这里不错操作。
        if err := configureKernelSecuritySupport(config, graphDriver); err != nil {
            return nil, err
        }

        logrus.Debugf("Max Concurrent Downloads: %d", *config.MaxConcurrentDownloads)
        //下载pull管理器，你让最大并发3
        d.downloadManager = xfer.NewLayerDownloadManager(d.layerStore, *config.MaxConcurrentDownloads)

        logrus.Debugf("Max Concurrent Uploads: %d", *config.MaxConcurrentUploads)
        //上传push管理器，默认最大并发5
        d.uploadManager = xfer.NewLayerUploadManager(*config.MaxConcurrentUploads)

        //创建文件系统。默认/var/lib/docker/image/devicemapper/imagedb下面的content/sha256和metadata/sha256目录，权限0700
        ifs, err := image.NewFSStoreBackend(filepath.Join(imageRoot, "imagedb"))
        if err != nil {
            return nil, err
        }

        //创建一个image store，其实就是layer store，会加载当前存在的image到image store中。
        d.imageStore, err = image.NewImageStore(ifs, d.layerStore)
        if err != nil {
            return nil, err
        }

        //--------------------------------------------这部分与volume相关--------------------------------------------
        // 配置volumes driver。首先，创建/var/lib/docker/volumes目录，作为volume的根目录。然后配置其他相关目录，mount的目录。
        volStore, err := d.configureVolumes(rootUID, rootGID)
        if err != nil {
            return nil, err
        }

        //加载或创建key文件。如果TrustKeyPath目录中存在，则加载，否则创建一个key file。
        trustKey, err := api.LoadOrCreateTrustKey(config.TrustKeyPath)
        if err != nil {
            return nil, err
        }

        //创建/var/lib/docker/trust
        trustDir := filepath.Join(config.Root, "trust")
        if err := system.MkdirAll(trustDir, 0700); err != nil {
            return nil, err
        }

        //创建/var/lib/docker/image/devicemapper/distribution
        distributionMetadataStore, err := dmetadata.NewFSMetadataStore(filepath.Join(imageRoot, "distribution"))
        if err != nil {
            return nil, err
        }

        //创建事件服务
        eventsService := events.New()

        //根据/var/lib/docker/image/devicemapper/repositories.json创建referenceStore
        referenceStore, err := reference.NewReferenceStore(filepath.Join(imageRoot, "repositories.json"))
        if err != nil {
            return nil, fmt.Errorf("Couldn't create Tag store repositories: %s", err)
        }

        //存储定制image，不错任何操作。
        if err := restoreCustomImage(d.imageStore, d.layerStore, referenceStore); err != nil {
            return nil, fmt.Errorf("Couldn't restore custom images: %s", err)
        }

        //迁移开始时间
        migrationStart := time.Now()
        //迁移旧的graph到新的graph中
        if err := v1.Migrate(config.Root, graphDriver, d.layerStore, d.imageStore, referenceStore, distributionMetadataStore); err != nil {
            logrus.Errorf("Graph migration failed: %q. Your old graph data was found to be too inconsistent for upgrading to content-addressable storage. Some of the old data was probably not upgraded. We recommend starting over with a clean storage directory if possible.", err)
        }
        logrus.Infof("Graph migration to content-addressability took %.2f seconds", time.Since(migrationStart).Seconds())

        // Discovery is only enabled when the daemon is launched with an address to advertise.  When
        // initialized, the daemon is registered and we can store the discovery backend as its read-only
        if err := d.initDiscovery(config); err != nil {
            return nil, err
        }

        sysInfo := sysinfo.New(false)
        // Linux系统必须要求mount cgroup device
        if runtime.GOOS == "linux" && !sysInfo.CgroupDevicesEnabled {
            return nil, fmt.Errorf("Devices cgroup isn't mounted")
        }

        d.ID = trustKey.PublicKey().KeyID()
        d.repository = daemonRepo
        d.containers = container.NewMemoryStore()               //创建内存仓库，map[string]*Container
        d.execCommands = exec.NewStore()
        d.referenceStore = referenceStore
        d.distributionMetadataStore = distributionMetadataStore
        d.trustKey = trustKey
        d.idIndex = truncindex.NewTruncIndex([]string{})
        d.statsCollector = d.newStatsCollector(1 * time.Second)
        d.defaultLogConfig = containertypes.LogConfig{
            Type:   config.LogConfig.Type,
            Config: config.LogConfig.Config,
        }
        d.RegistryService = registryService
        d.EventsService = eventsService
        d.volumes = volStore
        d.root = config.Root
        d.uidMaps = uidMaps
        d.gidMaps = gidMaps
        d.seccompEnabled = sysInfo.Seccomp

        d.nameIndex = registrar.NewRegistrar()
        d.linkIndex = newLinkIndex()
        d.containerdRemote = containerdRemote

        //运行gc
        go d.execCommandGC()

        d.containerd, err = containerdRemote.Client(d)
        if err != nil {
            return nil, err
        }

        if err := d.restore(); err != nil {
            return nil, err
        }

        // Plugin system initialization should happen before restore. Do not change order.
        if err := pluginInit(d, config, containerdRemote); err != nil {
            return nil, err
        }

        return d, nil
    }

### 1. installDefaultAppArmorProfile
含义：

    初始化默认的apparmor。apparmor是linux的安全访问控制系统。

路径：

    github.com/docker/docker/daemon/apparmor_default.go

定义：

    func installDefaultAppArmorProfile() {
        if apparmor.IsEnabled() {
            //生成一个默认的profile，并安装到/etc/apparmor.d/docker，
            if err := aaprofile.InstallDefault(defaultApparmorProfile); err != nil {
                apparmorProfiles := []string{defaultApparmorProfile}

                for _, policy := range apparmorProfiles {
                    if err := aaprofile.IsLoaded(policy); err != nil {
                        logrus.Errorf("AppArmor enabled on system but the %s profile could not be loaded.", policy)
                    }
                }
            }
        }
    }




_______________________________________________________________________
[[返回docker-daemon.md]](./docker-daemon.md) 

