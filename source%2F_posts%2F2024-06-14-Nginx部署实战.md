---
title: "Nginx 部署实战：从安装到生产环境配置"
date: 2024-06-14 11:00:00
categories: ["基础设施"]
tags: ["Nginx", "Web服务", "反向代理", "生产实践"]
cover: /images/hero/09.jpg
summary: "详细讲解 Nginx 的安装和生产环境配置，包括性能优化和安全加固。"
---

# Nginx 部署实战：从安装到生产环境配置

## 目录

- [引言](#引言)
- [一、安装 Nginx](#一安装-nginx)
- [二、基础配置](#二基础配置)
- [三、生产环境优化](#三生产环境优化)
- [四、日志管理](#四日志管理)
- [五、监控和告警](#五监控和告警)
- [总结](#总结)

## 引言

在现代 Web 架构中，Nginx 已经成为不可或缺的核心组件。无论是作为 Web 服务器、反向代理还是负载均衡器，Nginx 凭借其高性能、高并发和低资源消耗的特点，占据了生产环境的主导地位。

本文将从基础安装到高级配置，全面讲解如何在生产环境中部署和优化 Nginx。

## 一、安装 Nginx

### 1.1 使用官方源安装（推荐）

```bash
# 安装官方源
cat > /etc/yum.repos.d/nginx.repo << EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/releasever/basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

# 安装 Nginx
dnf install nginx -y

# 启动 Nginx
systemctl start nginx
systemctl enable nginx
```

### 1.2 源码编译安装

```bash
# 安装依赖
dnf install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel -y

# 下载源码
wget http://nginx.org/download/nginx-1.25.3.tar.gz
tar -xzf nginx-1.25.3.tar.gz
cd nginx-1.25.3

# 配置编译选项
./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_gzip_static_module \
    --with-stream \
    --with-http_realip_module

# 编译安装
make && make install

# 创建系统服务
cat > /etc/systemd/system/nginx.service << EOF
[Unit]
Description=nginx - high performance web server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start nginx
systemctl enable nginx
```

## 二、基础配置

### 2.1 主配置文件

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main 'remote_addr - remote_user [time_local] "request" '
                    'status body_bytes_sent "http_referer" '
                    '"http_user_agent" "http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 65;

    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*.conf;
}
```

### 2.2 服务器块配置

```nginx
# /etc/nginx/conf.d/default.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## 三、生产环境优化

### 3.1 性能优化

```nginx
# worker_processes 设置为 CPU 核心数
worker_processes auto;

# 增加 worker_connections
worker_connections 4096;

# 启用 sendfile 和 tcp_nopush
sendfile on;
tcp_nopush on;
tcp_nodelay on;

# 调整 keepalive_timeout
keepalive_timeout 65;
keepalive_requests 100;

# 启用 gzip 压缩
gzip on;
gzip_vary on;
gzip_min_length 256;
gzip_types text/plain text/css application/json application/javascript;
```

### 3.2 安全加固

```nginx
# 隐藏 Nginx 版本号
server_tokens off;

# 限制请求大小
client_max_body_size 10M;

# 防止点击劫持
add_header X-Frame-Options DENY;

# 防止 MIME 类型嗅探
add_header X-Content-Type-Options nosniff;

# 启用 HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

## 四、日志管理

### 4.1 配置日志轮转

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 nginx nginx
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /var/run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

### 4.2 自定义日志格式

```nginx
log_format json '
    {
        "timestamp": "$time_iso8601",
        "remote_addr": "$remote_addr",
        "remote_user": "$remote_user",
        "request": "$request",
        "status": "$status",
        "body_bytes_sent": "$body_bytes_sent",
        "http_referer": "$http_referer",
        "http_user_agent": "$http_user_agent",
        "request_time": "$request_time"
    }
';

access_log /var/log/nginx/access.log json;
```

## 五、监控和告警

### 5.1 配置 Prometheus 监控

```nginx
server {
    listen 9113;
    server_name localhost;

    location /metrics {
        stub_status on;
        access_log off;
    }
}
```

### 5.2 配置健康检查

```bash
# 检查 Nginx 状态
nginx -t

systemctl is-active nginx

# 监控端口
ss -tlnp | grep nginx
```

## 总结

Nginx 的生产环境配置需要综合考虑多个方面：

1. **性能优化**：合理调整 worker_processes、worker_connections 等参数
2. **安全加固**：隐藏版本号、配置安全头、限制请求大小
3. **日志管理**：配置日志轮转和自定义格式便于分析
4. **监控告警**：配置 Prometheus 监控和健康检查

通过合理的配置，可以构建高性能、高安全的 Web 服务基础设施。

---

**参考资料**
- [Nginx 官方文档](http://nginx.org/en/docs/)
- [Nginx 性能优化指南](https://docs.nginx.com/nginx/admin-guide/web-server/tuning/)
