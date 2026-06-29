---
title: "Nginx 虚拟主机配置实战：一个服务器托管多个网站"
date: 2024-06-14 10:00:00
categories: ["基础设施"]
tags: ["Nginx", "虚拟主机", "Web服务器", "配置"]
cover: /images/hero/08.jpg
summary: "详细讲解 Nginx 虚拟主机配置，实现一个服务器托管多个网站。"
---

# Nginx 虚拟主机配置实战：一个服务器托管多个网站

## 引言

Nginx 是一个高性能的 Web 服务器和反向代理服务器，支持基于域名的虚拟主机配置。本文将详细讲解如何配置虚拟主机，实现一个服务器托管多个网站。

## 一、虚拟主机类型

### 1.1 基于域名的虚拟主机

```
┌─────────────────────────────────────┐
│           Nginx Server              │
├─────────────────────────────────────┤
│  example.com → /var/www/example    │
│  blog.com    → /var/www/blog       │
│  api.com     → /var/www/api        │
└─────────────────────────────────────┘
```

### 1.2 基于 IP 的虚拟主机

```
┌─────────────────────────────────────┐
│           Nginx Server              │
├─────────────────────────────────────┤
│  192.168.1.10 → /var/www/site1     │
│  192.168.1.11 → /var/www/site2     │
│  192.168.1.12 → /var/www/site3     │
└─────────────────────────────────────┘
```

### 1.3 基于端口的虚拟主机

```
┌─────────────────────────────────────┐
│           Nginx Server              │
├─────────────────────────────────────┤
│  :80   → /var/www/default          │
│  :8080 → /var/www/admin            │
│  :8443 → /var/www/api              │
└─────────────────────────────────────┘
```

## 二、配置基于域名的虚拟主机

### 2.1 创建网站目录

```bash
mkdir -p /var/www/example.com/public_html
mkdir -p /var/www/blog.com/public_html
mkdir -p /var/www/api.com/public_html

# 设置权限
chown -R nginx:nginx /var/www/
chmod -R 755 /var/www/
```

### 2.2 创建配置文件

```bash
# 创建配置目录
mkdir /etc/nginx/sites-available
mkdir /etc/nginx/sites-enabled

# 修改 nginx.conf
cat >> /etc/nginx/nginx.conf << EOF
include /etc/nginx/sites-enabled/*.conf;
EOF
```

### 2.3 配置示例网站

```nginx
# /etc/nginx/sites-available/example.com.conf
server {
    listen 80;
    server_name example.com www.example.com;

    root /var/www/example.com/public_html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;
}
```

### 2.4 配置博客网站

```nginx
# /etc/nginx/sites-available/blog.com.conf
server {
    listen 80;
    server_name blog.com www.blog.com;

    root /var/www/blog.com/public_html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # 缓存静态文件
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 7d;
        add_header Cache-Control "public, no-transform";
    }

    access_log /var/log/nginx/blog.com.access.log;
    error_log /var/log/nginx/blog.com.error.log;
}
```

### 2.5 配置 API 服务

```nginx
# /etc/nginx/sites-available/api.com.conf
server {
    listen 80;
    server_name api.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API 限流
    limit_req zone=api burst=10 nodelay;

    access_log /var/log/nginx/api.com.access.log;
    error_log /var/log/nginx/api.com.error.log;
}
```

## 三、启用配置

```bash
# 创建符号链接
ln -s /etc/nginx/sites-available/example.com.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/blog.com.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/api.com.conf /etc/nginx/sites-enabled/

# 检查配置
nginx -t

# 重启 Nginx
systemctl restart nginx
```

## 四、配置 HTTPS

### 4.1 安装 Certbot

```bash
# 安装 Certbot
dnf install certbot python3-certbot-nginx -y

# 获取证书
certbot --nginx -d example.com -d www.example.com
certbot --nginx -d blog.com -d www.blog.com
certbot --nginx -d api.com
```

### 4.2 HTTPS 配置示例

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # TLS 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    root /var/www/example.com/public_html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

## 五、高级配置

### 5.1 负载均衡配置

```nginx
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com weight=3;
    server backend3.example.com;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
    }
}
```

### 5.2 限流配置

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://localhost:3000;
    }
}
```

## 总结

通过 Nginx 的虚拟主机功能，可以在一台服务器上托管多个网站。配置基于域名的虚拟主机是最常见的方式，配合 HTTPS 和负载均衡，可以构建高性能的 Web 服务。

---

**参考资料**
- [Nginx 官方文档](http://nginx.org/en/docs/)
- [Certbot 文档](https://certbot.eff.org/docs/)
