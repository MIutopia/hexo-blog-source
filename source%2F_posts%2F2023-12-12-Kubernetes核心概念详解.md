---
title: "Kubernetes 核心概念详解：Pod、Service、Deployment 深度剖析"
date: 2023-12-12 10:00:00
categories: ["云原生"]
tags: ["Kubernetes", "云原生", "容器编排", "DevOps"]
cover: /images/hero/03.jpg
summary: "深入理解 Kubernetes 核心概念，从 Pod、Service 到 Deployment，掌握容器编排的精髓。"
---

# Kubernetes 核心概念详解：Pod、Service、Deployment 深度剖析

## 引言

Kubernetes（简称 K8s）是 Google 开源的容器编排平台，已经成为云原生领域的事实标准。本文将深入剖析 K8s 的核心概念，帮助你理解容器编排的精髓。

## 一、Pod：最小部署单元

### 1.1 什么是 Pod

Pod 是 Kubernetes 中最小的部署单元，可以包含一个或多个容器：

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

### 1.2 Pod 特性

- **共享网络命名空间**：同一 Pod 内的容器共享 IP 和端口空间
- **共享存储卷**：可以通过 Volume 共享数据
- **生命周期一致**：Pod 内的容器同时启动和停止

## 二、Service：服务发现与负载均衡

### 2.1 Service 类型

Kubernetes 支持多种 Service 类型：

#### 2.1.1 ClusterIP（默认）

仅在集群内部访问：

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

通过节点端口暴露服务：

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

Deployment 支持滚动更新策略：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

### 3.3 回滚

```bash
kubectl rollout undo deployment/my-deployment
kubectl rollout history deployment/my-deployment
```

## 四、其他核心概念

### 4.1 ReplicaSet

管理 Pod 副本数的控制器：

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

用于有状态应用的部署：

- 稳定的网络标识
- 稳定的存储
- 有序的部署和扩展

### 4.3 DaemonSet

在每个节点上运行一个 Pod：

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

Kubernetes 的核心概念包括 Pod、Service、Deployment 等，理解这些概念是掌握容器编排的关键。通过声明式的配置方式，Kubernetes 可以自动化地管理容器的部署、扩展和运维。

---

**参考资料**
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 实战指南](https://kubernetes.io/docs/tutorials/)
