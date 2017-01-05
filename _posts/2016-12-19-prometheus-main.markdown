---
layout:     post
title:      "prometheus"
subtitle:   " \"prometheus入门\""
date:       2016-12-19 12:00:00
author:     "Gadaigadai"
header-img: "img/trim1.jpg"
catalog: true
tags:
    - prometheus
---



![官方架构图](https://prometheus.io/assets/architecture.svg)



Prometheus是一个开源的监控告警系统，由多个组件构成，比如：[prometheus](https://github.com/prometheus/prometheus)（重量级的监控系统和时序数据库）和[alertmanager](https://github.com/prometheus/alertmanager)（告警系统）。和Heapster相比，Prometheus功能更完善、更全面，据说性能也足够支撑上万台规模的集群，让我们拭目以待吧。Heapster是k8s的一个子项目，用于获取集群的性能数据。Heapster除了支持基本的metric之外，还支持事件类型的数据，比如容器生命周期事件。而Prometheus是一个全套监控和预测方案，包括数据采集、存储、查询和可视化，以及报警功能。它采用了基于LevelDB的本地存储，并没有使用成熟的面向列的数据库。不过Prometheus 也在试验将数据存储到OpenTSDB。Prometheus 采用了pull模式，即服务器通过探针端的exporter来主动拉取数据，而不是探针主导上报。

## 编译启动

```shell
[root@localhost ~]# yum install -y go git
[root@localhost ~]# go version
go version go1.6.3 linux/amd64
[root@localhost ~]# export GOROOT=/usr/lib/golang/
[root@localhost ~]# export GOPATH=/gopath
[root@localhost ~]# cd /gopath/
[root@localhost gopath]# mkdir -p src/github.com/prometheus
[root@localhost gopath]# cd src/github.com/prometheus
[root@localhost prometheus]# git clone https://github.com/prometheus/prometheus.git
Cloning into 'prometheus'...
remote: Counting objects: 23706, done.
remote: Compressing objects: 100% (82/82), done.
remote: Total 23706 (delta 36), reused 0 (delta 0), pack-reused 23624
Receiving objects: 100% (23706/23706), 16.81 MiB | 1.58 MiB/s, done.
Resolving deltas: 100% (12953/12953), done.
[root@localhost prometheus]# cd prometheus/
[root@localhost prometheus]# git checkout -B Branch_v1.4.1
Switched to a new branch 'Branch_v1.4.1'
[root@localhost prometheus]# make build
>> fetching promu
>> building binaries
 >   prometheus
 >   promtool
[root@localhost prometheus]# ./prometheus -version
prometheus, version 1.4.1 (branch: Branch_v1.4.1, revision: 2e3b42ad6c9730891908eb126aa3dc4df65764f2)
  build user:       root@localhost
  build date:       20161221-17:09:59
  go version:       go1.6.3
[root@localhost prometheus]# cd ..
[root@localhost prometheus]# ll
total 8
drwxr-xr-x. 21 root root 4096 Dec 22 01:05 prometheus
drwxr-xr-x.  7 root root 4096 Dec 22 01:00 promu
[root@localhost prometheus]# git clone https://github.com/prometheus/alertmanager.git
Cloning into 'alertmanager'...
remote: Counting objects: 6479, done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 6479 (delta 1), reused 0 (delta 0), pack-reused 6470
Receiving objects: 100% (6479/6479), 7.66 MiB | 884.00 KiB/s, done.
Resolving deltas: 100% (3359/3359), done.
[root@localhost prometheus]# cd alertmanager/
[root@localhost alertmanager]# git checkout -B Branch_v0.5.1
Switched to a new branch 'Branch_v0.5.1'
[root@localhost alertmanager]# make build
>> building binaries
 >   alertmanager
[root@localhost alertmanager]# cd ..
[root@localhost prometheus]# git clone https://github.com/prometheus/pushgateway.git
Cloning into 'pushgateway'...
remote: Counting objects: 943, done.
remote: Total 943 (delta 0), reused 0 (delta 0), pack-reused 943
Receiving objects: 100% (943/943), 1.31 MiB | 218.00 KiB/s, done.
Resolving deltas: 100% (412/412), done.
[root@localhost prometheus]# cd pushgateway/
[root@localhost pushgateway]# git checkout -B Branch_v0.3.1
Switched to a new branch 'Branch_v0.3.1'
[root@localhost pushgateway]# make build
>> building binaries
 >   pushgateway
```

不过以docker方式启动prometheus会更好：

```shell
[root@localhost ~]# docker pull prom/prometheus:v1.4.1
[root@localhost ~]# docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml  prom/prometheus:v1.4.1
[root@localhost ~]# docker run -p 9090:9090 -v /prometheus-data prom/prometheus:v1.4.1 -config.file=/prometheus-data/prometheus.yml
```

## Configuration

有关yml配置选项的详细介绍文档：[configuration documentation](https://prometheus.io/docs/operating/configuration)

以及Kubernetes的配置文件样例：[example Prometheus configuration file for k8s](https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml) 

Prometheus可以在运行时重新加载其配置。 如果新配置的格式不正确，则不会应用更改。 通过向Prometheus进程发送SIGHUP来触发配置重新加载。 这也将重新加载任何配置的规则文件。

## Data model

Time series：隶属于相同的指标（metric name）和标签集合（a set of key-value pairs, also known as labels）的时序数据流。每一个time series由监控数据的指标名和标签集合唯一标识，Prometheus中的数据都是以time series的形式存储的。

指标名要能够表达所监控系统的主要功能类型，它可以包含ASCII字母和数字，以及下划线和冒号。它必须与正则表达式\[a-zA-Z_:][a-zA-Z0-9_:\]*匹配。分为以下四种类型：counter，累积值，一直增长的；gauge，测量值，离散型的；histogram，直方图；summary，统计摘要。

标签主要用于Prometheus的维度数据模型，标签名称可以包含ASCII字母，数字以及下划线。它们必须与正则表达式\[a-zA-Z_][a-zA-Z0-9_\]*匹配，以__开头的标签名称保留供内部使用。根据yml配置文件会自动生成类似的标签：{job="job-name", instance="ip:port"}

```shell
<metric name>{<label name>=<label value>, ...}
```

## LocalStorage

Prometheus有一个复杂的本地存储子系统。 对于索引，它使用LevelDB。对于批量的sample数据，它使用自定义的以1024字节大小的chunk为单位的存储结构，存储到磁盘上的时序文件中。

Prometheus会在内存中保存最常用数据chunk（默认保存1048576个chunk数据）。基于Prometheus提供的多维度数据聚合功能，查询时同一时刻会用到很多的数据chunk，因此Prometheus需要相当大的内存来存储常用数据。一般情况下，建议配置的storage.local.memory-chunks大小需要3倍于active time series，可以通过Prometheus自监控的prometheus_local_storage_memory_chunks和process_resident_memory_bytes指标了解内存使用情况。当active time series量超过配置的storage.local.memory-chunks大小的110%时，会启用节流规则，跳过数据的scrapes and rule evaluations，直到active time series量降到配置值的95%以下。等待写入磁盘的chunk队列大小storage.local.max-chunks-to-persist经验值为storage.local.memory-chunks大小的50%，同样队列长度大于配置值时会启用节流，直到队列减小到配置值的95%以下。等待写入磁盘的chunk队列压力较大时，会进入 “rushed mode”：数据写入后不进行时序文件同步操作；数据点只会以storage.local.checkpoint-interval的频率生成；不再起用节流，尽快写数据到磁盘文件。当打分值（prometheus_local_storage_persistence_urgency_score，Prometheus自监控的指标名，依赖于等待写入磁盘的chunk队列大小和内存中chunk数量）低于0.7时，结束 “rushed mode”。

PromQL 查询会给LevelDB后台索引造成很大的压力，需要对一些index-cache-size参数进行调优：storage.local.index-cache-size.label-name-to-label-values（正则表达式匹配用）；storage.local.index-cache-size.label-pair-to-fingerprints（如果大量时序数据使用相同的label pair，需要增大这个参数）；storage.local.index-cache-size.fingerprint-to-metric和-storage.local.index-cache-size.fingerprint-to-timerange（如果有大量的时序数据归档操作，需要增大这个参数）。一条涉及10w+时序数据的查询操作，大约需要1GB的cache大小来支持。

storage.local.path参数指定磁盘文件存储路径，默认为./data。storage.local.retention参数设置磁盘文件数据的保留时间。如果磁盘文件保留时间较长，相应的storage.local.series-file-shrink-ratio数据压缩比例参数值也要增大。

关于chunk数据的编码方式，由参数storage.local.chunk-encoding-version（type值可以是0,1,2，默认为1）指定。Type 0 是delta encoding；Type 1 是double-delta encoding，比delta encoding具有更好的压缩性能；Type 2 是variable bit-width encoding，其中timestamp字段依然使用double-delta encoding，只是算法稍有差别，sample value字段则根据该chunk中数据value类型的不同而采取不同的编码方式。

## Alertmanager

Prometheus根据配置好的告警规则过滤数据，将输出的alert发送给Alertmanager，Alertmanager将接收到的alert按组聚合、去重、applies silences、节流，最后发送告警通知（邮件、短信等方式）。

告警规则的语法：

```sh
ALERT <alert name>
  IF <expression>
  [ FOR <duration> ]
  [ LABELS <label set> ]
  [ ANNOTATIONS <label set> ]

###  example
# Alert for any instance that is unreachable for >5 minutes.
ALERT InstanceDown
  IF up == 0
  FOR 5m
  LABELS { severity = "page" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} down",
    description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.",
  }
```
















































