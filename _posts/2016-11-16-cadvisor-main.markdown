---
layout:     post
title:      "cAdvisor"
subtitle:   " \"cAdvisor源码分析\""
date:       2016-11-16 12:00:00
author:     "Gadaigadai"
header-img: "img/bg008.jpg"
catalog: true
tags:
    - cAdvisor
---

参考资料：[cAdvisor源码分析](http://wangzhezhe.github.io/blog/2016/02/12/cadvisor-part1/)。由于是源码分析，我习惯在代码行后加上注释，主要是个人的理解。去掉了部分代码，主要是日志之类的。

这里分析下Kubernetes中监控数据采集所用到的cAdvisor服务源码，直接从服务启动入手：cadvisor/cadvisor.go

```go
func main() {
   
	setMaxProcs()
	//默认设置为cpu核数

	memoryStorage, err := NewMemoryStorage()
	//根据传入的storage参数生成inMemoryCache的实例,backendStorage实例.
	//数据除了保留在内存中(inMemoryCache),还可以设定一个后端存储driver(backendStorage).

	sysFs, err := sysfs.NewRealSysFs()
	//封装了一系列读取系统filesystem数据的方法.

	collectorHttpClient := createCollectorHttpClient(*collectorCert, *collectorKey)

	containerManager, err := manager.New(memoryStorage, sysFs, *maxHousekeepingInterval, *allowDynamicHousekeeping, ignoreMetrics.MetricSet, &collectorHttpClient)
	//创建manager实例.manager具体结构字段见下文.
	//maxHousekeepingInterval(time.Durattion)和allowDynamicHousekeeping(bool)
  	//分别表示数据存在内存的时间以及是否允许动态配置housekeeping的时间,默认值分别为60s和true.

	mux := http.NewServeMux()

	if *enableProfiling {
		mux.HandleFunc("/debug/pprof/", pprof.Index)
		mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
		mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
		mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	}

	// Register all HTTP handlers.
	err = cadvisorhttp.RegisterHandlers(mux, containerManager, *httpAuthFile, *httpAuthRealm, *httpDigestFile, *httpDigestRealm)
  	//添加了证书的方式，把上面生成的containerManager注册进去，具体实现在cadvisor/http/handler.go中.

	cadvisorhttp.RegisterPrometheusHandler(mux, containerManager, *prometheusEndpoint, nil)

	// Start the manager.
	if err := containerManager.Start(); err != nil {
		glog.Fatalf("Failed to start container manager: %v", err)
	}
	//启动一些一直要运行的go routine来用于实现各种监控操作.

	// Install signal handler.
	installSignalHandler(containerManager)
	//为containerManager注册了singlehandler，如果收到了系统发来的kill信号,
	//程序就会捕获到，就直接执行manager的stop函数，manager停止工作.
	glog.Infof("Starting cAdvisor version: %s-%s on port %d", version.Info["version"], version.Info["revision"], *argPort)

	addr := fmt.Sprintf("%s:%d", *argIp, *argPort)
	glog.Fatal(http.ListenAndServe(addr, mux))
}
```

看下manager的具体结构：cadvisor/manager/manager.go

```go
type manager struct {
	containers               map[namespacedContainerName]*containerData
  	//容器信息
	containersLock           sync.RWMutex
	//对map中容器数据存取的控制锁
	memoryCache              *memory.InMemoryCache
  	//内存中保存的监控数据
	fsInfo                   fs.FsInfo
  	//文件系统信息
	machineInfo              info.MachineInfo
  	//machine信息
	quitChannels             []chan error
	//用于存放退出信号的channel,manager关闭的时候会给其中的channel发送退出信号
	cadvisorContainer        string
  	//cAdvisor所在容器,如果部署在容器中的话
	inHostNamespace          bool
  	//是否在host的namespace中
	eventHandler             events.EventManager
	//对event相关操作进行的封装
	startupTime              time.Time
  	//manager启动时间
	maxHousekeepingInterval  time.Duration
	allowDynamicHousekeeping bool
	ignoreMetrics            container.MetricSet
  	//采集的监控指标
	containerWatchers        []watcher.ContainerWatcher
  	//Registers a channel to listen for events affecting subcontainers (recursively).
	eventsChannel            chan watcher.ContainerEvent
	collectorHttpClient      *http.Client
}
```

接下来看manager的具体创建过程：cadvisor/manager/manager.go

```go
// New takes a memory storage and returns a new manager.
func New(memoryCache *memory.InMemoryCache, sysfs sysfs.SysFs, maxHousekeepingInterval time.Duration, allowDynamicHousekeeping bool, ignoreMetricsSet container.MetricSet, collectorHttpClient *http.Client) (Manager, error) {

	// Detect the container we are running on.
	selfContainer, err := cgroups.GetThisCgroupDir("cpu")
  	//确认该进程是否运行在容器中,读取/proc/self/cgroup文件的某个子系统（这里是用cpu子系统）来获取容器id.

	dockerStatus, err := docker.Status()
  	//调用docker api获取docker信息.

	rktPath, err := rkt.RktPath()
  	//rkt,Linux的容器引擎.

	context := fs.Context{
		Docker: fs.DockerContext{
			Root:         docker.RootDir(),
			Driver:       dockerStatus.Driver,
			DriverStatus: dockerStatus.DriverStatus,
		},
		RktPath: rktPath,
	}
	fsInfo, err := fs.NewFsInfo(context)
  	//filesystem的map信息,用于生成manager.

	// If cAdvisor was started with host's rootfs mounted, assume that its running
	// in its own namespaces.
	inHostNamespace := false
	if _, err := os.Stat("/rootfs/proc"); os.IsNotExist(err) {
		inHostNamespace = true
	}
	//判断容器是否存在于hostnamespace(主要判断/rootfs/proc是否存在).
	// Register for new subcontainers.
	eventsChannel := make(chan watcher.ContainerEvent, 16)

	newManager := &manager{
		containers:               make(map[namespacedContainerName]*containerData),
      	//容器信息,对容器进行实际操作的各种handler.
		quitChannels:             make([]chan error, 0, 2),
		memoryCache:              memoryCache,
		fsInfo:                   fsInfo,
		cadvisorContainer:        selfContainer,
		inHostNamespace:          inHostNamespace,
		startupTime:              time.Now(),
		maxHousekeepingInterval:  maxHousekeepingInterval,
		allowDynamicHousekeeping: allowDynamicHousekeeping,
		ignoreMetrics:            ignoreMetricsSet,
		containerWatchers:        []watcher.ContainerWatcher{},
		eventsChannel:            eventsChannel,
		collectorHttpClient:      collectorHttpClient,
	}

	machineInfo, err := machine.Info(sysfs, fsInfo, inHostNamespace)
	newManager.machineInfo = *machineInfo
	versionInfo, err := getVersionInfo()
	glog.Infof("Version: %+v", *versionInfo)

	newManager.eventHandler = events.NewEventManager(parseEventsStoragePolicy())
  	//注册eventHandler.封装了一些对event进行处理的操作,相当于是一个event manager.
	return newManager, nil
}
```

manager的start: cadvisor/manager/manager.go

```go
// Start the container manager.
func (self *manager) Start() error {
  	//facotory实质上是对容器的一些操作的方法的封装.
    //ContainerHandler主要是对容器的一些操作的实现.
    //ContainerHandlerFactory是更上层的抽象,比如创建一个containerhandle或者判断当前containerhandler可否使用.
       err := docker.Register(self, self.fsInfo, self.ignoreMetrics)
       err = rkt.Register(self, self.fsInfo, self.ignoreMetrics)
       if err != nil {
              glog.Warningf("Registration of the rkt container factory failed: %v", err)
       } else {
              watcher, err := rktwatcher.NewRktContainerWatcher()
              if err != nil {
                     return err
              }
              self.containerWatchers = append(self.containerWatchers, watcher)
       }
       err = systemd.Register(self, self.fsInfo, self.ignoreMetrics)
       err = raw.Register(self, self.fsInfo, self.ignoreMetrics)
       rawWatcher, err := rawwatcher.NewRawContainerWatcher()
       self.containerWatchers = append(self.containerWatchers, rawWatcher)

       // Watch for OOMs.
       err = self.watchForNewOoms()
       //读取内核的日志文件，并进行解析，看是否捕获OOM信息
  
       // If there are no factories, don't start any housekeeping and serve the information we do have.
       if !container.HasFactories() {
              return nil
       }

       // Create root and then recover all containers.
       err = self.createContainer("/", watcher.Raw)
       glog.Infof("Starting recovery of all containers")
       err = self.detectSubcontainers("/")
       glog.Infof("Recovery completed")
       //添加注册cadvisor文件系统中名字为"/“的容器以及文件目录层次中的其它容器.

       // Watch for new container.
       quitWatcher := make(chan error)
       err = self.watchForNewContainers(quitWatcher)
       //watch cgroup的文件系统
       self.quitChannels = append(self.quitChannels, quitWatcher)

       // Look for new containers in the main housekeeping thread.
       quitGlobalHousekeeping := make(chan error)
       self.quitChannels = append(self.quitChannels, quitGlobalHousekeeping)
       go self.globalHousekeeping(quitGlobalHousekeeping)

       return nil
}
```
看下detectSubcontainers。首先是执行getContainersDiff，从manager存储的containers(一个map)字段中检测上一步中已经注册进去的containerName为"/“的containerData，之后通过其handler调用ListContainers方法，得到所有的container，之后添加新加入的container，以及移除已经不在list中的container。通过前面的分析，清楚了createContainer(“/”)，注册进来的容器实际操作的时候，使用的hadler是rawContainerFactory。每次ListContainer操作实际上就是把对应的cgroups整个文件系统遍历了一次。具体分析下添加注册cadvisor文件系统中名字为"/“的容器以及文件目录层次中的其它容器的过程：cadvisor/manager/manager.go

```go
// Create a container.
func (m *manager) createContainer(containerName string, watchSource watcher.ContainerWatchSource) error {
	m.containersLock.Lock()
	defer m.containersLock.Unlock()

	return m.createContainerLocked(containerName, watchSource)
}

func (m *manager) createContainerLocked(containerName string, watchSource watcher.ContainerWatchSource) error {
	namespacedName := namespacedContainerName{
		Name: containerName,
	}
	handler, accept, err := container.NewContainerHandler(containerName, watchSource, m.inHostNamespace)

	collectorManager, err := collector.NewCollectorManager()
    //返回的是一个线程兼容的GenericCollectorManager.
    //里面有两个字段，一个是`[]*collectorData{}`数组，另外一个是下次开始搜集的时间.

	logUsage := *logCadvisorUsage && containerName == m.cadvisorContainer
	cont, err := newContainerData(containerName, m.memoryCache, handler, logUsage, collectorManager, m.maxHousekeepingInterval, m.allowDynamicHousekeeping)

	// Add collectors
	labels := handler.GetContainerLabels()
	collectorConfigs := collector.GetCollectorConfigs(labels)
	err = m.registerCollectors(collectorConfigs, cont)
  	//注册collector.

	// Add the container name and all its aliases. The aliases must be within the namespace of the factory.
	m.containers[namespacedName] = cont
	for _, alias := range cont.info.Aliases {
		m.containers[namespacedContainerName{
			Namespace: cont.info.Namespace,
			Name:      alias,
		}] = cont
	}

	contSpec, err := cont.handler.GetSpec()
	contRef, err := cont.handler.ContainerReference()
	newEvent := &info.Event{
		ContainerName: contRef.Name,
		Timestamp:     contSpec.CreationTime,
		EventType:     info.EventContainerCreation,
	}
	err = m.eventHandler.AddEvent(newEvent)
	// Start the container's housekeeping.
	return cont.Start()
	//执行go c.housekeeping()
}
```

container.NewContainerHandler：创建containerHandler,主要是通过遍历factories(一个全局的factories map)，根据containerName看是否能该factories处理，如果可以处理，就调用对应factory的NewContainerHandler方法。rawFactory的CanHandleAndAccept逻辑： 很直接了，如果dockeronly参数被设置为false，或者容器的name为"/“，则CanHandleAndAccept都会返回true。factory注册的顺序先是dockerfactory其次是rawfactory，在检测的时候是遍历factory,执行它们的CanHandleAndAccept方法，哪个先返回true，就先把相应的factory注册进去，所以以”/“命名的容器应该被rawfactory处理，后面的应该被dockerfactory处理。

看一下newContainerData返回的containerData结构：cadvisor/manager/container.go

```go
type containerData struct {
       handler                  container.ContainerHandler
       //对容器操作的具体实现.
       info                     containerInfo
       //容器信息(id,name,alias,namespace,label).子容器信息.容器细节信息(creationtime,cpu,mem,image等等)
       memoryCache              *memory.InMemoryCache
       //manager的InMemoryCache.
       lock                     sync.Mutex
       //互斥锁
       loadReader               cpuload.CpuLoadReader
       //nr_sleeping,nr_running,nr_stopped,nr_uninterruptible,nr_io_wait.
       summaryReader            *summary.StatsSummary
       //摘要信息.availableResources,secondSamples,minuteSamples,derivedStats,dataLock
       loadAvg                  float64 // smoothed load average seen so far.
       housekeepingInterval     time.Duration
       maxHousekeepingInterval  time.Duration
       allowDynamicHousekeeping bool
       lastUpdatedTime          time.Time
       lastErrorTime            time.Time

       // Decay value used for load average smoothing. Interval length of 10 seconds is used.
       loadDecay float64
       //这里loadDecay默认10秒.Ps:Linux系统5秒迭代一次load数值.
       // Whether to log the usage of this container when it is updated.
       logUsage bool

       // Tells the container to stop.
       stop chan bool

       // Runs custom metric collectors.
       collectorManager collector.CollectorManager
}
```

在createContainer的最后一步是执行containerData的Start方法，实际上是用一个goroutine来执行`go c.housekeeping()`，这个housekeeping的主要部分是一个for循环：cadvisor/manager/container.go

```go
func (c *containerData) Start() error {
	go c.housekeeping()
	return nil
}

func (c *containerData) housekeeping() {
  
    ......
  
	// Housekeep every second.
	glog.V(3).Infof("Start housekeeping for container %q\n", c.info.Name)
	lastHousekeeping := time.Now()
	for {
		select {
		case <-c.stop:
			// Stop housekeeping when signaled.
			return
		default:
			// Perform housekeeping.
			start := time.Now()
			c.housekeepingTick()
          
        ......
          
		next := c.nextHousekeeping(lastHousekeeping)
		// Schedule the next housekeeping. Sleep until that time.
		if time.Now().Before(next) {
			time.Sleep(next.Sub(time.Now()))
		} else {
			next = time.Now()
		}
		lastHousekeeping = next
	}
}
  
func (c *containerData) housekeepingTick() {
	err := c.updateStats()
    ....
}

func (c *containerData) updateStats() error {
	stats, statsErr := c.handler.GetStats()
    ......
	if c.loadReader != nil {
		// TODO(vmarmol): Cache this path.
		path, err := c.handler.GetCgroupPath("cpu")
		if err == nil {
			loadStats, err := c.loadReader.GetCpuLoad(c.info.Name, path)
			stats.TaskStats = loadStats
			c.updateLoad(loadStats.NrRunning)
			// convert to 'milliLoad' to avoid floats and preserve precision.
			stats.Cpu.LoadAverage = int32(c.loadAvg * 1000)
		}
	}
	if c.summaryReader != nil {
		err := c.summaryReader.AddSample(*stats)
	}
	var customStatsErr error
	cm := c.collectorManager.(*collector.GenericCollectorManager)
	if len(cm.Collectors) > 0 {
		if cm.NextCollectionTime.Before(time.Now()) {
			customStats, err := c.updateCustomStats()
			if customStats != nil {
				stats.CustomMetrics = customStats
			}
		}
	}

	ref, err := c.handler.ContainerReference()
	err = c.memoryCache.AddStats(ref, stats)
	return customStatsErr
}
```



参考资料：

[cAdvisor获取数据详解](http://blog.opskumu.com/cadvisor.html)

