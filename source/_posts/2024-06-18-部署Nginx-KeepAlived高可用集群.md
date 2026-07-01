---
title: 部署 Nginx + Keepalived 高可用集群
date: 2024-06-18 10:00:00
categories:
  - 负载均衡
tags:
  - Nginx
  - Keepalived
  - 高可用
---

## 前言

使用 Nginx 与 Keepalived 构建高可用集群，是提升系统稳定性和可靠性的常见方案。这篇文章从概念讲起，逐步搭建完整的高可用环境。

## 核心概念

### Nginx

Nginx 是一款高性能的 HTTP 和反向代理服务器，主要应用于负载均衡和静态内容服务，同时也提供了 IMAP/SMTP/POP3 服务。

### Keepalived

Keepalived 是一个用于监听和故障转移的工具，基于 VRRP 协议（虚拟路由冗余协议）实现。它可以监控服务器状态，并在主服务器故障时自动切换到备份服务器。

Keepalived 提供了几种冗余方式：
- 主备模式（Master-Backup）
- 负载均衡模式（Load Balancing）
- 主备负载均衡模式

### 双机高可用方案

- Keepalived + Nginx 双机主从模式
- Keepalived + Nginx 双机主主模式

## Nginx 核心功能

### 正向代理

帮助内网客户端访问外网服务器的代理方式。客户端无法直接访问目标服务器，但可以通过代理服务器进行访问。

**主要目的**：解决访问限制问题，如绕过地区限制或组织内部的访问限制。

### 反向代理

把外网客户端的请求转发给内网服务器的代理方式。客户端并不知道实际提供服务的服务器地址，只与代理服务器进行交互。

**作用**：
- 作为负载均衡器，将客户端请求分发到多个后端服务器
- 安全防护，过滤恶意请求和响应

### 负载均衡

将工作负载均匀地分配到多个服务器上，防止任何单一节点过载。

**度量标准**：CPU 使用率、内存使用率、磁盘 I/O、网络带宽、并发用户数量等。

**分类**：
- **客户端负载均衡**：发生在客户端，由自己决定访问哪台服务器
- **服务端负载均衡**：通过专门的负载均衡器（如 Nginx、HAProxy）分发请求

## 环境搭建

### 1. 安装依赖

```bash
yum -y install gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

### 2. 下载并解压 Nginx

```bash
# 创建目录
mkdir /opt/nginx
cd /opt/nginx

# 下载并解压
wget http://nginx.org/download/nginx-1.26.1.tar.gz
tar -xzvf nginx-1.26.1.tar.gz
```

### 3. 编译安装

```bash
# 创建安装目录
mkdir nginx-1.26.1_install

# 进入源码目录
cd nginx-1.26.1

# 配置
./configure --prefix=/opt/nginx/nginx-1.26.1_install

# 编译安装
make && make install
```

### 4. 启动 Nginx

```bash
# 检查配置
/opt/nginx/nginx-1.26.1_install/sbin/nginx -t

# 启动
/opt/nginx/nginx-1.26.1_install/sbin/nginx

# 验证
curl http://192.168.10.100:80
```

## Keepalived 配置

### 主节点配置

```bash
vi /opt/nginx/keepalived/etc/keepalived/keepalived.conf
```

```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
   }
   router_id LVS_DEVEL
}

vrrp_script chk_nginx {
    script "/opt/nginx/keepalived/check_nginx.sh"
    interval 2
    weight 2
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
        192.168.10.100
    }
    track_script {
        chk_nginx
    }
}
```

### 检查脚本

```bash
vi /opt/nginx/keepalived/check_nginx.sh
```

```bash
#!/bin/bash
if ! pgrep -x nginx > /dev/null; then
    systemctl start nginx
    if ! pgrep -x nginx > /dev/null; then
        systemctl stop keepalived
    fi
fi
```

```bash
chmod +x /opt/nginx/keepalived/check_nginx.sh
```

### 备节点配置

与主节点类似，修改以下参数：

```
state BACKUP
priority 90
```

## 启动服务

```bash
# 主节点
/opt/nginx/keepalived/sbin/keepalived

# 备节点
/opt/nginx/keepalived/sbin/keepalived

# 验证 VIP
ip addr show eth0
```

## 常见问题

### ❌ VIP 无法访问

**原因**：Keepalived 未正确启动或配置错误。

**解决**：

```bash
# 检查 Keepalived 状态
ps -ef | grep keepalived

# 检查日志
tail -f /var/log/messages
```

### ❌ Nginx 故障未切换

**原因**：检查脚本未正确执行。

**解决**：

```bash
# 手动测试脚本
/opt/nginx/keepalived/check_nginx.sh

# 检查脚本权限
chmod +x /opt/nginx/keepalived/check_nginx.sh
```

---

*以上就是 Nginx + Keepalived 高可用集群的完整搭建过程。*
