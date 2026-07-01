---
title: OpenStack Train 搭建实战（2024版）
date: 2023-12-13 00:00:00
categories:
  - 云计算
tags:
  - OpenStack
  - 运维
---

# OpenStack Train 搭建实战（2024版）

## 前言

OpenStack 是一个开源的云计算管理平台项目，由 NASA 和 Rackspace 于 2010 年联合发起。它可以帮助企业搭建私有云或公有云环境，管理计算、存储、网络等基础资源。本文基于 CentOS 7 + Train 版本，手把手带你搭建一个双节点的 OpenStack 环境。

> ⚠️ OpenStack 配置文件中**不能有中文**（包括注释），否则会导致服务启动失败。

## 环境规划

| 节点 | IP 地址（内/外） | 规格 |
|------|-----------------|------|
| controller | 192.168.10.10 / 192.168.11.10 | 4核4G，硬盘100G |
| compute | 192.168.10.20 / 192.168.11.20 | 4核4G，硬盘100G+100G |

双网卡配置：NAT（内网通信）+ 仅主机（外部访问）。两台主机默认密码 `000000`。

## 基础环境配置

### 修改主机名

```bash
# controller 节点
hostnamectl set-hostname controller && bash

# compute 节点
hostnamectl set-hostname compute && bash
```

### 主机映射（双节点）

```bash
cat >>/etc/hosts <<EOF
192.168.10.10 controller
192.168.10.20 compute
EOF
```

### 关闭防火墙与 SELinux（双节点）

```bash
systemctl stop firewalld && setenforce 0
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
```

### 配置免密登录（双节点）

```bash
ssh-keygen -t rsa          # 连按三次回车
ssh-copy-id controller     # 第一次输入 yes，第二次输入密码
ssh-copy-id compute        # 同上
```

### 配置阿里云镜像仓库（双节点）

```bash
rm -rf /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache
yum -y upgrade
yum -y install epel-release
```

### 时间同步（Chrony）

推荐安装 Chrony，比 NTP 更适合虚拟化环境。

```bash
yum -y install chrony
```

**controller 节点** — 修改 `/etc/chrony.conf`：

```
server ntp3.aliyun.com iburst
allow all
local stratum 10
```

**compute 节点** — 修改 `/etc/chrony.conf`：

```
server controller iburst
allow all
local stratum 10
```

**双节点启动服务：**

```bash
systemctl restart chronyd
systemctl enable chronyd
chronyc sources    # 左侧显示 * 号表示同步成功
```

## 安装基础组件

### 部署 OpenStack Train 仓库（双节点）

```bash
yum -y install centos-release-openstack-train
```

> ❌ CentOS 7 已停止维护，部署后需要手动换源为阿里云镜像。

修改以下四个 repo 文件，将 `baseurl` 改为 `mirrors.aliyun.com`，注释掉 `mirrorlist`：

| 文件 | 说明 |
|------|------|
| `/etc/yum.repos.d/CentOS-OpenStack-train.repo` | OpenStack 主仓库 |
| `/etc/yum.repos.d/CentOS-Ceph-Nautilus.repo` | Ceph 存储仓库 |
| `/etc/yum.repos.d/CentOS-QEMU-EV.repo` | QEMU 虚拟化仓库 |
| `/etc/yum.repos.d/CentOS-NFS-Ganesha-28.repo` | NFS Ganesha 仓库 |

```bash
yum clean all && yum repolist
```

### 安装 OpenStack 客户端（双节点）

```bash
yum -y install python-openstackclient
yum -y install openstack-selinux
```

### 安装 MariaDB 数据库（controller）

```bash
yum -y install mariadb mariadb-server python2-PyMySQL
```

编辑 `/etc/my.cnf.d/openstack.cnf`：

```ini
[mysqld]
bind-address = 192.168.10.10
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

```bash
systemctl start mariadb.service
systemctl enable mariadb.service
mysql_secure_installation    # 设置 root 密码为 000000
```

### 安装 RabbitMQ 消息队列（controller）

```bash
yum -y install rabbitmq-server
systemctl start rabbitmq-server.service
systemctl enable rabbitmq-server.service

# 创建 OpenStack 用户
rabbitmqctl -n rabbit@controller add_user openstack openstack123
rabbitmqctl -n rabbit@controller set_permissions openstack ".*" ".*" ".*"
```

### 安装 Memcached 缓存（controller）

```bash
yum install memcached python-memcached -y
```

修改 `/etc/sysconfig/memcached`，将 `OPTIONS` 改为：

```
OPTIONS="-l 127.0.0.1,::1,controller"
```

```bash
systemctl start memcached.service
systemctl enable memcached.service
```

### 安装 etcd（controller，可选）

从 Stein 版本开始引入，用于分布式键值存储。

```bash
yum -y install etcd
```

编辑 `/etc/etcd/etcd.conf`，根据 `[Member]` 和 `[Clustering]` 段修改对应 IP 地址，然后：

```bash
systemctl start etcd
systemctl enable etcd
etcdctl cluster-health    # 输出 healthy 即成功
```

## Keystone 身份认证服务

### 创建数据库

```bash
mysql -uroot -p000000
```

```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone123';
```

### 安装配置

```bash
yum install openstack-keystone httpd mod_wsgi -y
```

编辑 `/etc/keystone/keystone.conf`：

```ini
[database]
connection = mysql+pymysql://keystone:keystone123@controller/keystone

[token]
provider = fernet
```

```bash
# 同步数据库
su -s /bin/sh -c "keystone-manage db_sync" keystone

# 初始化 Fernet 密钥
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# 初始化管理员用户
keystone-manage bootstrap --bootstrap-password admin \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### 配置 HTTP 服务器

```bash
vi /etc/httpd/conf/httpd.conf
# 添加：ServerName controller

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl start httpd.service
systemctl enable httpd.service
```

### 创建环境变量脚本

创建 `admin.sh`：

```bash
#!/bin/bash
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

```bash
source admin.sh
openstack endpoint list    # 验证
```

### 创建域、项目、用户和角色

```bash
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject
openstack user create --domain default --password-prompt myuser
openstack role create myrole
openstack role add --project myproject --user myuser myrole
```

## Glance 镜像服务

### 创建数据库和用户

```sql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance123';
```

```bash
source /etc/openstack/keystone/admin-openrc.sh
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description 'Openstack Image' image

# 创建 API 端点
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

### 安装配置

编辑 `/etc/glance/glance-api.conf`，配置数据库连接、Keystone 认证和存储后端：

```ini
[database]
connection = mysql+pymysql://glance:glance123@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api.service
systemctl start openstack-glance-api.service
```

## Placement 服务

### 创建数据库和用户

```sql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'placement123';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement123';
```

```bash
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement --description 'placement API' placement

openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

### 安装配置

```bash
yum -y install openstack-placement-api
```

编辑 `/etc/placement/placement.conf`，配置数据库和 Keystone 认证后：

```bash
su -s /bin/sh -c "placement-manage db sync" placement
placement-status upgrade check    # 验证
```

## Nova 计算服务

### 创建数据库和用户

```sql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
```

```bash
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "Openstack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

### controller 节点配置

```bash
yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y
```

编辑 `/etc/nova/nova.conf` 关键配置：

```ini
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack123@controller:5672/
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
my_ip = 192.168.10.10

[api]
auth_strategy = keystone

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[placement]
auth_url = http://controller:5000/v3
username = placement
password = placement
```

```bash
# 同步数据库
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova

# 启动服务
systemctl enable openstack-nova-api --now
systemctl enable openstack-nova-scheduler --now
systemctl enable openstack-nova-conductor --now
systemctl enable openstack-nova-novncproxy --now
```

### compute 节点配置

```bash
yum -y install openstack-nova-compute
```

编辑 `/etc/nova/nova.conf`，主要修改 `my_ip = 192.168.10.20`，VNC 监听改为 `0.0.0.0`，并添加 `novncproxy_base_url`。

```bash
# 检查硬件加速支持
egrep -c '(vmx|svm)' /proc/cpuinfo
# 返回 0 则需设置 virt_type = qemu

systemctl enable libvirtd --now
systemctl enable openstack-nova-compute --now
```

**controller 节点验证：**

```bash
openstack compute service list --service nova-compute
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

## Neutron 网络服务

### 创建数据库和用户

```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutron123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron123';
```

```bash
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "Openstack Networking" network

openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

### controller 节点配置

```bash
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```

核心配置文件包括：

| 配置文件 | 关键配置 |
|----------|---------|
| `/etc/neutron/neutron.conf` | 数据库、RabbitMQ、Keystone、Nova 联动 |
| `/etc/neutron/plugins/ml2/ml2_conf.ini` | ML2 插件：flat/vlan 网络类型 |
| `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` | Linux Bridge 映射物理网卡 |
| `/etc/neutron/dhcp_agent.ini` | DHCP 驱动配置 |
| `/etc/neutron/metadata_agent.ini` | 元数据代理配置 |

> ⚠️ 需要配置内核参数 `net.bridge.bridge-nf-call-iptables = 1`

```bash
# 创建软连接并同步数据库
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

systemctl restart neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
```

### compute 节点配置

```bash
yum install openstack-neutron-linuxbridge ebtables ipset -y
```

配置 `neutron.conf`、`linuxbridge_agent.ini` 和 `nova.conf` 中的 neutron 段后：

```bash
systemctl restart openstack-nova-compute
systemctl enable neutron-linuxbridge-agent --now
```

## Dashboard 仪表盘

```bash
yum install openstack-dashboard -y
```

编辑 `/etc/openstack-dashboard/local_settings`，关键配置：

```python
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*']
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
OPENSTACK_KEYSTONE_DEFAULT_ROLE = 'user'
TIME_ZONE = "Asia/Shanghai"
WEBROOT = '/dashboard'
```

```bash
systemctl restart httpd memcached
# 访问 http://192.168.10.10/dashboard
# 域: default | 用户名: admin | 密码: admin
```

## Cinder 块存储服务

### controller 节点

```sql
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinder123';
```

```bash
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

# 创建 v2/v3 的 public/internal/admin 端点（端口 8776）
yum install openstack-cinder -y
```

编辑 `/etc/cinder/cinder.conf` 配置数据库、RabbitMQ 和 Keystone 后：

```bash
sudo su -s /bin/sh -c "cinder-manage db sync" cinder
systemctl restart openstack-nova-api
systemctl start openstack-cinder-api openstack-cinder-scheduler
systemctl enable openstack-cinder-api openstack-cinder-scheduler
```

### compute 节点

```bash
yum install lvm2 device-mapper-persistent-data -y
systemctl enable lvm2-lvmetad.service && systemctl start lvm2-lvmetad.service

# 创建物理卷和卷组
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

编辑 `/etc/lvm/lvm.conf` 添加过滤器：

```
filter = [ "a/sda/", "a/sdb/", "r/.*/"]
```

```bash
yum install openstack-cinder targetcli python-keystone -y
```

配置 `/etc/cinder/cinder.conf`，添加 `[lvm]` 段指定卷驱动后：

```bash
systemctl start openstack-cinder-volume.service target.service
systemctl enable openstack-cinder-volume.service target.service
```

## Swift 对象存储服务

### 创建服务凭据

```bash
openstack user create --domain default --password-prompt swift
openstack role add --project service --user swift admin
openstack service create --name swift --description "OpenStack Object Storage" object-store

# 创建 public/internal/admin 端点（端口 8080）
```

### controller 节点

```bash
yum install openstack-swift-proxy python-swiftclient python-keystoneclient \
  python-keystonemiddleware memcached -y

curl -o /etc/swift/proxy-server.conf \
  https://opendev.org/openstack/swift/raw/branch/master/etc/proxy-server.conf-sample
```

编辑 `proxy-server.conf` 配置 Keystone 认证和缓存后：

```bash
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service
```

### compute 节点

```bash
yum install xfsprogs rsync -y

# 格式化并挂载磁盘
mkfs.xfs /dev/sdb && mkfs.xfs /dev/sdc
mkdir -p /srv/node/sdb /srv/node/sdc
# 编辑 /etc/fstab 添加挂载项后 mount

# 配置 rsync 并启动
systemctl enable rsyncd.service && systemctl start rsyncd.service

# 安装 Swift 存储组件
yum install openstack-swift-account openstack-swift-container openstack-swift-object -y
```

### 创建和分发 Ring（controller）

```bash
cd /etc/swift/

# Account Ring
swift-ring-builder account.builder create 10 3 1
swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6202 --device sdb --weight 100
swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6202 --device sdc --weight 100
swift-ring-builder account.builder rebalance

# Container Ring
swift-ring-builder container.builder create 10 3 1
swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6201 --device sdb --weight 100
swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6201 --device sdc --weight 100
swift-ring-builder container.builder rebalance

# Object Ring
swift-ring-builder object.builder create 10 3 1
swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6200 --device sdb --weight 100
swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.10.20 --port 6200 --device sdc --weight 100
swift-ring-builder object.builder rebalance

# 分发到 compute 节点
scp swift.conf compute:/etc/swift/swift.conf
chown -R root:swift /etc/swift
```

## 常见问题与踩坑记录

❌ **CentOS 7 源失效**：部署 OpenStack 仓库后必须手动换源，否则 yum 安装会报 404。

❌ **配置文件中混入中文**：OpenStack 所有服务的配置文件（包括注释）都不能有中文，否则服务启动直接报错。

❌ **Nova 计算节点无法发现**：需要在 controller 节点执行 `nova-manage cell_v2 discover_hosts`，或在 `nova.conf` 中设置 `discover_hosts_in_cells_interval = 300` 自动发现。

❌ **Neutron 网络不通**：检查内核参数 `net.bridge.bridge-nf-call-iptables` 是否设置为 1，以及 Linux Bridge 的物理网卡映射是否正确。

❌ **Swift Ring 不平衡**：每次添加或删除存储节点后都需要执行 `rebalance`，否则数据分布不均匀。

❌ **Dashboard 无法访问**：确认 `WEBROOT = '/dashboard'`，访问地址为 `http://controller/dashboard`，而不是根路径。

## 服务安装顺序总结

整个 OpenStack 的搭建按照以下顺序进行：基础环境 → Keystone（认证）→ Glance（镜像）→ Placement（资源调度）→ Nova（计算）→ Neutron（网络）→ Dashboard（仪表盘）→ Cinder（块存储）→ Swift（对象存储）。每一步都需要先创建数据库和用户，再安装配置组件，最后启动服务并验证。

搭建完成后，可以通过 Dashboard 或 CLI 命令创建实例、分配浮动 IP、挂载云硬盘等操作，真正体验 OpenStack 的云计算能力。
