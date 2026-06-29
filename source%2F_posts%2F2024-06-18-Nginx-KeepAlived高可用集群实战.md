---
title: "Nginx + KeepAlived 高可用集群实战：企业级架构方案"
date: 2024-06-18 11:00:00
categories: ["基础设施"]
tags: ["Nginx", "KeepAlived", "高可用", "负载均衡", "企业级"]
cover: /images/hero/13.jpg
summary: "详细讲解如何构建企业级 Nginx + KeepAlived 高可用集群，实现服务的高可用和负载均衡。"
---

# Nginx + KeepAlived 高可用集群实战：企业级架构方案

## 引言

在企业级应用中，高可用性是至关重要的。本文将详细讲解如何构建 Nginx + KeepAlived 高可用集群，实现服务的高可用和负载均衡。

## 一、架构设计

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        用户请求                            │
└──────────────────────────┬────────────────────────────────┘
                           │
                           ▼
                    Virtual IP (VIP)
                    192.168.1.100
                           │
            ┌──────────────┴──────────────┐
            ▼                             ▼
    ┌─────────────┐              ┌─────────────┐
    │   Nginx     │              │   Nginx     │
    │  (Master)   │              │  (Backup)   │
    │ KeepAlived  │◀── VRRP ──▶│ KeepAlived  │
    └──────┬──────┘              └──────┬──────┘
           │                             │
    ┌──────┴──────┐              ┌──────┴──────┐
    ▼             ▼              ▼             ▼
┌─────────┐ ┌─────────┐    ┌─────────┐ ┌─────────┐
│ Web 1   │ │ Web 2   │    │ Web 3   │ │ Web 4   │
└─────────┘ └─────────┘    └─────────┘ └─────────┘
```

### 1.2 组件职责

| 组件 | 职责 |
|------|------|
| Nginx | 反向代理和负载均衡 |
| KeepAlived | 高可用和故障转移 |
| VIP | 虚拟 IP，对外提供服务 |
| Web Server | 实际处理请求的后端服务 |

## 二、环境准备

### 2.1 节点规划

| 节点 | IP | 角色 |
|------|-----|------|
| LB1 | 192.168.1.10 | Nginx + KeepAlived (Master) |
| LB2 | 192.168.1.11 | Nginx + KeepAlived (Backup) |
| VIP | 192.168.1.100 | 虚拟 IP |
| Web1 | 192.168.2.10 | 后端 Web 服务器 |
| Web2 | 192.168.2.11 | 后端 Web 服务器 |
| Web3 | 192.168.2.12 | 后端 Web 服务器 |

### 2.2 基础配置

```bash
# 在所有节点上执行
cat >> /etc/hosts << EOF
192.168.1.10 lb1
192.168.1.11 lb2
192.168.2.10 web1
192.168.2.11 web2
192.168.2.12 web3
EOF

systemctl disable firewalld
systemctl stop firewalld
setenforce 0
```

## 三、部署后端 Web 服务器

### 3.1 安装配置 Web 服务器

```bash
# 在所有 Web 节点上执行
dnf install httpd -y

echo "<h1>Web Server 1</h1>" > /var/www/html/index.html

systemctl start httpd
systemctl enable httpd
```

## 四、部署 Nginx

### 4.1 安装 Nginx

```bash
# 在所有 LB 节点上执行
dnf install nginx -y
```

### 4.2 配置 Nginx 负载均衡

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream backend {
        server 192.168.2.10:80 weight=3;
        server 192.168.2.11:80 weight=2;
        server 192.168.2.12:80 weight=1;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 健康检查
        location /health {
            proxy_pass http://backend;
            access_log off;
        }
    }
}
```

### 4.3 启动 Nginx

```bash
systemctl start nginx
systemctl enable nginx
```

## 五、部署 KeepAlived

### 5.1 安装 KeepAlived

```bash
# 在所有 LB 节点上执行
dnf install keepalived -y
```

### 5.2 Master 节点配置

```conf
! Configuration File for keepalived

global_defs {
    router_id NGINX_MASTER
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
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
        192.168.1.100/24 dev eth0
    }
    
    track_script {
        chk_nginx
    }
    
    notify_master /etc/keepalived/notify.sh master
    notify_backup /etc/keepalived/notify.sh backup
    notify_fault /etc/keepalived/notify.sh fault
}
```

### 5.3 Backup 节点配置

```conf
! Configuration File for keepalived

global_defs {
    router_id NGINX_BACKUP
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
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
        192.168.1.100/24 dev eth0
    }
    
    track_script {
        chk_nginx
    }
}
```

### 5.4 创建健康检查脚本

```bash
#!/bin/bash
# /etc/keepalived/check_nginx.sh

if [ -z "$(ps aux | grep nginx | grep -v grep)" ]; then
    exit 1
fi

exit 0

chmod +x /etc/keepalived/check_nginx.sh
```

### 5.5 创建通知脚本

```bash
#!/bin/bash
# /etc/keepalived/notify.sh

case "$1" in
    master)
        echo "$(date) - KeepAlived 切换为 MASTER" >> /var/log/keepalived.log
        systemctl start nginx
        ;;
    backup)
        echo "$(date) - KeepAlived 切换为 BACKUP" >> /var/log/keepalived.log
        systemctl stop nginx
        ;;
    fault)
        echo "$(date) - KeepAlived 检测到故障" >> /var/log/keepalived.log
        ;;
esac

chmod +x /etc/keepalived/notify.sh
```

### 5.6 启动 KeepAlived

```bash
systemctl start keepalived
systemctl enable keepalived
```

## 六、验证高可用集群

### 6.1 检查 VIP 状态

```bash
# 在 Master 节点上检查
ip addr show eth0

# 在 Backup 节点上检查
ip addr show eth0
```

### 6.2 测试负载均衡

```bash
# 通过 VIP 访问服务
curl http://192.168.1.100
curl http://192.168.1.100
curl http://192.168.1.100
```

### 6.3 测试故障转移

```bash
# 在 Master 节点上停止 Nginx
systemctl stop nginx

# 在 Backup 节点上检查 VIP
ip addr show eth0

# 通过 VIP 访问服务
curl http://192.168.1.100

# 在 Master 节点上恢复 Nginx
systemctl start nginx

# 检查 VIP 是否切回
ip addr show eth0
```

## 七、高级配置

### 7.1 配置 SSL/TLS

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://backend;
    }
}
```

### 7.2 配置会话保持

```nginx
upstream backend {
    ip_hash;
    server 192.168.2.10:80;
    server 192.168.2.11:80;
    server 192.168.2.12:80;
}
```

### 7.3 配置限流

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location / {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

## 八、监控和告警

### 8.1 配置 Prometheus 监控

```bash
# 安装 Prometheus 和 Grafana
dnf install prometheus grafana -y

# 配置 Prometheus
cat > /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'nginx'
    static_configs:
      - targets: ['lb1:9113', 'lb2:9113']
  - job_name: 'keepalived'
    static_configs:
      - targets: ['lb1:9168', 'lb2:9168']
EOF

systemctl start prometheus
systemctl start grafana
systemctl enable prometheus
systemctl enable grafana
```

## 总结

Nginx + KeepAlived 高可用集群是企业级应用的标准架构方案。通过负载均衡和故障转移，可以实现服务的高可用性和可扩展性。

---

**参考资料**
- [Nginx 官方文档](http://nginx.org/en/docs/)
- [KeepAlived 官方文档](https://keepalived.readthedocs.io/)
