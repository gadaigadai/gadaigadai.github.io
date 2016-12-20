---
layout:     post
title:      "heapster"
subtitle:   " \"heapster源码分析\""
date:       2016-12-05 12:00:00
author:     "Gadaigadai"
header-img: "img/bg013.jpg"
catalog: true
tags:
    - heapster
---

Heapster用于采集k8s集群中node和pod资源的数据，其通过node上的kubelet来调用cAdvisor API接口，之后进行数据聚合传至后端存储系统。
直接撸源码：heapster/metrics/heapster.go

```go
func main() {
	sourceFactory := sources.NewSourceFactory()
	sourceProvider, err := sourceFactory.BuildAll(argSources)
	sourceManager, err := sources.NewSourceManager(sourceProvider, sources.DefaultMetricsScrapeTimeout)
  
	// sinks  
	//--sink=influxdb:http://monitoring-influxdb:8086
	sinksFactory := sinks.NewSinkFactory()
	metricSink, sinkList, historicalSource := sinksFactory.BuildAll(argSinks, *argHistoricalSource)

	sinkManager, err := sinks.NewDataSinkManager(sinkList, sinks.DefaultSinkExportDataTimeout, sinks.DefaultSinkStopTimeout)

	// data processors
	metricsToAggregate := []string{
		core.MetricCpuUsageRate.Name,
		core.MetricMemoryUsage.Name,
		core.MetricCpuRequest.Name,
		core.MetricCpuLimit.Name,
		core.MetricMemoryRequest.Name,
		core.MetricMemoryLimit.Name,
	}

	metricsToAggregateForNode := []string{
		core.MetricCpuRequest.Name,
		core.MetricCpuLimit.Name,
		core.MetricMemoryRequest.Name,
		core.MetricMemoryLimit.Name,
	}

	dataProcessors := []core.DataProcessor{
		// Convert cumulaties to rate
		processors.NewRateCalculator(core.RateMetricsMapping),
	}

	kubernetesUrl, err := getKubernetesAddress(argSources)
	kubeConfig, err := kube_config.GetKubeClientConfig(kubernetesUrl)
	kubeClient := kube_client.NewOrDie(kubeConfig)
	podLister, err := getPodLister(kubeClient)
	nodeLister, err := getNodeLister(kubeClient)
	podBasedEnricher, err := processors.NewPodBasedEnricher(podLister)
	dataProcessors = append(dataProcessors, podBasedEnricher)
	namespaceBasedEnricher, err := processors.NewNamespaceBasedEnricher(kubernetesUrl)
	dataProcessors = append(dataProcessors, namespaceBasedEnricher)

	// then aggregators
	dataProcessors = append(dataProcessors,
		processors.NewPodAggregator(),
		&processors.NamespaceAggregator{
			MetricsToAggregate: metricsToAggregate,
		},
		&processors.NodeAggregator{
			MetricsToAggregate: metricsToAggregateForNode,
		},
		&processors.ClusterAggregator{
			MetricsToAggregate: metricsToAggregate,
		})

	nodeAutoscalingEnricher, err := processors.NewNodeAutoscalingEnricher(kubernetesUrl)
	dataProcessors = append(dataProcessors, nodeAutoscalingEnricher)

	// main manager
	manager, err := manager.NewManager(sourceManager, dataProcessors, sinkManager, *argMetricResolution,
		manager.DefaultScrapeOffset, manager.DefaultMaxParallelism)
	manager.Start()

	handler := setupHandlers(metricSink, podLister, nodeLister, historicalSource)
	addr := fmt.Sprintf("%s:%d", *argIp, *argPort)

	mux := http.NewServeMux()
	promHandler := prometheus.Handler()
	if len(*argTLSCertFile) > 0 && len(*argTLSKeyFile) > 0 {
		if len(*argTLSClientCAFile) > 0 {
			authPprofHandler, err := newAuthHandler(handler)
			handler = authPprofHandler

			authPromHandler, err := newAuthHandler(promHandler)
			promHandler = authPromHandler
		}
		mux.Handle("/", handler)
		mux.Handle("/metrics", promHandler)
		healthz.InstallHandler(mux, healthzChecker(metricSink))

		// If allowed users is set, then we need to enable Client Authentication
		if len(*argAllowedUsers) > 0 {
			server := &http.Server{
				Addr:      addr,
				Handler:   mux,
				TLSConfig: &tls.Config{ClientAuth: tls.RequestClientCert},
			}
			glog.Fatal(server.ListenAndServeTLS(*argTLSCertFile, *argTLSKeyFile))
		} else {
			glog.Fatal(http.ListenAndServeTLS(addr, *argTLSCertFile, *argTLSKeyFile, mux))
		}

	} else {
		mux.Handle("/", handler)
		mux.Handle("/metrics", promHandler)
		healthz.InstallHandler(mux, healthzChecker(metricSink))

		glog.Fatal(http.ListenAndServe(addr, mux))
	}
}
```

# source

source用于配置监控来源，它支持的参数：

- inClusterConfig - Use kube config in service accounts associated with heapster's namesapce. (default: true)
- kubeletPort - kubelet port to use (default: 10255)
- kubeletHttps - whether to use https to connect to kubelets (default: false)
- apiVersion - API version to use to talk to Kubernetes. Defaults to the version in kubeConfig.
- insecure - whether to trust kubernetes certificates (default: false)
- auth - client auth file to use. Set auth if the service accounts are not usable.
- useServiceAccount - whether to use the service account token if one is mounted at/var/run/secrets/kubernetes.io/serviceaccount/token (default: false)

heapster启动时source参数样例：--source=kubernetes:http://10.8.65.117:8080?inClusterConfig=false&kubeletHttps=true&kubeletPort=10250&useServiceAccount=true&auth=。NewSourceManager返回的sourceManager结构如下：heapster/metrics/sources/manager.go

```go
type sourceManager struct {
       metricsSourceProvider MetricsSourceProvider
       metricsScrapeTimeout  time.Duration
}
```

MetricsSourceProvider是一个kubeletProvider实例：heapster/metrics/sources/kubelet/kubelet.go

```go
type kubeletProvider struct {
       nodeLister    *cache.StoreToNodeLister
       reflector     *cache.Reflector
       kubeletClient *KubeletClient
}
```

实例化在NewKubeletProvider：heapster/metrics/sources/kubelet/kubelet.go

```go
func NewKubeletProvider(uri *url.URL) (MetricsSourceProvider, error) {
       // create clients
       kubeConfig, kubeletConfig, err := GetKubeConfigs(uri)
       kubeClient := kube_client.NewOrDie(kubeConfig)
       kubeletClient, err := NewKubeletClient(kubeletConfig)
       //初始化kubeClient和kubeletClient
       // Get nodes to test if the client is configured well. Watch gives less error information.
       if _, err := kubeClient.Nodes().List(kube_api.ListOptions{
       //List方法在kubernetes/pkg/client/unversioned/nodes.go
              LabelSelector: labels.Everything(),
              FieldSelector: fields.Everything()}); err != nil {
              glog.Errorf("Failed to load nodes: %v", err)
       }

       // watch nodes
       lw := cache.NewListWatchFromClient(kubeClient, "nodes", kube_api.NamespaceAll, fields.Everything())
       nodeLister := &cache.StoreToNodeLister{Store: cache.NewStore(cache.MetaNamespaceKeyFunc)}
       reflector := cache.NewReflector(lw, &kube_api.Node{}, nodeLister.Store, time.Hour)
       reflector.Run()
       //lw定时通过kubeClient获取node list

       return &kubeletProvider{
              nodeLister:    nodeLister,
              reflector:     reflector,
              kubeletClient: kubeletClient,
       }, nil
}
```

# sink

sink用于设置后端存储，metricSink默认为：heapster/metrics/sinks/factory.go

```
case "metric":
       return metricsink.NewMetricSink(140*time.Second, 15*time.Minute, []string{
              core.MetricCpuUsageRate.MetricDescriptor.Name,
              core.MetricMemoryUsage.MetricDescriptor.Name}), nil
```

对于sink启动参数样例：--sink=influxdb:http://monitoring-influxdb:8086，sinkList包含上面默认的metricSink和InfluxdbSink(heapster/metrics/sinks/influxdb/influxdb.go)。historicalSource默认为空。

生成的sinkManager结构如下：heapster/metrics/sinks/manager.go

```go
type sinkManager struct {
       sinkHolders       []sinkHolder
       exportDataTimeout time.Duration
       stopTimeout       time.Duration
}
```

其中sinkHolder包含DataSink、dataBatchChannel和stopChannel字段。

# dataProcessor



初始化dataProcessors列表，初始元素RateCalculator：heapster/metrics/processors/rate_calculator.go

```go
type RateCalculator struct {
	rateMetricsMapping map[string]core.Metric
	previousBatch      *core.DataBatch
}
```

需要进行速率转化的指标：heapster/metrics/core/metrics.go

```go
var RateMetricsMapping = map[string]Metric{
       MetricCpuUsage.MetricDescriptor.Name:              MetricCpuUsageRate,
       MetricMemoryPageFaults.MetricDescriptor.Name:      MetricMemoryPageFaultsRate,
       MetricMemoryMajorPageFaults.MetricDescriptor.Name: MetricMemoryMajorPageFaultsRate,
       MetricNetworkRx.MetricDescriptor.Name:             MetricNetworkRxRate,
       MetricNetworkRxErrors.MetricDescriptor.Name:       MetricNetworkRxErrorsRate,
       MetricNetworkTx.MetricDescriptor.Name:             MetricNetworkTxRate,
       MetricNetworkTxErrors.MetricDescriptor.Name:       MetricNetworkTxErrorsRate}
```

接下来实例化kubeClient，获取podLister和nodeLister，创建维护cache中pod、namespace和node信息的worker：podBasedEnricher、namespaceBasedEnricher和nodeAutoscalingEnricher，添加到dataProcessors列表。还有就是分别基于Pod、Namespace、Node和Cluster的数据聚合worker，添加到dataProcessors列表。

# manager

初始化manager实例：heapster/metrics/manager/manager.go

```go
type realManager struct {
       source                 core.MetricsSource
       processors             []core.DataProcessor
       sink                   core.DataSink
       resolution             time.Duration
       scrapeOffset           time.Duration
       stopChan               chan struct{}
       housekeepSemaphoreChan chan struct{}
       housekeepTimeout       time.Duration
}
```
manager的start：heapster/metrics/manager/manager.go

```go
func (rm *realManager) Start() {
       go rm.Housekeep()
}

func (rm *realManager) Housekeep() {
	for {
		// Always try to get the newest metrics
		now := time.Now()
		start := now.Truncate(rm.resolution)
		end := start.Add(rm.resolution)
		timeToNextSync := end.Add(rm.scrapeOffset).Sub(now)

		select {
		case <-time.After(timeToNextSync):
			rm.housekeep(start, end)
		case <-rm.stopChan:
			rm.sink.Stop()
			return
		}
	}
}

func (rm *realManager) housekeep(start, end time.Time) {
	if !start.Before(end) {
		glog.Warningf("Wrong time provided to housekeep start:%s end: %s", start, end)
		return
	}

	select {
	case <-rm.housekeepSemaphoreChan:
		// ok, good to go

	case <-time.After(rm.housekeepTimeout):
		glog.Warningf("Spent too long waiting for housekeeping to start")
		return
	}

	go func(rm *realManager) {
		// should always give back the semaphore
		defer func() { rm.housekeepSemaphoreChan <- struct{}{} }()
		data := rm.source.ScrapeMetrics(start, end)

		for _, p := range rm.processors {
		//循环执行dataProcessors列表内的worker
			newData, err := process(p, data)
			if err == nil {
				data = newData
			} else {
				glog.Errorf("Error in processor: %v", err)
				return
			}
		}

		// Export data to sinks
		rm.sink.ExportData(data)

	}(rm)
}
```














