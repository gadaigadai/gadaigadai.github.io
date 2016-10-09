---
layout:     post
title:      "Kubernetes"
subtitle:   " \"Kubernetes架构梳理\""
date:       2016-09-07 12:00:00
author:     "Gadaigadai"
header-img: "img/bg007.jpg"
catalog: true
tags:
    - kubernetes
---

# 系统架构

参考文章：[Kubernetes系统架构简介](http://www.infoq.com/cn/articles/Kubernetes-system-architecture-introduction)

![k8s架构图](http://dockerone.com/uploads/article/20151031/e4106352b7d6ca95cf8dcd69af544f57.png)



Kubernetes是Google容器集群管理系统的开源版本，提供应用的部署、维护和扩展等功能。节点分为master和node两大类，master节点部署Api-server、Scheduler、Replication-Controller、etcd，node节点部署Kubelet、Kube-proxy、docker。Pod是Kubernetes的基本操作单元，由一个或多个容器组成，一般运行相同的应用。Services是服务的抽象，对外提供单一访问端口，后端由多个Pod组成，由Kube-proxy将外部请求分发到后端容器相应的endpoints。Replication-Controller确保任何时候Kubernetes集群中有指定数量的Pod副本在运行。

# 主要组件

## etcd

Etcd是一个分布式的高性能key-value存储系统，负责Kubernetes中所有REST API对象的持久化保存。

## kube-apiserver

Kubernetes对外提供统一的入口，通过REST API的方式供客户端调用，例如kubectl。

## kube-controller-manager

控制集群中程序任务的后台线程。逻辑上，每个控制任务对应一个线程，包括Node Controller、Replication Controller、Service Controller、Endpoints Controller等。

## kube-scheduler

维护可用的node节点及资源列表，根据输入的待调度pod请求，通过调度算法选择最优node节点，输出绑定了pod的node节点。

## kubelet

Kubelet是Kubernetes集群中每个node和Api-server的连接点，负责容器和pod的实际管理工作。

## kube-proxy

为pod提供代理服务，用来实现 kubernetes 的 service 机制。