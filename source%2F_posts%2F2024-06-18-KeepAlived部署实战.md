---
title: "KeepAlived 部署实战：构建高可用集群"
date: 2024-06-18 10:00:00
categories: ["基础设施"]
tags: ["KeepAlived", "高可用", "负载均衡", "HA"]
cover: /images/hero/12.jpg
summary: "详细讲解 KeepAlived 的部署和配置，实现服务的高可用和故障转移。"
---

# KeepAlived 部署实战：构建高可用集群

## 引言

KeepAlived 是一个基于 VRRP（Virtual Router Redundancy Protocol）协议的高可用解决方案，广泛应用于构建高可用集群。本文将详细讲解 KeepAlived 的部署和配置。

## 一、VRRP 原理

### 1.1 VRRP 协议

VRRP（Virtual Router Redundancy Protocol）是一种容错协议，允许多台路由器组成一个虚拟路由器：

```
┌─────────────────────────────────────────┐
│           虚拟 IP (VIP)                 │
│         192.168.1.100                  │
└─────────────────┬───────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Master  │  │ Backup 1 │  │ Backup 2 │
│ (主节点) │  │ (备节点) │  │ (备节点) │
└──────────┘  └──────────┘  └──────────┘
    ↓
  处理请求
```

### 1.2 工作机制

| 角色 | 说明 |
|------|------|
| Master | 主节点，处理所有请求 |
| Backup | 备用节点，监听 Master 状态 |
| VIP | 虚拟 IP，对外提供服务 |

## 二、环境准备

### 2.1 节点规划

| 节点 | IP | 角色 |
|------|-----|------|
| Node1 | 192.168.1.10 | Master |
| Node2 | 192.168.1.11 | Backup |
| VIP | 192.168.1.100 | 虚拟 IP |

### 2.2 基础配置

```bash
# 配置 hosts
cat >> /etc/hosts << EOF
192.168.1.10 node1
192.168.1.11 node2
EOF

# 禁用防火墙
systemctl disable firewalld
systemctl stop firewalld

# 禁用 SELinux
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# 开启 IP 转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

## 三、安装 KeepAlived

```bash
# 安装依赖
dnf install -y gcc openssl-devel

# 下载 KeepAlived
wget https://www.keepalived.org/software/keepalived-2.2.8.tar.gz
tar -xzf keepalived-2.2.8.tar.gz
cd keepalived-2.2.8

# 编译安装
./configure --prefix=/usr/local/keepalived
make && make install

# 创建配置目录
mkdir -p /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/

# 创建系统服务
cat > /etc/systemd/system/keepalived.service << EOF
[Unit]
Description=Keepalived
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/keepalived/sbin/keepalived -f /etc/keepalived/keepalived.conf
ExecReload=/usr/local/keepalived/sbin/keepalived -f /etc/keepalived/keepalived.conf -r
ExecStop=/usr/local/keepalived/sbin/keepalived -f /etc/keepalived/keepalived.conf -s stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
```

## 四、配置 KeepAlived

### 4.1 Master 节点配置

```conf
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}
```

### 4.2 Backup 节点配置

```conf
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}
```

### 4.3 启动服务

```bash
# 在所有节点上启动 KeepAlived
systemctl start keepalived
systemctl enable keepalived

# 检查状态
systemctl status keepalived

# 查看 VIP 绑定情况
ip addr show eth0
```

## 五、高级配置

### 5.1 配置通知脚本

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    notify_master /etc/keepalived/notify.sh master
    notify_backup /etc/keepalived/notify.sh backup
    notify_fault /etc/keepalived/notify.sh fault
    
    virtual_ipaddress {
        192.168.1.100
    }
}
```

```bash
#!/bin/bash
# /etc/keepalived/notify.sh

case "$1" in
    master)
        echo "KeepAlived 切换为 MASTER" >> /var/log/keepalived.log
        # 启动服务
        systemctl start nginx
        ;;
    backup)
        echo "KeepAlived 切换为 BACKUP" >> /var/log/keepalived.log
        # 停止服务
        systemctl stop nginx
        ;;
    fault)
        echo "KeepAlived 检测到故障" >> /var/log/keepalived.log
        ;;
esac

chmod +x /etc/keepalived/notify.sh
```

### 5.2 配置健康检查

```conf
vrrp_script chk_http_port {
    script "/etc/keepalived/check_http.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    track_script {
        chk_http_port
    }
    
    virtual_ipaddress {
        192.168.1.100
    }
}
```

```bash
#!/bin/bash
# /etc/keepalived/check_http.sh

if curl -s http://localhost:80 > /dev/null; then
    exit 0
else
    exit 1
fi

chmod +x /etc/keepalived/check_http.sh
```

### 5.3 多实例配置

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    virtual_ipaddress {
        192.168.1.100
    }
}

vrrp_instance VI_2 {
    state BACKUP
    interface eth0
    virtual_router_id 52
    priority 90
    advert_int 1
    virtual_ipaddress {
        192.168.1.101
    }
}
```

## 六、验证高可用

### 6.1 测试故障转移

```bash
# 在 Master 节点上停止 KeepAlived
systemctl stop keepalived

# 在 Backup 节点上检查 VIP
ip addr show eth0

# 在 Master 节点上恢复 KeepAlived
systemctl start keepalived

# 检查 VIP 是否切回
ip addr show eth0
```

### 6.2 测试服务可用性

```bash
# 通过 VIP 访问服务
curl http://192.168.1.100

# 停止 Master 节点的服务
systemctl stop nginx

# 再次访问，应该由 Backup 节点处理
curl http://192.168.1.100
```

## 总结

KeepAlived 基于 VRRP 协议实现高可用集群，通过虚拟 IP 实现故障转移。配置健康检查和通知脚本可以实现更完善的高可用方案。

---

**参考资料**
- [KeepAlived 官方文档](https://keepalived.readthedocs.io/)
- [VRRP 协议规范](https://datatracker.ietf.org/doc/html/rfc5798)
