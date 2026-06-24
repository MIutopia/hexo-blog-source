---
title: OpenStack搭建（2024版）
date: 2023-12-13 00:00:00
categories:
  - 教程
tags:
  - OpenStack
  - 运维
---

# [](#主机规划)主机规划

主机名(hostname)
IP地址(双网卡(NAT,仅主机)）
规格

controller(控制节点)
内网:192.168.10.10  外网:192.168.11.10
4核4g，硬盘100g

compute(计算节点)
内网:192.168.10.20  外网:192.168.11.20
4核4g，硬盘100g+100g

*注：为了不影响后续操作，两台主机默认密码都是“000000”*

​        *OpenStack的配置文件中不能有中文(注释也不行)*

## [](#修改主机名)修改主机名

**controller节点**

1
2
hostnamectl set-hostname controller
bash          

**compute节点**

1
2
hostnamectl set-hostname compute
bash

## [](#主机映射)主机映射

**双节点(两个节点都需要执行)**

1
2
3
4
cat &gt;&gt;/etc/hosts &lt;&lt;EOF
&gt;192.168.10.10 controller
&gt;192.168.10.20 compute
&gt;EOF

## [](#关闭防火墙配置免密登录)关闭防火墙配置免密登录

**双节点**

1
2
3
4
5
6
7
8
systemctl stop firewalld &amp;&amp; setenforce 0  #暂时关闭防火墙和seliunx(重启后会自启动)
systemctl disable firewalld               #永久关闭防火墙(关闭开机自启动，需要手动开启)
 
sed -i &#x27;s/enforcing/disabled/&#x27; /etc/selinux/config #使用命令修改文件永久关闭seliunx

ssh-keygen -t rsa         #按三次回车
ssh-copy-id controller    #第一次输入yes,第二次输入主机密码
ssh-copy-id compute       #第一次输入yes,第二次输入主机密码

## [](#配置阿里云镜像仓库)配置阿里云镜像仓库

1
2
3
4
5
6
7
8
rm-rf /etc/yum.repo.d/*                          #清除本地镜像仓库

#配置阿里云yum源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

yum makecache                                    #重建yum源缓存
yum -y upgrade                                   #升级内核
yum -y install epel-release                      #配置EPEL仓库

# [](#时间同步)时间同步

**双节点**

**安装 Chrony(推荐) 或 NTP 来实现（选择其中一个安装即可(Chrony更适合虚拟化))**

## [](#Chrony)Chrony

**双节点**

1
yum -y install chrony   #安装Chrony服务

**controller节点**

1
2
3
4
5
6
7
8
vi /etc/chrony.conf     #修改配置文件
server 0.centos.pool.ntp.org iburst 改为 server ntp3.aliyun.com iburst
server 1.centos.pool.ntp.org iburst #删除
server 2.centos.pool.ntp.org iburst #删除
server 3.centos.pool.ntp.org iburst #删除

#allow 192.168.0.0/16 改为 allow all并去掉注释
#local stratum 10   #去掉注释

**compute**节点

1
2
3
4
5
6
7
8
vi /etc/chrony.conf     #修改配置文件
server 0.centos.pool.ntp.org iburst 改为 server controller iburst
server 1.centos.pool.ntp.org iburst #删除
server 2.centos.pool.ntp.org iburst #删除
server 3.centos.pool.ntp.org iburst #删除

#allow 192.168.0.0/16 改为 allow all并去掉注释
#local stratum 10   #去掉注释

**修改后的配置文件：**

![](https://www.z4a.net/images/2023/12/15/2023-12-15-110620.png)

1
2
3
4
5
6
7
8
9
systemctl restart chronyd   #重启chrony服务
systemctl enable chronyd    #设置为开机自启动
systemctl status chronyd    #查看状态
chronyc sources             #验证
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 203.107.6.88                  2   6    37    47    -63us[ -319us] +/-   41ms
#左下角显示*号说明配置成功

## [](#NTP)NTP

1
2
3
4
5
6
7
8
9
10
yum -y install ntp       #安装ntp服务

vi /etc/ntp.conf         #编辑配置文件
server &lt;NTP服务器IP&gt;      #添加的内容(与chrony服务器的方法相同)

systemctl restart ntpd   #重启
systemctl status ntpd    #查看状态
systemctl enable ntpd    #设置为开机自启动

ntpdate -q &lt;主机IP&gt;       #输入主机IP验证两台主机时间是否同步成功

*注：公共的ntp服务器*

- *[cn.pool.ntp.org](http://cn.pool.ntp.org/)（中国公共NTP服务器）*

- *[pool.ntp.org](http://pool.ntp.org/)（全球公共NTP服务器）*

# [](#安装软件包、数据库、消息队列、缓存)安装软件包、数据库、消息队列、缓存

## [](#安装OpenStack-Train仓库)安装OpenStack-Train仓库

**双节点**

1
yum -y install centos-release-openstack-train    #部署OpenStack仓库

*由于centos官方已经不再支持更新centos7，所以在部署openstack仓库后需要换源*

1
2
3
4
5
6
7
8
9
10
vi /etc/yum.repos.d/CentOS-OpenStack-train.repo
#修改后的内容
[centos-openstack-train]
name=CentOS-7 - OpenStack train
baseurl=http://mirrors.aliyun.com/$contentdir/$releasever/cloud/$basearch/openstack-train/  #去掉注释，改为mirrors.aliyun.com
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&amp;arch=$basearch&amp;repo=cloud-openstack-train  #添加 # 号
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

1
2
3
4
5
6
7
8
9
vi /etc/yum.repos.d/CentOS-Ceph-Nautilus.repo
#修改后的内容(步骤同上)
[centos-ceph-nautilus]
name=CentOS-$releasever - Ceph Nautilus
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&amp;arch=$basearch&amp;repo=storage-ceph-nautilus 
baseurl=http://mirrors.aliyun.com/$contentdir/$releasever/storage/$basearch/ceph-nautilus/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

1
2
3
4
5
6
7
8
9
vi /etc/yum.repos.d/CentOS-QEMU-EV.repo 
#修改后的内容(步骤同上)
[centos-qemu-ev]
name=CentOS-$releasever - QEMU EV
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&amp;arch=$basearch&amp;repo=virt-kvm-common  
baseurl=http://mirrors.aliyun.com/$contentdir/$releasever/virt/$basearch/kvm-common/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Virtualization

1
2
3
4
5
6
7
8
9
10
11
12
vi /etc/yum.repos.d/CentOS-NFS-Ganesha-28.repo
#修改后的内容(步骤同上)
[centos-nfs-ganesha28]
name=CentOS-$releasever - NFS Ganesha 2.8
#mirrorlist=http://mirrorlist.centos.org?arch=$basearch&amp;release=$releasever&amp;repo=storage-nfs-ganesha-28
baseurl=https://mirrors.aliyun.com/$contentdir/$releasever/storage/$basearch/nfs-ganesha-28/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

#修改完成后清除本地yum索引缓存，然后重建索引缓存
yum clean all &amp;&amp; yum repolist

## [](#安装OpenStack客户端)安装OpenStack客户端

**双节点**

1
2
yum -y install python-openstackclient            #安装OpenStack客户端
yum -y install openstack-selinux                 #服务策略

## [](#安装SQL数据库)安装SQL数据库

**controller节点**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
yum -y install mariadb mariadb-server python2-PyMySQL      #安装组件

vi /etc/my.cnf.d/openstack.cnf                             #编辑配置文件
#添加以下内容
[mysqld]
bind-address = 192.168.10.10                               #改为自己的主机IP

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

systemctl start mariadb.service                            #启动数据库
systemctl enable mariadb.service                           #设置为为开机自启动
mysql_secure_installation                                  #初始化
#第一次按回车
#第二次按Y设置初始密码000000
#第三次按Y
#第四次按n
#第五次按Y
#第六次按Y

## [](#消息队列)消息队列

**controller节点**

1
2
3
4
5
6
7
8
9
yum -y install rabbitmq-server                         #安装软件包
systemctl start rabbitmq-server.service                #启动服务
systemctl enable rabbitmq-server.service               #设置为开机自启动

#创建用户,密码&#x27;openstack123&#x27;
rabbitmqctl -n rabbit@controller add_user openstack openstack123

#添加权限允许用户进行配置、写入和读取访问
rabbitmqctl  -n rabbit@controller set_permissions openstack &quot;.*&quot; &quot;.*&quot; &quot;.*&quot;

## [](#安装缓存)安装缓存

**controller节点**

1
2
3
4
5
6
7
yum install memcached python-memcached -y         #安装软件包

vi /etc/sysconfig/memcached                       #编辑配置文件
OPTIONS=&quot;-l 127.0.0.1,::1,controller&quot;               #更改现有行&quot;OPTIONS=&quot;-l 127.0.0.1,::1&quot;

systemctl start memcached.service                #启动服务
systemctl enable memcached.service                #设置为开机自启动

## [](#安装etcd)安装etcd

*从S版开始有，详情请参考[官网](https://docs.openstack.org/2023.2/)(无特殊需求可以跳过此步骤)*

**controller节点**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
yum -y install etcd             #安装软件包

vi /etc/etcd/etcd.conf          #编辑配置文件(根据对应的选项进行修改)
#[Member]
ETCD_DATA_DIR=&quot;/var/lib/etcd/default.etcd&quot;
ETCD_LISTEN_PEER_URLS=&quot;http://&lt;controller节点IP&gt;:2380&quot;
ETCD_LISTEN_CLIENT_URLS=&quot;http://&lt;controller节点IP&gt;:2379,http://127.0.0.1:2379&quot;
ETCD_NAME=&quot;controller&quot;
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS=&quot;http://&lt;controller节点IP&gt;:2380&quot;
ETCD_ADVERTISE_CLIENT_URLS=&quot;http://&lt;controller节点IP&gt;:2379&quot;
ETCD_INITIAL_CLUSTER=&quot;controller=http://&lt;controller节点IP&gt;&quot;
ETCD_INITIAL_CLUSTER_TOKEN=&quot;etcd-cluster-01&quot;
ETCD_INITIAL_CLUSTER_STATE=&quot;new&quot;

systemctl start etcd            #启动服务
systemctl enable etcd           #设置为开机自启动
etcdctl cluster-health          #验证(输出以下结果表示配置成功)
member 4a1ad6ae1797e6ea is healthy: got healthy result from http://&lt;controller节点IP&gt;:2379

# [](#Keystone服务)Keystone服务

## [](#controller节点)controller节点

### [](#先决条件)先决条件

**创建数据库**

1
2
3
4
5
6
7
8
9
10
11
#进入数据库
mysql -uroot -p000000

#创建数据库keystone
MariaDB [(none)]&gt; CREATE DATABASE keystone;

#赋予权限(keystone123为密码)
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON keystone.* TO &#x27;keystone&#x27;@&#x27;localhost&#x27; \
IDENTIFIED BY &#x27;keystone123&#x27;;
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON keystone.* TO &#x27;keystone&#x27;@&#x27;%&#x27; \
IDENTIFIED BY &#x27;keystone123&#x27;;

*sql语句一定要有  ;  号结尾*

### [](#安装配置组件)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
#安装软件包
yum install openstack-keystone httpd mod_wsgi -y

#编辑配置文件
vi /etc/keystone/keystone.conf

#找到需要修改的部分添加内容,查找方法(:/\查找的内容)
[database]
# ...
connection = mysql+pymysql://keystone:keystone123@controller/keystone   #直接添加
[token]
# ...
provider = fernet   #直接添加

#同步数据库
su -s /bin/sh -c &quot;keystone-manage db_sync&quot; keystone

#初始化 Fernet 密钥存储库
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

#初始化 Keystone 系统，并创建第一个管理员用户
keystone-manage  bootstrap --bootstrap-password admin --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne 

### [](#配置HTTP服务器)配置HTTP服务器

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
#编辑配置文件
vi /etc/httpd/conf/httpd.conf
ServerName controller   #添加这一行

#创建指向文件的链接
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

#启动服务
systemctl start httpd.service
systemctl enable httpd.service

#创建环境变量脚本添加以下内容
vi admin.sh
#!/bin/bash
export OS_USERNAME=admin  #设置管理员用户的用户名为&quot;admin&quot;
export OS_PASSWORD=admin  #设置管理员用户的密码为&quot;admin&quot;
export OS_PROJECT_NAME=admin  #设置管理员用户所在的项目为&quot;admin&quot;
export OS_USER_DOMAIN_NAME=Default  #设置管理员用户所在域的名称为&quot;Default&quot;
export OS_PROJECT_DOMAIN_NAME=Default  #设置管理员用户所在项目的域的名称为&quot;Default&quot;
export OS_AUTH_URL=http://controller:5000/v3  #设置OpenStack的认证URL为&quot;http://controller:5000/v3&quot;
export OS_IDENTITY_API_VERSION=3  #设置OpenStack的身份认证API版本为&quot;3&quot;
export OS_IMAGE_API_VERSION=2

#启动脚本
source admin.sh

#验证
openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                        |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+
| 5c4d1897030749209c3a6f22975ec618 | RegionOne | keystone     | identity     | True    | admin     | http://controller:5000/v3/ |
| 67e3604466c14e698b0efd7f94e66288 | RegionOne | keystone     | identity     | True    | public    | http://controller:5000/v3/ |
| 70ac89874ddf4a238699d73d05d8bb15 | RegionOne | keystone     | identity     | True    | internal  | http://controller:5000/v3/ |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+

### [](#创建域、项目、用户和角色)创建域、项目、用户和角色

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
#创建一个名为&quot;Example&quot;的域
openstack domain create --description &quot;An Example Domain&quot; example

#在名为 default 的域下创建名为 service 的项目
openstack project create --domain default --description &quot;Service Project&quot; service

#在 default 域下创建项目 myproject (创建其他用户时,不需要重复该步骤)
openstack project create --domain default --description &quot;Demo Project&quot; myproject

#创建用户 myuser (需要设置密码,统一为myuser)
openstack user create --domain default --password-prompt myuser

#创建角色 myrole
openstack role create myrole

#将角色添加到项目和用户 myrolemyprojectmyuser
openstack role add --project myproject --user myuser myrole

*注：重复以上步骤可创建多个项目和用户*

### [](#验证)验证

1
2
3
4
5
6
7
8
9
10
11
12
#取消设置临时变量和环境变量 OS_AUTH_URLOS_PASSWORD
unset OS_AUTH_URL OS_PASSWORD

#需要输入admin密码(需要输入两次)
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue

#需要输入myuser密码(需要输入两次)
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue

### [](#创建-OpenStack-客户端环境脚本)创建 OpenStack 客户端环境脚本

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
#创建编辑脚本 admin-openrc
mkdir -p /etc/openstack/keystone
vi /etc/openstack/keystone/admin-openrc.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

#使用脚本
source /etc/openstack/keystone/admin-openrc.sh
openstack token issue

# [](#Glance服务)Glance服务

## [](#controller节点-1)controller节点

### [](#先决条件-1)先决条件

**创建数据库**

1
2
3
4
5
6
7
8
9
#进入数据库
mysql -uroot -p000000

#创建数据库glance
MariaDB [(none)]&gt; CREATE DATABASE glance;

#授予对 glance 库的适当访问权限
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON glance.* TO &#x27;glance&#x27;@&#x27;localhost&#x27; IDENTIFIED BY &#x27;glance123&#x27;;
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON glance.* TO &#x27;glance&#x27;@&#x27;%&#x27; IDENTIFIED BY &#x27;glance123&#x27;;

**创建服务凭据**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
#获取要访问的凭据
source /etc/openstack/keystone/admin-openrc.sh 

#在名为 default 的域下创建名为 glance 的用户,需要设置密码,密码统一为glance
openstack user create --domain default --password-prompt glance 

#将 glance 用户添加到 service 项目的 admin 角色中
openstack role add --project service --user glance admin

#创建一个名为glance的镜像服务
openstack service create --name glance --description &#x27;Openstack Image&#x27; image

#验证(查看服务id列表)
openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 97ebf2fe63a2402fbfaa36237a2101e1 | glance   | image    |
| d14e27b4de294015912bcdd58d1fb449 | keystone | identity |
+----------------------------------+----------+----------+ 

**创建API端点**

1
2
3
4
5
6
7
8
# 为名为 glance 的镜像服务创建一个公共的终端，并指定 URL 地址为 http://controller:9292
openstack endpoint create --region RegionOne image public http://controller:9292

#为 glance 的镜像服务创建一个内部的终端，并指定 URL 地址为 http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292 

# 为 glance 的镜像服务创建一个管理员的终端，并指定 URL 地址为 http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292 

### [](#安装配置组件-1)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
#安装软件包
yum -y install openstack-glance

#编辑配置文件(与配置keystone方法一样)
vi /etc/glance/glance-api.conf
[database]
# ...
connection = mysql+pymysql://glance:glance123@controller/glance #连接数据库

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000  #Keystone 的 HTTP 认证端点地址，用于与 Glance 进行认证和授权等操作
auth_url = http://controller:5000  #Keystone 的认证端点地址
memcached_servers = controller:11211  # Memcached 缓存服务器的地址和端口信息，用于缓存 Glance 服务的认证令牌和会话信息
auth_type = password  #认证类型，一般为 password
project_domain_name = Default  #项目域名，默认为 Default
user_domain_name = Default  #用户域名，默认为 Default
Project_name = service  #项目名称，用于指定访问 Keystone API 时使用的项目
username = glance  #用户名，用于指定与 Keystone 进行身份验证时使用的用户名
password = glance  #密码，用于与 Keystone 进行身份验证时使用的密码

[paste_deploy]
# ...
flavor = keystone  #flavor 参数指定了 Glance 服务所使用的应用程序类型，取值为 keystone。这表示 Glance 服务是以 Keystone 身份认证服务为基础构建的，因此需要与之进行交互，以实现认证和授权等功能

[glance_store]
# ...
stores = file,http  # Glance 所使用的存储后端类型，可以指定多个，用逗号进行分隔。常见的存储后端类型包括 file、http、https、rbd、swift、cinder 等
default_store = file  #默认的存储后端类型，当一个新的镜像被创建时，将使用该存储后端来保存镜像数据
filesystem_store_datadir = /var/lib/glance/images/  #文件系统存储类型的数据目录，用于指定文件系统存储后端保存镜像数据的路径

#同步数据库
su -s /bin/sh -c &quot;glance-manage db_sync&quot; glance

#启动服务
systemctl enable openstack-glance-api.service
systemctl start openstack-glance-api.service

### [](#验证-1)验证

验证方式可参考官方文档：

[OpenStack Docs：验证操作](https://docs.openstack.org/glance/train/install/verify.html)

# [](#Placement服务)Placement服务

## [](#controller节点-2)controller节点

### [](#先决条件-2)先决条件

**创建数据库**

1
2
3
4
5
6
7
8
9
#进入数据库
mysql -uroot -p000000

#创建palacement库
MariaDB [(none)]&gt; CREATE DATABASE placement;

#授权(placement123为密码)
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON placement.* TO &#x27;placement&#x27;@&#x27;localhost&#x27; IDENTIFIED BY &#x27;placement123&#x27;;
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON placement.* TO &#x27;placement&#x27;@&#x27;%&#x27; IDENTIFIED BY &#x27;placement123&#x27;;

**配置用户和端点**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
#获取凭据(admin)
source /etc/openstack/keystone/admin-openrc.sh 

#创建用户(密码为placement,需要输入两次)
openstack user create --domain default --password-prompt placement

#将placement赋予admin权限
openstack role add --project service --user placement admin

#创建Placement API项目
openstack service create --name placement --description &#x27;placement API&#x27; placement

#创建外部端点
openstack endpoint create --region RegionOne placement public http://controller:8778

#创建内部端点
openstack endpoint create --region RegionOne placement internal http://controller:8778

#创建管理员端点
openstack endpoint create --region RegionOne placement admin http://controller:8778 

### [](#安装和配置组件)安装和配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
#安装软件包
yum -y install openstack-placement-api

#编辑配置文件(和之前的方法一致)
vi /etc/placement/placement.conf
[placement_database]
# ...
connection = mysql+pymysql://placement:placement123@controller/placement

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement

#同步数据库 placement(忽略此输出中的任何弃用消息)
su -s /bin/sh -c &quot;placement-manage db sync&quot; placement

### [](#验证-2)验证

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
#获取凭证(admin)
source /etc/openstack/keystone/admin-openrc.sh

#查看状态
placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+

# [](#Nova服务)Nova服务

## [](#controller节点-3)controller节点

### [](#先决条件-3)先决条件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
#进入数据库
mysql -uroot -p000000

#创建三个数据库
create database nova_api;
create database nova;
create database nova_cell0;

#对以创建的三个数据库赋予权限
grant all privileges on nova_api.* to &#x27;nova&#x27;@&#x27;%&#x27; identified by &#x27;nova123&#x27;;
grant all privileges on nova.* to &#x27;nova&#x27;@&#x27;%&#x27; identified by &#x27;nova123&#x27;;
grant all privileges on nova_cell0.* to &#x27;nova&#x27;@&#x27;%&#x27; identified by &#x27;nova123&#x27;;

#创建 nova 用户(密码统一为 nova)
openstack user create --domain default --password-prompt nova

#将 nova 用户添加到 service 项目中并分配 admin 角色
openstack role add --project service --user nova admin

#创建一个名为 nova 的计算服务
openstack service create --name nova --description &quot;Openstack Compute&quot; compute

#创建一个名为 compute 的计算服务公共 API 端点
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1

#创建一个名为 compute 的计算服务内部 API 端点
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1

#创建一个名为 compute 的计算服务管理员 API 端点
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1

### [](#安装和配置组件-1)安装和配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
#安装软件包
yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y

#编辑配置文件 /etc/nova/nova.conf
vi /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata  #指定启用 Nova 计算服务的 API 类型为 osapi_compute 和 metadata，分别对应计算服务和元数据服务接口
transport_url = rabbit://openstack:openstack123@controller:5672/  # 指定 Nova 计算服务使用的消息队列传输协议的 URL 地址，其中 openstack 和 openstack123 分别为 RabbitMQ 服务所需要的用户名和密码，controller 表示 RabbitMQ 所在的节点地址，5672 为 RabbitMQ 的默认端口号
use_neutron = true  #指定 Nova 计算服务使用 Neutron 网络服务连接虚拟机实例，默认为 false
firewall_driver = nova.virt.firewall.NoopFirewallDriver #指定 Nova 计算服务使用的防火墙驱动程序，该行配置表示使用 NoopFirewallDriver 空白防火墙驱动程序，即不对虚拟机实例进行防火墙设置
my_ip = 192.168.10.10

[api_database]
connection = mysql+pymysql://nova:nova123@controller/nova_api #连接数据库

[database]
connection = mysql+pymysql://nova:nova123@controller/nova #连接数据库

[api]
auth_strategy = keystone #指定 OpenStack API 服务使用 Keystone 身份认证服务进行身份验证和授权管理

[keystone_authtoken]
www_authenticate_url = http://controller:5000/ #指定 Keystone 身份认证服务的 HTTP 认证 URL 地址，用于向 Nova 计算服务提供身份认证和授权服务
auth_url = http://controller:5000/ # 指定 Keystone 身份认证服务的默认 URL 地址，用于向 Nova 计算服务提供身份认证和授权服务
memcached_servers = controller:11211 #指定 Keystone 使用的 memcached 服务器地址和端口号，用于缓存身份认证令牌信息，提高身份认证效。
auth_type = password #指定身份认证类型为密码认证，即使用用户名和密码进行身份认证
project_domain_name = Default #指定 Keystone 身份认证服务所使用的项目域名，默认为 Default
user_domain_name = Default #指定 Keystone 身份认证服务所使用的项目域名，默认为 Default
project_name = service  # 指定 Keystone 身份认证服务所用的项目名称，用于与 Nova 计算服务进行交互
username = nova #指定 Keystone 身份认证服务所用的用户名，用于与 Nova 计算服务进行交互
password = nova #指定 Keystone 身份认证服务所用的密码，用于与 Nova 计算服务进行交互

[vnc]
enabled = true #指定 VNC 控制台功能是否启用，该行配置表示启用 VNC 控制台功能
server_listen = $my_ip #指定 VNC 服务所监听的 IP 地址，该行配置中的 $my_ip 表示使用本地主机的 IP 地址进行监听。需要注意的是，如果计算节点的网络配置存在多个网卡或网络接口，需要指定具体的 IP 地址进行监听，否则可能导致 VNC 服务无法正常工作
server_proxyclient_address = $my_ip #指定 VNC 服务代理客户端所使用的 IP 地址，该行配置中的 $my_ip 表示使用本地主机的 IP 地址作为代理客户端 IP 地址。代理客户端通常是指浏览器等客户端应用程序，用于连接 VNC 服务，如果不生效，可能会导致 VNC 服务无法正常工作

[glance]
api_servers = http://controller:9292  #指定 Glance 镜像服务的 API 服务器地址和端口号，用于向 Nova 计算服务提供镜像服务

[oslo_concurrency]
lock_path = /var/lab/nova/tmp  #指定锁文件的存放路径，用于实现并发控制和资源管理

[placement]
region_name = RegionOne #指定 OpenStack 环境中的区域名称，默认为 RegionOne
project_domain_name = Default #指定项目领域的名称，默认为 Default
project_name = service #指定所属项目的名称
auth_type = password #指定认证方式为密码认证
user_domain_name = Default #指定用户领域的名称，默认为 Default
auth_url = http://controller:5000/v3 #指定认证服务的地址和版本号，用于向 Keystone 身份认证服务进行认证
username = placement #指定用于认证的用户名
password = placement #指定用于认证的密码

### [](#同步数据库)同步数据库

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
#填充数据库 nova-api
su -s /bin/sh -c &quot;nova-manage api_db sync&quot; nova

#注册数据库 cell0
su -s /bin/sh -c &quot;nova-manage cell_v2 map_cell0&quot; nova

#创建单元格 cell1
su -s /bin/sh -c &quot;nova-manage cell_v2 create_cell --name=cell1 --verbose&quot; nova

#填充nova数据库
su -s /bin/sh -c &quot;nova-manage db sync&quot; nova
#验证是否正确注册
su -s /bin/sh -c &quot;nova-manage cell_v2 list_cells&quot; nova
#启动服务(配置为在系统启动时启动)
systemctl enable openstack-nova-api --now
systemctl enable openstack-nova-scheduler --now
systemctl enable openstack-nova-conductor --now
systemctl enable openstack-nova-novncproxy --now

## [](#compute节点)compute节点

### [](#安装配置组件-2)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
#安装软件包
yum -y install openstack-nova-compute

#编辑配置文件 /etc/nova/nova.conf
vi /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata   #表示启用的API服务，包括计算服务（osapi_compute）和元数据服务（metadata）
transport_url = rabbit://openstack:openstack123@controller  #表示消息队列RabbitMQ的连接地址，用户名为openstack，密码为openstack123，连接到服务器为controller
my_ip = 192.168.10.20  #为当前节点的IP地址
use_neutron = true  #表示是否使用Neutron网络服务
firewall_driver = nova.virt.firewall.NoopFirewallDriver  #表示所使用的防火墙驱动。这里设置nova.virt.firewall.NoopFirewallDriver，#即不启用防火墙功能

[api]
auth_strategy = keystone  #表示认证策略，这里设为keystone

[keystone_authtoken]
www_authenticate_url = http://controller:5000/   #Keystone服务的认证地址
auth_url = http://controller:5000/  #Keystone服务的管理地址
memcached_servers = controller:11211 #为Memcached服务地址，用于缓存Keystone的token
auth_type = password  #表示认证类型，这里为password
project_domain_name = Default  #项目域名
user_domain_name = Default  #用户域名
project_name = service  #项目
username = nova  #用户名
password = nova  #密码

[vnc]
enabled = true  #enabled表示是否启用VNC
server_listen = 0.0.0.0  #表示监听地址
server_proxyclient_address = $my_ip  #表示代理客户端地址，这里为$my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html  #为使用novnc协议时的URL地址

[glance]
api_servers = http://controller:9292  #表示Glance镜像服务的API地址

[oslo_concurrency]
lock_path = /var/lab/nova/tmp  #表示锁文件的存储路径

[placement]
region_name = RegionOne #指定 OpenStack 环境中的区域名称，默认为 RegionOne
project_domain_name = Default #指定项目领域的名称，默认为 Default
project_name = service #指定所属项目的名称
auth_type = password #指定认证方式为密码认证
user_domain_name = Default #指定用户领域的名称，默认为 Default
auth_url = http://controller:5000/v3 #指定认证服务的地址和版本号，用于向 Keystone 身份认证服务进行认证
username = placement #指定用于认证的用户名
password = placement #指定用于认证的密码

1
2
3
4
5
6
#确定计算节点是否支持硬件加速
$ egrep -c &#x27;(vmx|svm)&#x27; /proc/cpuinfo

#编辑配置文件 /etc/nova/nova.conf
[libvirt]
virt_type = qemu

### [](#启动服务并验证)启动服务并验证

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
#启动服务
systemctl enable libvirtd --now
systemctl enable openstack-nova-compute --now

#在controller节点上执行
openstack compute service list --service nova-compute
+----+--------------+---------+------+---------+-------+----------------------------+
| ID | Binary       | Host    | Zone | Status  | State | Updated At                 |
+----+--------------+---------+------+---------+-------+----------------------------+
| 10 | nova-compute | compute | nova | enabled | up    | 2024-03-11T06:08:53.000000 |
+----+--------------+---------+------+---------+-------+----------------------------+
su -s /bin/sh -c &quot;nova-manage cell_v2 discover_hosts --verbose&quot; nova

Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell &#x27;cell1&#x27;: c3bce124-3708-41eb-8bc0-d00ad8c97174
Checking host mapping for compute host &#x27;compute&#x27;: 485f806a-29ce-4440-ac6b-93d6173421da
Creating host mapping for compute host &#x27;compute&#x27;: 485f806a-29ce-4440-ac6b-93d6173421da
Found 1 unmapped computes in cell: c3bce124-3708-41eb-8bc0-d0

1
2
3
4
#编辑配置文件 /etc/nova/nova.conf
vi /etc/nova/nova.conf
[scheduler]
discover_hosts_in_cells_interval = 300  #表示在Nova单元格（cell）架构中发现主机并将其映射到适当的单元格的频率

# [](#Neutron服务)Neutron服务

## [](#controller节点-4)controller节点

### [](#先决条件-4)先决条件

**创建数据库**

1
2
3
4
5
6
7
8
9
10
11
#进入数据库
mysql -uroot -p000000

#创建数据库 neutron
MariaDB [(none)]&gt; CREATE DATABASE neutron;

#赋予权限
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON neutron.* TO &#x27;neutron&#x27;@&#x27;localhost&#x27; \
  IDENTIFIED BY &#x27;neutron123&#x27;;
MariaDB [(none)]&gt; GRANT ALL PRIVILEGES ON neutron.* TO &#x27;neutron&#x27;@&#x27;%&#x27; \
  IDENTIFIED BY &#x27;neutron123&#x27;;

**创建服务凭据**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
#创建用户 neutron 密码为 neutron
openstack user create --domain default --password-prompt neutron

#为neutron用户赋予admin权限
openstack role add --project service --user neutron admin

#创建网络服务
openstack service create --name neutron --description &quot;Openstack Networking&quot; network 

#创建外部端点
openstack endpoint create --region RegionOne network public http://controller:9696 

#创建内部端点
openstack endpoint create --region RegionOne network internal http://controller:9696  

#创建管理员端点
openstack endpoint create --region RegionOne network admin http://controller:9696

### [](#安装配置组件-3)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
#安装软件包
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

#编辑配置文件 /etc/neutron/neutron.conf
vi /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2  #表示Neutron使用的核心插件，这里为ml2
service_plugin =   #表示Neutron使用的服务插件，这里为空字符串（即未设置）
transport_url = rabbit://openstack:openstack123@controller  #表示消息传输的URL，这里为使用RabbitMQ消息队列服务的地址
auth_strategy = keystone  #表示认证策略，这里为keystone
notify_nova_on_port_status_changes = true  #当某个port状态有变化时是否通知Nova，这里设置为true
notify_nova_on_port_data_changes = true  #当某个port数据有变化时是否通知Nova，这里设置为true

[database]
connection = mysql+pymysql://neutron:neutron123@controller/neutron  #表示Neutron使用的数据库连接信息

[keystone_authtoken]
www_authenticate_url = http://controller:5000  #表示认证时使用的URL，这里为Keystone服务的URL
auth_url = http://controller:5000  #表示认证时使用的URL，这里为Keystone服务的URL
memcached_servers = controller:11211  #表示存储缓存信息的Memcached服务地址，这里为controller服务器的11211端口
auth_type = password  #表示认证类型，这里为password
project_domain_name = default  #表示Neutron使用的项目域名，这里为默认值default
user_domain_name = default  #表示Neutron使用的用户域名，这里为默认值default
project_name = service  #表示Neutron使用的项目名称，这里为service
username = neutron  #表示Neutron使用的用户名，这里为neutron
password = neutron  #表示Neutron使用的用户密码，这里为neutron

#文件最下面新增
[nova]
auth_url = http://controller:5000  #表示Nova服务使用的Keystone认证URL，这里为controller服务器的5000端口
auth_type = password  #表示认证类型，这里为password
project_domain_name = default  #表示Nova服务使用的项目域名，这里为默认值default
user_domain_name = default  #表示Nova服务使用的用户域名，这里为默认值default
region_name = RegionOne  #表示Nova服务所在的区域名称，这里为RegionOne
project_name = service  #表示Nova服务使用的项目名称，这里为service
username = nova  #表示Nova服务使用的用户名，这里为nova
password = nova  #表示Nova服务使用的用户密码，这里为nova

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp  #表示存储锁文件的路径，这里为/var/lib/neutron/tmp

1
2
3
4
5
6
7
8
9
10
11
12
13
vi /etc/neutron/plugins/ml2/ml2_conf.ini

#文末新增
[ml2]
type_drivers = flat.vlan  #表示网络类型的驱动程序类型，这里有两种类型：flat和vlan
tenant_network_types =  #表示拥有哪些网络类型的租户网络，这里为空，即租户无法使用任何网络类型
mechanism_drivers = linuxbridge  #表示底层网络设备驱动程序类型，这里为linuxbridge
extension_drivers = port_security  #表示支持的扩展驱动程序类型，这里为port_security，表示支持安全端口功能
flat_networks = extnet  #表示该服务支持的平面网络名称列表，这里为extnet

[securitygroup]
enable_ipset = true  #指定是否启用ipset安全组驱动程序，这里为true

1
2
3
4
5
6
7
8
9
10
11
12
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
#文件末新增
[linux_bridge]
physical_interface_mappings = extnet:ens33  #指定Linux网桥的物理网络接口名称和虚拟网络名称之间的映射关系，这里为extnet:ens33，表示将虚拟网络extnet连接到物理网络接口ens33上

[vxlan]
enable_vxlan = false  #指定是否启用VXLAN网络，这里为false，表示不使用VXLAN网络
 
[securitygroup]
enable_security_group = true  #指定是否启用安全组，这里为true，表示启用安全组功能
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver  #指定防火墙驱动程序类型，这里为neutron.agent.linux.iptables_firewall.IptablesFirewallDriver，表示使用iptables防火墙驱动程序
#配置内核

1
2
3
4
#配置内核
vi /etc/sysctl.conf
net.bridge.bridge-nf-call-iptables = 1  #表示当Linux网桥接收到网络数据包时，将会检查iptables过滤规则，并按照规则进行过滤。这个参数在使用Linux网桥作为虚拟网络设备时非常重要，因为它可以确保虚拟网络中的数据传输符合安全策略，提高网络的可靠性和安全性
net.bridge.bridge-nf-call-ip6tables = 1  #表示当Linux网桥接收到IPv6网络数据包时，将会检查ip6tables过滤规则，并按照规则进行过滤。这个参数同样对于使用IPv6地址的虚拟网络设备非常重要。

1
2
3
4
5
vi /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge  #指定DHCP代理的接口驱动程序类型为Linux桥接
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq  #指定DHCP代理的接口驱动程序类型为Linux桥接
enable_isolated_metadata = true  #指定是否启用隔离元数据代理服务，这里为true，表示启用服务

1
2
3
4
vi /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = controller  #指定OpenStack Nova API的元数据服务主机地址，这里为controller，表示元数据服务运行在OpenStack控制节点上
metadata_proxy_shared_secret = xier123  #指定元数据代理与元数据服务之间共享的密钥，这里为xier123，需要确保元数据服务与元数据代理使用相同的密钥

1
2
3
4
5
6
7
8
9
10
11
12
vi /etc/nova/nova.conf
[neutron]
auth_url = http://controller:5000  #指定OpenStack Identity服务的URL地址
auth_type = password  #指定认证类型为密码认证
project_domain_name = default  #指定项目域名称为默认
user_domain_name = default  #指定用户域名称为默认
region_name = RegionOne  #指定OpenStack服务所在的区域/地区名称
project_name = service  #指定Nova服务的项目名称
username = neutron  #指定连接到Neutron服务时使用的用户名
password = neutron  #指定连接到Neutron服务时使用的密码
service_metadata_proxy = true  #表示Nova计算服务将启用元数据代理服务，并将元数据代理服务与Neutron服务集成
metadata_proxy_shared_secret = xier123  #指定元数据代理与元数据服务之间共享的密钥，这里为xier123，需要确保元数据代理与元数据服务使用相同的密钥

### [](#同步数据库并启动服务)同步数据库并启动服务

1
2
3
4
5
6
7
8
9
#创建软连接
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini  /etc/neutron/plugin.ini

#同步数据库
su -s /bin/sh -c &quot;neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head&quot; neutron

#重启服务并设置开机自启
systemctl restart neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
 systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent

## [](#compute节点-1)compute节点

### [](#安装配置组件-4)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
#安装软件包
yum install openstack-neutron-linuxbridge ebtables ipset -y

#修改配置文件 /etc/neutron/neutron.conf
vi /etc/neutron/neutron.conf

[DEFAULT]
transport_url = rabbit://openstack:openstack123@controller  #指定RabbitMQ的连接URL地址
auth_strategy = keystone  #指定认证策略为Keystone认证

[keystone_authtoken]
www_authenticate_url = http://controller:5000  #指定用于向用户提供身份验证页面的URL地址
auth_url = http://controller:5000  #指定Keystone服务的URL地址
memcached_servers = controller:11211  #指定用于缓存令牌(token)的Memcached服务地址和端口号
auth_type = password  #指定认证类型为密码认证
project_domain_name = default  #指定项目域名称为默认(default)
user_domain_name = default  #指定用户域名称为默认(default)
project_name = service  #指定服务所在的项目名称
username = neutron  #指定连接到Keystone服务时使用的用户名
password = neutron  #指定连接到Keystone服务时使用的密码

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp  #指定保存锁文件的路径

1
2
3
4
5
6
7
8
9
10
11
vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini
#文件末新增
[linux_bridge]
physical_interface_mappings = extnet:ens33  #指定了网桥与物理网络接口的映射关系

[vxlan]
enable_vxlan = false  #参数指定是否启用VXLAN虚拟化网络

[securitygroup]
enable_security_group = true  #参数指定是否启用安全组功能
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver  #参数指定使用哪个防火墙驱动程序，这里使用基于iptables的驱动程序IptablesFirewallDriver

1
2
3
4
5
6
7
8
9
10
vi /etc/nova/nova.conf
[neutron]
auth_url = http://controller:5000  #参数指定了认证服务的访问地址
auth_type = password  #参数指定了认证方式
project_domain_name = default  #参数指定了所使用的域名
user_domain_name = default  #参数指定了所使用的域名
region_name = RegionOne  #参数指定了Neutron服务所属的区域名称
project_name = service  #参数指定了Neutron服务所属的项目名称
username = neutron  #用户名
password = neutron  #密码

### [](#启动服务)启动服务

1
2
systemctl restart openstack-nova-compute
systemctl enable neutron-linuxbridge-agent --now

# [](#Dashboard服务)Dashboard服务

## [](#controller节点-5)controller节点

### [](#安装配置组件-5)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
#安装软件包
yum install openstack-dashboard -y

#修改配置文件 /etc/openstack-dashboard/local_settings
vi /etc/openstack-dashboard/local_settings

OPENSTACK_HOST = &quot;controller&quot;  #参数指定了OpenStack服务所在主机的名称或IP地址
ALLOWED_HOSTS = [&#x27;*&#x27;]  #参数指定了OpenStack服务所在主机的名称或IP地址，这里表示允许所有
SESSION_ENGINE = &#x27;django.contrib.sessions.backends.cache&#x27;  
CACHES = &#123;
    &#x27;default&#x27;: &#123;        
        &#x27;BACKEND&#x27;: &#x27;django.core.cache.backends.memcached.MemcachedCache&#x27;,   #
        &#x27;LOCATION&#x27;: &#x27;controller:11211&#x27;,
    &#125;  #指定了使用缓存来存储用户会话信息的方式，这里使用的是Memcached
&#125;
OPENSTACK_KEYSTONE_URL = &quot;http://%s:5000/v3&quot; % OPENSTACK_HOST  #参数指定了Keystone认证服务的访问地址
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True  #新增,表示支持多域模式
OPENSTACK_API_VERSIONS = &#123;
    &quot;identity&quot;: 3,
    &quot;image&quot;: 2,
    &quot;volume&quot;: 2,
&#125;  #新增，指定api版本信息
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = &#x27;Default&#x27; #新增，参数指定了默认域名
OPENSTACK_KEYSTONE_DEFAULT_ROLE = &#x27;user&#x27; #新增 ，参数指定了默认角色，这里为user
OPENSTACK_NEUTRON_NETWORK = &#123;
    &#x27;enable_auto_allocated_network&#x27;: False,
    &#x27;enable_distributed_router&#x27;: False,
    &#x27;enable_fip_topology_check&#x27;: True,
    &#x27;enable_ha_router&#x27;: False,
    &#x27;enable_lb&#x27;: False,
    &#x27;enable_firewall&#x27;: False,
    &#x27;enable_vpn&#x27;: False,
    &#x27;enable_ipv6&#x27;: True,
    &#x27;enable_fip_topology_check&#x27;: False,
    # TODO(amotoki): Drop OPENSTACK_NEUTRON_NETWORK completely from here.
    # enable_quotas has the different default value here.
    &#x27;enable_quotas&#x27;: False,
    &#x27;enable_rbac_policy&#x27;: True,
    &#x27;enable_router&#x27;: False,
    &#x27;default_dns_nameservers&#x27;: [],
    &#x27;supported_provider_types&#x27;: [&#x27;*&#x27;],
    &#x27;segmentation_id_range&#x27;: &#123;&#125;,
    &#x27;extra_provider_types&#x27;: &#123;&#125;,
    &#x27;supported_vnic_types&#x27;: [&#x27;*&#x27;],
    &#x27;physical_networks&#x27;: [],

&#125;
TIME_ZONE = &quot;Asia/Shanghai&quot;  #新增，参数用于指定时区，这里设置为&quot;Asia/Shanghai&quot;，以符合中国标准时间
WEBROOT = &#x27;/dashboard&#x27;  #新增，用于指定OpenStack仪表盘的访问路径，这里设置为/dashboard

1
2
3
vi /etc/httpd/conf.d/openstack-dashboard.conf 

WSGIApplicationGroup %&#123;GLOBAL&#125;  #新增

### [](#启动服务并验证-1)启动服务并验证

1
2
3
4
5
6
7
8
#重启服务
systemctl restart httpd memcached

#验证
访问: 192.168.10.10/dashboard
域: default
用户名: admin
密码: admin

- ![](https://img2.imgtp.com/2024/03/12/tIypOq4x.png)

![](https://img2.imgtp.com/2024/03/12/vJfRTGJz.png)

# [](#Cinder服务)Cinder服务

## [](#controller节点-6)controller节点

### [](#先决条件-5)先决条件

1
2
3
4
5
#创建数据库赋予权限
mysql -uroot -p000000

MariaDB [(none)]&gt; CREATE DATABASE cinder;
MariaDB [(none)]&gt; grant all privileges on cinder.* to &#x27;cinder&#x27;@&#x27;%&#x27; identified by &#x27;cinder123&#x27;;

#### [](#创建用户凭据)创建用户凭据

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
#创建用户 密码为 cinder
openstack user create --domain default --password-prompt cinder

#添加到admin
openstack role add --project service --user cinder admin

#创建和服务实体 
openstack service create --name cinderv2 --description &quot;OpenStack Block Storage&quot; volumev2

openstack service create --name cinderv3 --description &quot;OpenStack Block Storage&quot; volumev3
 
#创建块存储服务 API 端点
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
  

### [](#安装配置组件-6)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
#安装软件包
yum install openstack-cinder -y

#编辑配置文件 /etc/cinder/cinder.conf
vi /etc/cinder/cinder.conf

[DEFAULT]
transport_url = rabbit://openstack:openstack123@controller
auth_strategy = keystone
my_ip = 192.168.10.10

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[database]
connection = mysql+pymysql://cinder:cinder123@controller/cinder

1
2
3
vi /etc/nova/nova.conf
[cinder]
os_region_name = RegionOne

### [](#完成安装)完成安装

1
2
3
4
5
6
7
8
9
10
11
12
13
#同步数据库
sudo su -s /bin/sh -c &quot;cinder-manage db sync&quot; cinder

systemctl restart openstack-nova-api
systemctl start openstack-cinder-api openstack-cinder-scheduler
systemctl enable openstack-cinder-api openstack-cinder-scheduler
#验证
openstack volume service list
+------------------+------------+------+---------+-------+----------------------------+
| Binary           | Host       | Zone | Status  | State | Updated At                 |
+------------------+------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller | nova | enabled | up    | 2024-03-12T08:29:34.000000 |
+------------------+------------+------+---------+-------+----------------------------+

## [](#compute节点-2)compute节点

### [](#先决条件-6)先决条件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
#安装软件包
yum install lvm2 device-mapper-persistent-data -y

#启动服务并设置开机自启动
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service

#创建物理卷
pvcreate /dev/sdb

#创建 LVM 卷组
vgcreate cinder-volumes /dev/sdb

vi /etc/lvm/lvm.conf
#在130行下插入
filter = [ &quot;a/sda/&quot;, &quot;a/sdb/&quot;, &quot;r/.*/&quot;]   #filter过滤器中每个项目的开头都为&quot;a&quot;或&quot;r&quot;，用于接受或拒绝某个设备

### [](#安装配置组件-7)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
#安装软件包
yum install openstack-cinder targetcli python-keystone -y

#编辑配置文件 /etc/cinder/cinder.conf
vi /etc/cinder/cinder.conf

[DEFAULT]
transport_url = rabbit://openstack:openstack123@controller
auth_strategy = keystone
my_ip = 192.168.10.10
enabled_backends = lvm
glance_api_servers = http://controller:9292

[database]
connection = mysql+pymysql://cinder:cinder123@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

#文末新增
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver  #表示使用的卷驱动程序
volume_group = cinder-volumes  #表示使用的卷驱动程序组名称
iscsi_protocol = iscsi  #表示卷访问使用的 iSCSI 协议版本，这里是iSCSIv2(v1通常不在被使用)
iscsi_helper = lioadm   #表示使用 Linux iSCSI Target 和 initiator 之间的 LIO 工具来管理 iSCSI 连接

### [](#启动服务并验证-2)启动服务并验证

1
2
3
4
5
6
7
8
9
10
11
12
#启动服务并设置为开机自启动
systemctl start openstack-cinder-volume.service target.service
systemctl enable openstack-cinder-volume.service target.service

#在控制节点验证
openstack volume service list
+------------------+-------------+------+---------+-------+----------------------------+
| Binary           | Host        | Zone | Status  | State | Updated At                 |
+------------------+-------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller  | nova | enabled | up    | 2024-03-12T08:47:44.000000 |
| cinder-volume    | compute@lvm | nova | enabled | up    | 2024-03-12T08:47:38.000000 |
+------------------+-------------+------+---------+-------+----------------------------+

# [](#Swift服务)Swift服务

## [](#controller节点-7)controller节点

### [](#先决条件-7)先决条件

**创建服务凭据**

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
#创建服务 密码为swift
openstack user create --domain default --password-prompt swift

#添加到admin中
openstack role add --project service --user swift admin

#创建服务实体
openstack service create --name swift --description &quot;OpenStack Object Storage&quot; object-store

#创建对象存储服务 API 端点
openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s

openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s

openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1

### [](#安装配置组件-8)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
#安装软件包
yum install openstack-swift-proxy python-swiftclient python-keystoneclient python-keystonemiddleware memcached -y

#从对象存储中获取代理服务配置文件 源存储库
curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/proxy-server.conf-sample

#编辑配置文件 /etc/swift/proxy-server.conf
vi /etc/swift/proxy-server.conf
[DEFAULT]
bind_port = 8080
user = swift
swift_dir = /etc/swift

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
account_autocreate = True

[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = admin,user

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = swift
delay_auth_decision = True

[filter:cache]
use = egg:swift#memcache
memcache_servers = controller:11211

## [](#compute节点-3)compute节点

### [](#先决条件-8)先决条件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
#安装支持实用程序包
yum install xfsprogs rsync

#将设备格式化
mkfs.xfs /dev/sdb
mkfs.xfs /dev/sdc

#创建挂载点目录
mkdir -p /srv/node/sdb
mkdir -p /srv/node/sdc

#查找新分区
blkid

#编辑文件并向其添加内容 /etc/fstab
UUID=&quot;&lt;UUID-from-output-above&gt;&quot; /srv/node/sdb xfs noatime 0 2
UUID=&quot;&lt;UUID-from-output-above&gt;&quot; /srv/node/sdc xfs noatime 0 2

#挂载设备
mount /srv/node/sdb
mount /srv/node/sdc

#创建或编辑文件以包含以下内容 /etc/rsyncd.conf
vi /etc/rsyncd.conf

uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 192.168.10.20

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock

#启动服务并设置为开机自启动
systemctl enable rsyncd.service
systemctl start rsyncd.service

### [](#安装配置组件-9)安装配置组件

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
#安装软件包
yum install openstack-swift-account openstack-swift-container openstack-swift-object -y

#获取配置文件
curl -o /etc/swift/account-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/account-server.conf-sample
curl -o /etc/swift/container-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/container-server.conf-sample
curl -o /etc/swift/object-server.conf https://opendev.org/openstack/swift/raw/branch/master/etc/object-server.conf-sample

编辑配置文件 /etc/swift/account-server.conf
vi /etc/swift/account-server.conf

[DEFAULT]
bind_ip = 192.168.10.20
bind_port = 6202
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon account-server

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
vim /etc/swift/container-server.conf

[DEFAULT]
bind_ip = 192.168.10.20
bind_port = 6201
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon container-server

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
vim /etc/swift/object-server.conf

[DEFAULT]
bind_ip = 192.168.10.20
bind_port = 6200
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon object-server

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock

1
2
3
4
5
#创建目录并赋予挂载目录权限
chown -R swift:swift /srv/node
mkdir -p /var/cache/swift
chown -R root:swift /var/cache/swift
chmod -R 775 /var/cache/swift

## [](#创建和分发initial-rings)创建和分发initial rings

### [](#controller)controller

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
cd /etc/swift/
swift-ring-builder account.builder create 10 3 1
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6202 --device sdc --weight 100
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6202 --device sdd --weight 100
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6202 --device sdc --weight 100 
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6202 --device sdd --weight 100  
swift-ring-builder account.builder
swift-ring-builder account.builder rebalance

swift-ring-builder container.builder create 10 3 1
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6201 --device sdc --weight 100
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6201 --device sdd --weight 100
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6201 --device sdc --weight 100
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6201 --device sdd --weight 100
swift-ring-builder container.builder
swift-ring-builder container.builder rebalance

swift-ring-builder object.builder create 10 3 1
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6200 --device sdc --weight 100
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.10.10 --port 6200 --device sdd --weight 100 
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6200 --device sdc --weight 100
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.10.20 --port 6200 --device sdd --weight 100
swift-ring-builder object.builder
swift-ring-builder object.builder rebalance
scp swift.conf compute:/etc/swift/swift.conf
chown -R root:swift /etc/swift
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service

### [](#compute)compute

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
chown -R root:swift /etc/swift
systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl enable openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl start openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service

                
                __END__

            
            
                
    
        ![](/images/User/vavtarsheep.jpg)
    
    
        文章作者：CloudSheep
        

        文章出处：[OpenStack搭建（2024版）](/2023/12/13/OpenStack%E6%90%AD%E5%BB%BA/)
        

        作者签名：不啻微芒，造炬成阳
        

        关于主题：[hexo-blog](https://github.com/lsSheep/hexo-blog.git)
        

        版权声明：文章除特别声明外，均采用 [BY-NC-SA](https://creativecommons.org/licenses/by-nc-nd/4.0/) 许可协议，转载请注明出处
        

    
    

                
                    
                        分类：
                        
                            [教程](/category/%E6%95%99%E7%A8%8B/)
                        
                    
                
                
                    
                        标签：
                        
                            [OpenStack](/tag/OpenStack/)
                        
                            [运维](/tag/%E8%BF%90%E7%BB%B4/)
                        
                    
                
                
                    
                        [« ](/2023/12/14/hello-world/) 上一篇：    [Hello World](/2023/12/14/hello-world/)
                        

                    
                    
                        [» ](/2023/12/12/Kubernetes%E6%B5%81%E7%A8%8B%E5%9B%BE/) 下一篇：    [Kubernetes流程图](/2023/12/12/Kubernetes%E6%B5%81%E7%A8%8B%E5%9B%BE/)
