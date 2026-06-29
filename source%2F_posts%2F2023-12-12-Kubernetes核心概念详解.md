---
title: "Kubernetes 核心概念详解：Pod、Service、Deployment 深度剖析"
date: 2023-12-12 10:00:00
categories: ["云原生技术"]
tags: ["Kubernetes", "云原生", "容器编排", "DevOps"]
cover: /images/hero/03.jpg
summary: "深入理解 Kubernetes 核心概念，从 Pod、Service 到 Deployment，掌握容器编排的精髓。"
---

# Kubernetes 核心概念详解：Pod、Service、Deployment 深度剖析

## 目录

- [引言](#引言)
- [一、Pod：最小部署单元](#一pod最小部署单元)
- [二、Service：服务发现与负载均衡](#二service服务发现与负载均衡)
- [三、Deployment：声明式应用部署](#三deployment声明式应用部署)
- [四、其他核心概念](#四其他核心概念)
- [五、实战演练](#五实战演练)
- [总结](#总结)

## 引言

Kubernetes（简称 K8s）是 Google 开源的容器编排平台，已经成为云原生领域的事实标准。无论是中小型应用还是大型分布式系统，K8s 都能提供强大的自动化部署、扩展和运维能力。

本文将深入剖析 K8s 的核心概念，帮助你理解容器编排的精髓，为后续的实践打下坚实基础。

## 一、Pod：最小部署单元

### 1.1 什么是 Pod

Pod 是 Kubernetes 中最小的部署单元，可以包含一个或多个紧密耦合的容器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  containers:
  - name: web
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ["sleep", "3600"]
```

### 1.2 Pod 核心特性

| 特性 | 说明 |
|------|------|
| **共享网络命名空间** | 同一 Pod 内的容器共享 IP 和端口空间 |
| **共享存储卷** | 可以通过 Volume 在容器间共享数据 |
| **生命周期一致** | Pod 内的容器同时启动和停止 |

## 二、Service：服务发现与负载均衡

### 2.1 Service 类型

Kubernetes 支持多种 Service 类型，满足不同场景的需求：

#### 2.1.1 ClusterIP（默认）

仅在集群内部访问，适用于内部服务通信：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

#### 2.1.2 NodePort

通过节点端口暴露服务到集群外部：

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000
```

#### 2.1.3 LoadBalancer

通过云厂商负载均衡器暴露服务：

```yaml
spec:
  type: LoadBalancer
```

#### 2.1.4 ExternalName

将服务映射到外部域名：

```yaml
spec:
  type: ExternalName
  externalName: example.com
```

## 三、Deployment：声明式应用部署

### 3.1 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: web
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 3.2 滚动更新

Deployment 支持声明式滚动更新策略：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

> **滚动更新要点**：
> - `maxSurge`：更新过程中最多额外创建的 Pod 数
> - `maxUnavailable`：更新过程中最多不可用的 Pod 数

### 3.3 回滚

```bash
# 回滚到上一个版本
kubectl rollout undo deployment/my-deployment

# 查看部署历史
kubectl rollout history deployment/my-deployment
```

## 四、其他核心概念

### 4.1 ReplicaSet

管理 Pod 副本数的控制器，确保指定数量的 Pod 运行：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
```

### 4.2 StatefulSet

用于有状态应用的部署，提供：

- **稳定的网络标识**：每个 Pod 有固定的主机名
- **稳定的存储**：每个 Pod 有独立的持久化存储
- **有序的部署和扩展**：按顺序创建和删除

### 4.3 DaemonSet

在每个节点上运行一个 Pod，适用于监控、日志收集等场景：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-app
```

## 五、实战演练

### 5.1 部署一个完整应用

```bash
# 创建 Deployment
kubectl apply -f deployment.yaml

# 查看状态
kubectl get deployments
kubectl get pods

# 扩容
kubectl scale deployment/my-deployment --replicas=5

# 更新镜像
kubectl set image deployment/my-deployment web=nginx:1.25
```

## 总结

Kubernetes 的核心概念构成了容器编排的基石：

1. **Pod**：最小部署单元，容器的组合
2. **Service**：服务发现和负载均衡，解耦服务间的通信
3. **Deployment**：声明式应用部署，自动化管理应用生命周期
4. **ReplicaSet/StatefulSet/DaemonSet**：不同场景的控制器

理解这些概念是掌握容器编排的关键。通过声明式的配置方式，Kubernetes 可以自动化地管理容器的部署、扩展和运维，让开发者专注于业务逻辑。

---

**参考资料**
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 实战指南](https://kubernetes.io/docs/tutorials/)
