---
title: "OpenStack 搭建实战：从零开始部署生产级云平台"
date: 2023-12-13 10:00:00
categories: ["云原生"]
tags: ["OpenStack", "云计算", "IaaS", "部署"]
cover: /images/hero/06.jpg
summary: "详细讲解如何从零开始搭建生产级 OpenStack 云平台，包括环境准备、组件部署和验证测试。"
---

# OpenStack 搭建实战：从零开始部署生产级云平台

## 引言

OpenStack 是一个强大的开源云计算平台，本文将详细讲解如何从零开始搭建生产级 OpenStack 云平台。

## 一、环境准备

### 1.1 硬件要求

| 角色 | CPU | 内存 | 存储 | 网卡 |
|------|-----|------|------|------|
| Controller | 8核+ | 16GB+ | 200GB+ | 至少2个 |
| Compute | 16核+ | 32GB+ | 500GB+ | 至少2个 |
| Storage | 8核+ | 16GB+ | 1TB+ | 至少2个 |

### 1.2 操作系统选择

推荐使用 CentOS 8 或 Rocky Linux：

```bash
# 检查系统版本
cat /etc/redhat-release

# 确保系统已更新
dnf update -y
```

### 1.3 网络规划

```
Management Network: 10.0.0.0/24
Provider Network:   192.168.1.0/24
Overlay Network:    172.16.0.0/16
```

## 二、基础配置

### 2.1 禁用防火墙和 SELinux

```bash
# 禁用防火墙
systemctl disable firewalld
systemctl stop firewalld

# 禁用 SELinux
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 2.2 配置主机名和 hosts

```bash
# 设置主机名
hostnamectl set-hostname controller

# 配置 hosts
cat >> /etc/hosts << EOF
10.0.0.10 controller
10.0.0.20 compute1
10.0.0.30 storage1
EOF
```

### 2.3 安装基础依赖

```bash
dnf install -y wget git vim net-tools
```

## 三、使用 Kolla 部署

### 3.1 安装 Kolla Ansible

```bash
# 创建虚拟环境
python3 -m venv /opt/kolla
source /opt/kolla/bin/activate

# 安装 Kolla Ansible
pip install kolla-ansible

# 创建配置目录
mkdir -p /etc/kolla
cp -r /usr/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```

### 3.2 配置 inventory

```ini
[control]
controller

[network]
controller

[compute]
compute1

[monitoring]
controller

[storage]
storage1

[deployment]
localhost ansible_connection=local
```

### 3.3 配置 globals.yml

```yaml
global:
  kolla_base_distro: centos
  kolla_install_type: source
  openstack_release: yoga
  kolla_internal_vip_address: 10.0.0.100
  network_interface: eth0
  neutron_external_interface: eth1
  enable_haproxy: yes
  enable_keepalived: yes
```

### 3.4 部署 OpenStack

```bash
# 生成密码
kolla-genpwd

# 部署
kolla-ansible deploy -i multinode

# 验证
kolla-ansible post-deploy
```

## 四、初始化 OpenStack

### 4.1 设置环境变量

```bash
source /etc/kolla/admin-openrc.sh
```

### 4.2 创建必要资源

```bash
# 创建外部网络
export EXT_NET_CIDR='192.168.1.0/24'
export EXT_NET_RANGE='start=192.168.1.100,end=192.168.1.200'
export EXT_NET_GATEWAY='192.168.1.1'

openstack network create --external --provider-physical-network public \n  --provider-network-type flat public
openstack subnet create --network public \n  --subnet-range $EXT_NET_CIDR \n  --allocation-pool $EXT_NET_RANGE \n  --gateway $EXT_NET_GATEWAY public-subnet

# 创建租户网络
openstack network create private
openstack subnet create --network private --subnet-range 10.0.0.0/24 private-subnet

# 创建路由器
openstack router create router
openstack router add subnet router private-subnet
openstack router set --external-gateway public router
```

## 五、验证部署

### 5.1 创建测试实例

```bash
# 创建密钥对
openstack keypair create mykey > mykey.pem
chmod 600 mykey.pem

# 创建安全组
openstack security group create allow-all
openstack security group rule create --protocol icmp allow-all
openstack security group rule create --protocol tcp --port-range 22 allow-all

# 创建实例
openstack server create --flavor m1.small --image cirros \n  --key-name mykey --security-group allow-all \n  --network private test-vm

# 分配浮动 IP
openstack floating ip create public
openstack server add floating ip test-vm <floating-ip>
```

### 5.2 验证功能

```bash
# 检查服务状态
openstack service list
openstack endpoint list

# 检查计算节点
openstack hypervisor list

# 检查存储
openstack volume type list
```

## 六、常见问题排查

### 6.1 服务启动失败

```bash
# 查看日志
docker logs -f <container-name>

# 检查配置
cat /etc/kolla/<service>/<config-file>
```

### 6.2 网络不通

```bash
# 检查网络配置
openstack network list
openstack subnet list
openstack router list

# 检查安全组
openstack security group rule list
```

## 总结

通过 Kolla Ansible 可以快速部署生产级 OpenStack 云平台。部署完成后，需要创建必要的网络资源和安全组，才能创建和访问虚拟机实例。

---

**参考资料**
- [Kolla Ansible 文档](https://docs.openstack.org/kolla-ansible/)
- [OpenStack 安装指南](https://docs.openstack.org/install-guide/)
