创建dockerd进程
==========================================================
## 简介
创建一个docker daemon，为client提供web服务。

## NewDaemon
含义：

    创建dockerd的全部所需，为client提供web请求服务。

路径：

    github.com/docker/docker/daemon/daemon.go

定义：

    func NewDaemon(config *Config, registryService registry.Service, containerdRemote libcontainerd.Remote) (daemon *Daemon, err error) {
        //设置默认mtu，如果config.Mtu不为空，则使用该值，否则设置为默认mtu=1500
        setDefaultMtu(config)

        // 设置root key limit不能小于1000000。读取/proc/sys/kernel/keys/root-maxkeys文件，如果值小于1000000，则设置为1000000。该值会限制启动容器的数量。
        if err := ModifyRootKeyLimit(); err != nil {
            logrus.Warnf("unable to modify root key limit, number of containers could be limitied by this quota: %v", err)
        }

        // 验证Config配置信息是否有效。包括：
        // config.bridgeConfig.Iface和config.bridgeConfig.IP只能设置其中之一。
        // 设置config.bridgeConfig.EnableIPTables=false，表示不允许用iptables，如果再设置config.bridgeConfig.InterContainerCommunication=false，表示使用iptables，那这就矛盾了，这种情况也不允许。默认是设置EnableIPTables=true，InterContainerCommunication=true。
        // 如果设置了config.bridgeConfig.EnableIPTables=false，则必须设置config.bridgeConfig.EnableIPMasq=false。
        // 检查native.cgroupdriver，要么是空，要么是cgroupfs或systemd，其他不允许。在前面的centos例子中，设置为systemd。
        // cgroup-parent对systemd的cgroup来说，必须是一个有效的slice名字，"xxx.slice"。
        // 如果没有设置DefaultRuntime，则设置DefaultRuntime=runc。前面已经介绍过设置DefaultRuntime=docker-runc
        // 如果用systemd，则设置--systemd-cgroup=true，添加到config.Runtimes中。
        if err := verifyDaemonSettings(config); err != nil {
            return nil, err
        }

        // 判断网桥是否可用，如果config.bridgeConfig.Iface==none，则返回true。
        config.DisableBridge = isBridgeNetworkDisabled(config)

        // 验证平台是否支持daemon，linux默认platformSupported=true。
        if !platformSupported {
            return nil, errSystemNotSupported
        }

        // 验证平台相关的属性是否满足要求。检查内存管理区zone，docker daemon必须运行在zone=0（即全局zone）的内存区中。检查内核版本要大于等于5.12.0。
        if err := checkSystem(); err != nil {
            return nil, err
        }

        // 设置SIGUSR1信号处理器。SIGUSR1是内核留给用户使用的信号。
        setupDumpStackTrap()

        //获得uid和gid映射，下面会详细说明。
        uidMaps, gidMaps, err := setupRemappedRoot(config)
        if err != nil {
            return nil, err
        }
        //根据uidMaps获取容器中root的uid所对应的宿主机上面的rootUID，同理，根据gidMaps获取容器中root的gid对应的宿主机上面的rootGID。
        rootUID, rootGID, err := idtools.GetRootUIDGID(uidMaps, gidMaps)
        if err != nil {
            return nil, err
        }

        // 获得docker的root目录：/var/lib/docker
        var realRoot string
        if _, err := os.Stat(config.Root); err != nil && os.IsNotExist(err) {
            realRoot = config.Root
        } else {
            realRoot, err = fileutils.ReadSymlinkedDirectory(config.Root)
            if err != nil {
                return nil, fmt.Errorf("Unable to get the full path to root (%s): %s", config.Root, err)
            }
        }

        //设置docker daemon的root目录。如果存在，则设置权限0711，否则创建目录，并设置权限0711。如果config.RemappedRoot不为空，则在root根目录下创建rootUID.rootGID，权限0711，rootUID:rootGID。
        if err := setupDaemonRoot(config, realRoot, rootUID, rootGID); err != nil {
            return nil, err
        }
        //往/proc/self/oom_score_adj文件中写入config.OOMScoreAdjust值。
        if err := setupDaemonProcess(config); err != nil {
            return nil, err
        }

        // 设置临时目录，默认在docker daemon的根目录/var/lib/docker下面创建tmp目录，权限0700，rootUID:rootGID。如果设置了环境变量DOCKER_TMPDIR，则创建的临时目录名称为该环境变量的值。
        tmp, err := tempDir(config.Root, rootUID, rootGID)
        if err != nil {
            return nil, fmt.Errorf("Unable to get the TempDir under %s: %s", config.Root, err)
        }
        realTmp, err := fileutils.ReadSymlinkedDirectory(tmp)
        if err != nil {
            return nil, fmt.Errorf("Unable to get the full path to the TempDir (%s): %s", tmp, err)
        }
        ///将临时目录的路径设置为环境变量TMPDIR的值。
        os.Setenv("TMPDIR", realTmp)

        //创建Daemon结构体。设置配置信息config。
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

        //设置Golang线程数最大值。一般为内核最大线程数的90%，即读取/proc/sys/kernel/threads-max文件获取内核最大线程数，然后乘以90%，即设置src/runtime下面的sched.maxmcount值。不过该值不能超过int32的最大值，即0x7fffffff。
        if err := configureMaxThreads(config); err != nil {
            logrus.Warnf("Failed to configure golang's threads limit: %v", err)
        }

        //---------------------------------------这部分与apparmor相关------------------------------------------
        //初始化AppArmor，AppArmor是linux的强制安全访问控制系统。下面详细介绍一下apparmor。
        installDefaultAppArmorProfile()

        //---------------------------------------这部分与containers相关-----------------------------------------
        //创建daemon repo目录：/var/lib/docker/containers，权限0700，rootUID:rootGID。
        daemonRepo := filepath.Join(config.Root, "containers")
        if err := idtools.MkdirAllAs(daemonRepo, 0700, rootUID, rootGID); err != nil && !os.IsExist(err) {
            return nil, err
        }

        //----------------------------------------这部分是与image文件系统有关------------------------------------
        //设置docker driver，如果环境变量DOCKER_DRIVER存在，则使用它的值，否则取config.GraphDriver，对应json："storage-driver"。默认是：devicemapper。
        driverName := os.Getenv("DOCKER_DRIVER")
        if driverName == "" {
            driverName = config.GraphDriver
        }

        //创建一个layer store对象。image是分层结构，关于image的文件系统的详细内容很多文章已经介绍过了，这里就不详细介绍了。
        d.layerStore, err = layer.NewStoreFromOptions(layer.StoreOptions{
            StorePath:                 config.Root,                                           // /var/lib/docker
            MetadataStorePathTemplate: filepath.Join(config.Root, "image", "%s", "layerdb"),  // /var/lib/docker/image/devicemapper/layerdb
            GraphDriver:               driverName,
            GraphDriverOptions:        config.GraphOptions,
            UIDMaps:                   uidMaps,
            GIDMaps:                   gidMaps,
        })
        if err != nil {
            return nil, err
        }

        //获得graphdriver名字，默认：devicemapper
        graphDriver := d.layerStore.DriverName()

        //镜像的根目录，默认：/var/lib/docker/image/devicemapper
        imageRoot := filepath.Join(config.Root, "image", graphDriver)

        // 配置和验证内核是否支持安全性配置。这里不错操作。
        if err := configureKernelSecuritySupport(config, graphDriver); err != nil {
            return nil, err
        }

        logrus.Debugf("Max Concurrent Downloads: %d", *config.MaxConcurrentDownloads)
        //下载pull管理器，默认最大并发3
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

        //存储定制image，linux不存储定制image。
        if err := restoreCustomImage(d.imageStore, d.layerStore, referenceStore); err != nil {
            return nil, fmt.Errorf("Couldn't restore custom images: %s", err)
        }

        //迁移镜像
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

### 1. setupRemappedRoot
含义：

    将容器的uid和gid映射到宿主机的uid和gid

路径：

    github.com/docker/docker/daemon/daemon_unix.go

定义：

    func setupRemappedRoot(config *Config) ([]idtools.IDMap, []idtools.IDMap, error) {
        if runtime.GOOS != "linux" && config.RemappedRoot != "" {
            return nil, nil, fmt.Errorf("User namespaces are only supported on Linux")
        }

        // 将容器的uid和gid映射到宿主机的uid和gid。
        // IDMap={ContainerID int `json:"container_id"`, HostID int `json:"host_id"`, Size int `json:"size"`}，对应前面介绍过的/proc/[pid]/uid_map和/proc/[pid]/gid_map文件中值，即：内部id   外部id    长度。
        var (
            uidMaps, gidMaps []idtools.IDMap  
        )

        //命令行flag参数："--userns-remap"可以设置config.RemappedRoot的值。格式：uid:gid。默认是空。
        if config.RemappedRoot != "" {
            //根据参数config.RemappedRoot的值解析出uid和gid，然后验证uid必须来自于/etc/passwd文件，并且根据uid获得username，如果config.RemappedRoot的值没有gid，则使用uid作为gid的值，验证gid必须来自/etc/group文件，并且根据gid获得groupname。
            username, groupname, err := parseRemappedRoot(config.RemappedRoot)
            if err != nil {
                return nil, nil, err
            }
            //root用户不能重新映射为自身。
            if username == "root" {
                // Cannot setup user namespaces with a 1-to-1 mapping; "--root=0:0" is a no-op
                // effectively
                logrus.Warn("User namespaces: root cannot be remapped with itself; user namespaces are OFF")
                return uidMaps, gidMaps, nil
            }
            logrus.Infof("User namespaces: ID ranges will be mapped to subuid/subgid ranges of: %s:%s", username, groupname)
            // 更新config.RemappedRoot为username和groupname
            config.RemappedRoot = fmt.Sprintf("%s:%s", username, groupname)

            //读/etc/subuid文件，解析每一行，每一行以冒号分隔，3个字段，找到第一个字段为username或ALL值，然后获得范围rangList{startid, length}，其中startid是第二个字段，length是第三个字段。最后将startid赋给uidMaps.HostID，将length赋给uidMaps.Size，uidMaps.ContainerID的第一个值为0，以后的值递增length长度。gitMaps的设置类似，只是需要读/etc/subgid文件。
            uidMaps, gidMaps, err = idtools.CreateIDMappings(username, groupname)
            if err != nil {
                return nil, nil, fmt.Errorf("Can't create ID mappings: %v", err)
            }
        }
        return uidMaps, gidMaps, nil
    }


### 2. installDefaultAppArmorProfile
含义：

    初始化默认的apparmor。apparmor是linux的安全访问控制系统。

路径：

    github.com/docker/docker/daemon/apparmor_default.go

定义：

    func installDefaultAppArmorProfile() {
        //检查apparmor在宿主机上是否支持。如果支持，则安装。
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

