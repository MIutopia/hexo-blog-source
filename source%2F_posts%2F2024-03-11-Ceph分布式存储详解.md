---
title: "Ceph 分布式存储详解：从原理到部署实战"
date: 2024-03-11 10:00:00
categories: ["基础设施"]
tags: ["Ceph", "分布式存储", "存储", "CRUSH"]
cover: /images/hero/07.jpg
summary: "深入理解 Ceph 分布式存储的原理，掌握部署和运维技巧。"
---

# Ceph 分布式存储详解：从原理到部署实战

## 目录

- [引言](#引言)
- [一、Ceph 架构原理](#一ceph-架构原理)
- [二、部署 Ceph](#二部署-ceph)
- [三、创建存储池](#三创建存储池)
- [四、Ceph 运维](#四ceph-运维)
- [五、性能优化](#五性能优化)
- [总结](#总结)

## 引言

Ceph 是一个强大的开源分布式存储系统，以其高可用、高扩展和高性能的特点，成为企业级存储解决方案的首选。它支持对象存储、块存储和文件存储三种接口，适用于多种场景。

本文将深入讲解 Ceph 的架构原理和部署方法，帮助你掌握分布式存储的核心技术。

## 一、Ceph 架构原理

### 1.1 Ceph 核心组件

```plaintext
┌──────────────────────────────────────────────────────┐
│                     Client                          │
└──────────────────────────┬─────────────────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    ▼                      ▼                      ▼
┌──────────┐          ┌──────────┐          ┌──────────┐
│ Monitor  │          │ OSD      │          │ MDS      │
│ (监控)   │          │ (对象存储)│          │ (元数据) │
└──────────┘          └──────────┘          └──────────┘
    │                      │                      │
    └──────────────────────┼──────────────────────┘
                           │
                       RADOS
                    (可靠、自动分布的对象存储)
```

### 1.2 CRUSH 算法

Ceph 使用 CRUSH（Controlled Replication Under Scalable Hashing）算法进行数据分布，这是 Ceph 的核心优势：

```plaintext
数据对象
    ↓
计算 hash
    ↓
CRUSH 规则
    ↓
确定存储位置
    ↓
写入 OSD
```

> **CRUSH 算法优势**：
> - 数据分布均匀，无需中心化元数据管理
> - 支持动态扩展，添加节点后自动重新分布
> - 支持自定义故障域，提高数据可靠性

### 1.3 数据副本策略

```yaml
global:
  osd_pool_default_size: 3           # 副本数
  osd_pool_default_min_size: 2       # 最小副本数
  osd_pool_default_pg_num: 128       # PG 数量
  osd_pool_default_pgp_num: 128      # PG 放置组
```

## 二、部署 Ceph

### 2.1 环境准备

```bash
# 节点要求
# 至少 3 个节点，每个节点至少 2 块磁盘

# 配置 hosts
cat >> /etc/hosts << EOF
10.0.0.10 ceph-mon1
10.0.0.11 ceph-mon2
10.0.0.12 ceph-mon3
EOF

# 禁用防火墙和 SELinux
systemctl disable firewalld
systemctl stop firewalld
setenforce 0
```

### 2.2 使用 ceph-ansible 部署

```bash
# 安装 ansible
pip install ansible

# 克隆 ceph-ansible
git clone https://github.com/ceph/ceph-ansible.git
cd ceph-ansible

# 配置 inventory
cat > inventory << EOF
[mons]
ceph-mon1
ceph-mon2
ceph-mon3

[osds]
ceph-mon1
ceph-mon2
ceph-mon3

[mgrs]
ceph-mon1
ceph-mon2
EOF

# 配置 group_vars
cat > group_vars/all.yml << EOF
ceph_origin: repository
ceph_repository: community
ceph_stable_release: quincy
monitor_interface: eth0
public_network: 10.0.0.0/24
cluster_network: 10.0.1.0/24
osd_scenario: collocated
devices:
  - /dev/sdb
  - /dev/sdc
EOF

# 部署
ansible-playbook site.yml
```

## 三、创建存储池

### 3.1 创建块存储池

```bash
# 创建池
ceph osd pool create rbd 128 128

# 初始化 RBD
rbd pool init rbd

# 创建镜像
rbd create myimage --size 10G --pool rbd

# 映射镜像
rbd map myimage --pool rbd
```

### 3.2 创建对象存储池

```bash
# 创建池
ceph osd pool create cephfs_data 128 128
ceph osd pool create cephfs_metadata 128 128

# 创建 CephFS
ceph fs new cephfs cephfs_metadata cephfs_data

# 挂载 CephFS
mount -t ceph ceph-mon1:6789:/ /mnt/cephfs
```

## 四、Ceph 运维

### 4.1 检查集群状态

```bash
# 查看集群状态
ceph -s

# 查看 OSD 状态
ceph osd tree

# 查看 PG 状态
ceph pg stat

# 查看监控状态
ceph mon stat
```

### 4.2 添加 OSD

```bash
# 准备磁盘
ceph-volume lvm create --data /dev/sdd

# 检查新 OSD
ceph osd tree
```

### 4.3 故障处理

```bash
# 标记 OSD 为 down
ceph osd down osd.0

# 标记 OSD 为 out
ceph osd out osd.0

# 删除 OSD
ceph osd crush remove osd.0
ceph auth del osd.0
ceph osd rm osd.0

# 重新加入 OSD
ceph osd in osd.0
```

## 五、性能优化

### 5.1 PG 数量调整

```bash
# 计算 PG 数量
# Total PGs = (Total OSDs × 100) / Number of replicas

# 修改 PG 数量
ceph osd pool set rbd pg_num 256
ceph osd pool set rbd pgp_num 256
```

### 5.2 OSD 配置优化

```ini
[osd]
osd_memory_target = 4G
osd_max_write_size = 5242880
osd_client_message_size_cap = 2147483648
```

## 总结

Ceph 是一个功能强大的分布式存储系统，核心优势包括：

1. **三种存储接口**：支持对象存储、块存储和文件存储
2. **CRUSH 算法**：实现数据的自动分布和冗余
3. **高可用**：多副本机制保证数据可靠性
4. **高扩展**：线性扩展，添加节点即可提升性能

通过合理的部署和配置，可以构建高性能、高可靠的分布式存储服务。

---

**参考资料**
- [Ceph 官方文档](https://docs.ceph.com/)
- [Ceph 部署指南](https://docs.ceph.com/projects/ceph-ansible/)
