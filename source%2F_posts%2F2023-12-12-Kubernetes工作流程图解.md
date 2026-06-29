---
title: "Kubernetes 工作流程图解：从 Pod 创建到流量转发"
date: 2023-12-12 11:00:00
categories: ["云原生"]
tags: ["Kubernetes", "容器编排", "工作流程", "图解"]
cover: /images/hero/04.jpg
summary: "通过流程图解深入理解 Kubernetes 的工作流程，从 Pod 创建到流量转发的完整过程。"
---

# Kubernetes 工作流程图解：从 Pod 创建到流量转发

## 引言

理解 Kubernetes 的工作流程对于掌握容器编排至关重要。本文通过流程图解的方式，详细讲解 K8s 的核心工作流程。

## 一、Pod 创建流程

### 1.1 创建 Pod 的完整流程

```
用户请求
    ↓
kubectl apply -f pod.yaml
    ↓
API Server
    ↓
etcd（持久化存储）
    ↓
Scheduler（调度）
    ↓
选择合适的 Node
    ↓
Kubelet（节点上的代理）
    ↓
创建 Pod 和容器
    ↓
返回 Pod 状态
```

### 1.2 详细步骤

1. **用户提交请求**：通过 kubectl 或 API 提交 Pod 创建请求
2. **API Server 接收**：验证请求并存储到 etcd
3. **Scheduler 调度**：根据节点资源、亲和性等选择合适节点
4. **Kubelet 创建**：在目标节点上创建 Pod 和容器
5. **状态反馈**：Pod 状态同步到 API Server

## 二、Service 流量转发流程

### 2.1 ClusterIP Service 流量流程

```
客户端请求
    ↓
Service ClusterIP:Port
    ↓
kube-proxy（IPVS/iptables）
    ↓
选择后端 Pod
    ↓
Pod IP:Port
    ↓
容器处理请求
```

### 2.2 kube-proxy 模式

#### 2.2.1 iptables 模式

传统的流量转发方式，通过 iptables 规则实现负载均衡：

```bash
# 查看 Service 的 iptables 规则
iptables -t nat -L | grep my-service
```

#### 2.2.2 IPVS 模式

高性能的负载均衡模式，适合大规模集群：

```bash
# 切换到 IPVS 模式
kubectl edit configmap kube-proxy -n kube-system
# mode: ipvs
```

## 三、Deployment 滚动更新流程

### 3.1 滚动更新流程

```
用户提交更新
    ↓
创建新 ReplicaSet
    ↓
逐步增加新 Pod
    ↓
逐步减少旧 Pod
    ↓
更新完成
```

### 3.2 更新策略

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多额外创建 1 个 Pod
      maxUnavailable: 0  # 更新过程中不能有不可用的 Pod
```

## 四、水平自动扩缩容流程

### 4.1 HPA 工作流程

```
Metrics Server 采集指标
    ↓
HPA Controller 评估
    ↓
比较当前副本数与目标副本数
    ↓
调整 Deployment 副本数
    ↓
滚动更新 Pod
```

### 4.2 HPA 配置示例

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## 五、容器网络流程

### 5.1 Pod 网络通信流程

```
Pod A
    ↓
CNI 插件（如 Calico/Flannel）
    ↓
虚拟网络
    ↓
Pod B
```

### 5.2 常见 CNI 插件

| 插件 | 特性 | 适用场景 |
|------|------|---------|
| Calico | BGP 路由，网络策略 | 大规模集群 |
| Flannel | VXLAN/Host-GW | 简单场景 |
| Weave | 自动配置 | 中小规模 |
| Cilium | eBPF 加速 | 高性能需求 |

## 六、存储卷挂载流程

### 6.1 Volume 挂载流程

```
Pod 创建请求
    ↓
Volume 配置
    ↓
CSI 驱动
    ↓
存储后端（PV/PVC）
    ↓
挂载到容器
```

### 6.2 存储类型

| 类型 | 说明 |
|------|------|
| EmptyDir | 临时存储，Pod 删除时丢失 |
| HostPath | 宿主机目录 |
| PersistentVolume | 持久化存储 |
| ConfigMap/Secret | 配置和敏感信息 |

## 总结

Kubernetes 的工作流程涉及多个组件的协作，理解这些流程对于排查问题和优化性能至关重要。通过流程图解，我们可以更直观地理解 K8s 的运行机制。

---

**参考资料**
- [Kubernetes 架构文档](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes 网络模型](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
