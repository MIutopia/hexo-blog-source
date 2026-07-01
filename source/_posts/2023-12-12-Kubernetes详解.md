---
title: Kubernetes 核心概念详解
date: 2023-12-12 10:00:00
categories:
  - 云原生
tags:
  - Kubernetes
  - DevOps
---

## 前言

Kubernetes（K8s）是容器编排的事实标准。这篇文章从 Pod 开始，逐步讲解 K8s 的核心概念和资源对象。

## Pod

### 概述

Pod 是 K8s 中最小的部署单元，是一组（一个或多个）容器的集合。Pod 就像是豌豆荚，而容器就像是豌豆荚中的豌豆。这些容器共享存储、网络等资源。

### Pod 资源清单

```yaml
apiVersion: v1          # API 版本
kind: Pod               # 资源对象类型
metadata:               # Pod 元数据
  name: nginx-pod       # Pod 名称
  labels:               # 定义标签
    tybe: app
    test: 1.0.0
  namespace: 'default'  # 命名空间
spec:                   # Pod 规格
  containers:           # 容器列表
  - name: nginx
    image: 'nginx:1.20.2'
    imagePullPolicy: IfNotPresent
    startupProbe:       # 启动探针
      exec:
        command:
        - /bin/sh
        - -c
        - "sleep 3; echo success > /inited"
      initialDelaySeconds: 1
      periodSeconds: 5
      failureThreshold: 3
      successThreshold: 1
      timeoutSeconds: 5
    readinessProbe:     # 就绪探针
      httpGet:
        path: /started.html
        port: 80
      initialDelaySeconds: 1
      periodSeconds: 5
      failureThreshold: 3
    command:            # 启动命令
    - nginx
    - -g
    - 'daemon off;'
    workingDir: /usr/share/nginx/html
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    env:                # 环境变量
    - name: JVM_OPTS
      value: '-Xms128m -Xmx128m'
    resources:          # 资源限制
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
  restartPolicy: OnFailure
```

## Deployment 控制器

### 配置文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    controller: deploy
spec:
  replicas: 3                    # 副本数量
  revisionHistoryLimit: 5        # 保留历史版本数
  paused: false                  # 是否暂停部署
  progressDeadlineSeconds: 600   # 部署超时时间
  strategy:                      # 更新策略
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 30%              # 最大额外副本数
  selector:                      # 选择器
    matchLabels:
      app: nginx-pod
  template:                      # Pod 模板
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.2
        ports:
        - containerPort: 80
```

### 版本管理

```bash
# 查看升级状态
kubectl rollout status deployment nginx-deploy

# 查看升级历史
kubectl rollout history deployment nginx-deploy

# 暂停升级
kubectl rollout pause deployment nginx-deploy

# 继续升级
kubectl rollout resume deployment nginx-deploy

# 回滚到上一版本
kubectl rollout undo deployment nginx-deploy

# 回滚到指定版本
kubectl rollout undo deployment nginx-deploy --to-revision=2
```

### 滚动更新

```bash
# 更新镜像
kubectl set image deployment nginx-deploy nginx=nginx:1.21.0

# 或直接编辑
kubectl edit deployment nginx-deploy
```

### 金丝雀发布

金丝雀发布是一种灰度发布策略，先部署少量新版本实例，验证无误后再逐步扩大范围。

```bash
# 创建金丝雀部署（1个副本）
kubectl set image deployment nginx-deploy nginx=nginx:1.21.0
kubectl scale deployment nginx-deploy --replicas=1

# 验证通过后，全量更新
kubectl set image deployment nginx-deploy nginx=nginx:1.21.0
```

## Service

Service 用于暴露 Pod 的访问入口，提供稳定的 IP 和 DNS 名称。

### Service 类型

| 类型 | 说明 |
|------|------|
| ClusterIP | 默认类型，仅集群内部可访问 |
| NodePort | 通过节点端口暴露服务 |
| LoadBalancer | 使用云提供商的负载均衡器 |
| ExternalName | 映射到外部 DNS 名称 |

### 配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-pod
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

## ConfigMap 和 Secret

### ConfigMap

用于存储非敏感配置数据。

```bash
# 创建 ConfigMap
kubectl create configmap my-config --from-file=config.properties

# 或在 YAML 中定义
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  config.properties: |
    key1=value1
    key2=value2
```

### Secret

用于存储敏感数据（密码、密钥等）。

```bash
# 创建 Secret
kubectl create secret generic my-secret --from-literal=password=my-password

# 在 Pod 中使用
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
```

## 常见问题

### ❌ Pod 一直处于 Pending 状态

**原因**：节点资源不足或没有匹配的节点。

**解决**：

```bash
# 查看 Pod 事件
kubectl describe pod <pod-name>

# 查看节点资源
kubectl top nodes
```

### ❌ 镜像拉取失败

**原因**：镜像地址错误或网络问题。

**解决**：

```bash
# 检查镜像名称
kubectl get pod <pod-name> -o yaml | grep image

# 配置镜像仓库密钥
kubectl create secret docker-registry my-registry --docker-server=...
```

---

*以上就是 Kubernetes 核心概念的详解。K8s 内容丰富，后续文章会继续深入讲解各个组件。*
