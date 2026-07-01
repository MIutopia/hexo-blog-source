---
title: Kubernetes 工作流程图解
date: 2023-12-12 10:00:00
categories:
  - 云原生
tags:
  - Kubernetes
  - 架构
---

## 前言

本文以流程图形式展示 Kubernetes 的核心工作流程，涵盖 Pod 调度、服务发现、网络通信等关键环节。

> 注：原文中的图片已移除，后续将补充文字说明。

## Pod 调度流程

```
用户提交 Pod 请求
       ↓
API Server 接收请求
       ↓
Scheduler 根据策略选择节点
       ↓
Controller Manager 通知 Kubelet
       ↓
Kubelet 启动容器
       ↓
Pod 运行中
```

## 服务发现流程

```
Pod 启动并注册到 Endpoints
       ↓
Service 通过 Selector 匹配 Pod
       ↓
kube-proxy 配置 iptables/IPVS 规则
       ↓
客户端访问 Service ClusterIP
       ↓
流量转发到后端 Pod
```

## 网络通信模型

K8s 网络模型遵循以下原则：

- 每个 Pod 都有独立的 IP 地址
- Pod 之间可以直接通信
- 节点上的代理（kube-proxy）负责服务发现和负载均衡

---

*后续将补充更详细的流程图和说明。*
