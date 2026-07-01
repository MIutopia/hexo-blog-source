---
title: Keepalived 部署实战
date: 2024-06-18 10:00:00
categories:
  - 负载均衡
tags:
  - Keepalived
  - 高可用
---

## 前言

Keepalived 是一个用于 Linux 服务器的轻量级高可用解决方案。这篇文章记录了在 CentOS 上从源码编译安装 Keepalived 的完整过程，为后续搭建 Nginx + Keepalived 高可用集群做准备。

## 安装依赖

```bash
# 安装 libnl-3 开发库
sudo yum install libnl3-devel libnl3-cli-devel
```

> 注：本次部署专注于构建基于 Nginx 和 Keepalived 的高可用集群，已安装的 Nginx 相关工具将不再重复安装。

## 下载并解压 Keepalived

```bash
# 进入目录
cd /opt/nginx

# 解压 Keepalived
tar -xvzf keepalived-2.3.1.tar.gz
```

> 官网下载地址：[Keepalived for Linux](https://keepalived.org/download.html)

## 编译安装

```bash
# 创建安装目录
mkdir /opt/nginx/keepalived

# 进入源码目录
cd /opt/nginx/keepalived-2.3.1

# 配置
./configure --prefix=/opt/nginx/keepalived

# 编译安装
make && make install
```

## 验证安装

编译安装完成后，如果显示以下内容，则表示安装成功：

```
Use IPVS Framework       : Yes
IPVS use libnl           : Yes
Use VRRP Framework       : Yes
Use VRRP VMAC            : Yes
Use VRRP authentication  : Yes
With track_process       : Yes
With linkbeat            : Yes
```

> **WARNING**：如果出现警告，可能缺少 Keepalived 所需的第三方库/工具。

## 常见问题

### ❌ 编译报错：缺少依赖

**原因**：缺少 libnl 或 openssl 开发库。

**解决**：

```bash
yum install -y libnl3-devel openssl-devel
```

### ❌ 启动失败

**原因**：配置文件错误或权限不足。

**解决**：

```bash
# 检查配置文件
cat /opt/nginx/keepalived/etc/keepalived/keepalived.conf

# 使用 root 用户启动
/opt/nginx/keepalived/sbin/keepalived
```

---

*以上就是 Keepalived 安装的全部内容。后续文章将介绍如何搭建 Nginx + Keepalived 高可用集群。*
