---
title: 部署Nginx+KeepAlived高可用集群
date: 2024-06-18 00:00:00
categories:
  - 负载均衡
tags:
  - 前端
  - nginx
  - keepalived
---
# [](#概述)**概述**
**前端使用nginx与keepalived构建高可用集群，是一种常见的提升系统稳定性和可靠性的方法。**
## [](#Nginx)**Nginx**
 **是一款高性能的HTTP和反向代理的服务器主要应用于负载均衡和静态内容服务（前端），同时也提供了IAMP&#x2F;SMTP&#x2F;POP3服务。常用于负载均衡和静态内容服务。**
## [](#KeepAlived)**KeepAlived**
 **用于监听和故障转移的工具，基于VRRP协议（虚拟路由冗余协议）实现，可以监控服务器的状态，并在主服务器故障时自动切换到备份服务器。主要用于提高服务的高可用性**
keepalivd提供了几种不同的冗余方式
- 主备模式（master-slave）
- 负载均衡模式（load balancing）
- 主备负载均衡模式（master-slave load balancing）
## [](#双机高可用的方法)**双机高可用的方法**
- **KeepAlived+Nginx双机主从模式**
- **KeepAlived+Nginx双机主主模式**
# [](#Nginx-1)**Nginx**
## [](#正向代理)**正向代理**
 **是帮助内网客户端访问外网服务器的代理方式。在这种模式下，客户端无法直接访问目标服务器，但可以通过一个代理服务器进行访问。**
- **其主要目的是解决访问限制问题，如绕过地区限制或组织内部的访问限制，访问被封锁的资源。**
## [](#反向代理)**反向代理**
**是把外网客户端的请求转发给内网服务器的代理方式。在这种模式下，客户端并不知道实际提供服务的服务器地址，只与代理服务器进行交互。**
- **作用目的：一方面是作为负载均衡器，将客户端请求分发到多个后端服务器；另一方面起到安全防护的作用，过滤恶意请求和响应。**
## [](#负载均衡)**负载均衡**
**负载均衡是将工作负载均匀地分配到多个服务器或网络设备上，以防止任何单一节点过载，从而提高整体系统的可用性和性能。是一种用于在多台服务器之间分配网络流量的技术，旨在优化系统资源利用率、提高服务可用性、增强系统的伸缩性和容错率**
工作负载的度量标准：CPU使用率、内存使用率、磁盘I&#x2F;O、网络带宽、并发用户数量、请求率、事物处理器等
**客户端负载均衡：发生在客户端。客户端在发送请求前，根据负载均衡策略（如随机、轮询等）选择一台服务器进行访问，由自己来决定所访问的服务器**
**服务端负载均衡：发生在服务端。通常通过专门的负载均衡器（如Nginx、HAProxy等）接收客户端的请求，然后根据负载均衡算法（如轮询、加权轮询、最少连接等）将请求分发到后端的多台服务器中的一台进行处理。将服务请求发送给目标网站，由目标网站来决定所访问的服务器（Nginx+KeepAlived）**
**实现负载均衡的方式：HTTP从定向负载均衡、DNS域名解析负载均衡、反向代理负载均衡、IP负载均衡、数据链路层负载均衡、硬件负载均衡器、软件负载均衡器**
## [](#Nginx环境搭建)**Nginx环境搭建**
### [](#安装nginx第三方库)安装nginx第三方库
**执行以下命令进行安装**
1
yum -y install gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-develyum -y install gcc gcc-c++ automake
### [](#下载Nginx解压包并解压)下载Nginx解压包并解压
- 在官网上直接下载   网址：[http://nginx.org/en/doload.html](http://nginx.org/en/download.html)
- 在Linux服务器上下载  `wget http://nginx.org/download/nginx-1.26.1.tar.gz`
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
# 创建nginx目录 
mkdir /opt/nginx
# 进入nginx目录
cd /opt/nginx 
# 下载nginx解压包  
wget http://nginx.org/download/nginx-1.26.1.tar.gz
# 解压文件
tar -xzvf nginx-1.26.1.tar.gz
### [](#安装Nginx)安装Nginx
1
2
3
4
5
6
7
8
# opt/nginx文件夹下创建新文件夹Nginx-1.26.1_install
cd /opt/nginx
mkdir nginx-1.26.1_install
# 进入之前解压后得到的文件夹nginx-1.26.1
cd nginx-1.26.1
#运行configure脚本程序，可以直接运行./configure,也可以通过--prefix=path 指定nginx的安装目录
conf：该目录存放了Nginx的所有配置文件，该文件夹下包含nginx.conf文件，它是Nginx服务器的住配置文件，其他文件则是用    来配置Nginx的相关功能。
html：该目录存放了Nginx服务器在运行过程中调用的一些html文件。
logs：该目录存放了Nginx服务器的日志。
sbin：该目录中只包含了一个文件-nginx，它就是Nginx服务器的主程序。
### [](#修改配置文件)修改配置文件
1
2
3
#修改nginx.conf文件中的端口，改为81
cd /opt/nginx/nginx-1.26.1_install/conf/
vi nginx.conf
### [](#启动Nginx服务器)启动Nginx服务器
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
#在启动服务器之前，我们可以通过如下指令来查看Nginx服务器配置文件是否有语法错误：
/opt/nginx/nginx-1.26.1_install/sbin/nginx -t
#绝对路径
/opt/nginx/nginx-1.26.1_install/sbin/nginx
#在nginx-1.26.1_install文件夹中时的相对路径
./sbin/nginx -t
#通过如下指令可以查看Nginx服务器版本
./sbin/nginx -v
#使用默认配置启动Nginx
./sbin/nginx
#查看nginx进程状态
ps -ef|grep nginx
#停止Nginx服务器
#绝对路径
/opt/nginx/Nginx-1.26.1_install/sbin/nginx -s stop
#Nginx-1.26.1_install文件夹下相对路径
./sbin/nginx -s stop
#重启Nginx服务器
/opt/nginx/Nginx-1.26.1_install/sbin/nginx -s reopen
#重新载入配置文件
/opt/nginx/Nginx-1.26.1_install/sbin/nginx -s reload
### [](#验证)验证
1
2
3
#需要关闭Linux服务器的防火墙
systemctl stop firewall &amp;&amp;systemctl disbale firewalld
http:192.168.10.100:81
## [](#虚拟主机)**虚拟主机**
- **一个服务器，多个站点：在nginx的配置中，虚拟主机是一个重要概念，它允许单个nginx服务器为多个域名或IP地址提供服务，而不需要为每个域名或IP地址运行单独的nginx实例**
- **基于名称的虚拟主机：最常见的虚拟主机是基于服务器名称（通常是域名）的**
例如，您可能有一个服务器，它同时为 `www.example1.com` 和 `www.example2.com` 提供服务。`nginx` 根据请求中的 `Host` 头字段来确定应该将请求路由到哪个虚拟主机
- **基于 IP 的虚拟主机 ： nginx 也支持基于 IP 地址的虚拟主机。这意味着您可以为每个 IP 地址配置一个独立的虚拟主机**
由于 IP 地址的稀缺性和成本，这种配置方式通常不如基于名称的虚拟主机实用
- **配置文件 ：在 nginx 中，虚拟主机的配置通常位于 nginx.conf 文件中的  serve r块内，或者位于包含在其他位置并由 nginx.conf  引入的单独文件中。每个 server 块代表一个虚拟主机，并包含该虚拟主机的所有配置指令**
- **资源隔离 ：虽然 nginx 在单个进程中处理所有虚拟主机的请求，但每个虚拟主机都拥有其自己的配置和上下文。这意味着一个虚拟主机的错误或配置问题不会影响其他虚拟主机**
- **扩展性 ：由于 nginx 的模块化设计，您可以为不同的虚拟主机安装和使用不同的模块，从而根据每个站点的需求进行定制和优化**
### [](#虚拟主机配置)虚拟主机配置
在配置文件 `/opt/nginx/nginx-1.26.1_install/conf/nginx.conf`中的 `server`项进行配置
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
# 打开nginx.conf配置文件
vi /opt/nginx/nginx-1.26.1_install/conf/nginx.conf
#找到 server 项进行编辑
#虚拟主机的配置
    server &#123;
        listen       81;       #该虚拟主机监听的端口
        server_name  caro2o.worfwode.cn;   #虚拟主机监听的域名/ip(此处时基于域名)
        #charset koi8-r;
        #access_log  logs/host.access.log  main;
        #当请求到当前虚拟主机后，映射什么访问路径，/表示映射根路径请求到哪里
        location / &#123;
            root /www/worfcode/caro2o;  #表示访问当前路径时，访问哪个目录中的内容，html表示的是相对路径
            index  index.html;  #当请求路径后面不携带任意文件时，默认访问的文件名（可以只保留一个）
        &#125;
可以利用本地hosts文件来使用域名访问
## [](#日志文件)**日志文件**
存放于nginx的安装目录中
1
2
3
4
5
#进入logs目录中
cd /opt/nginx/nginx-1.26.1_install/logs
#查看日志文件的内容(最后20条)
tail -f -n20 access.log、
每次刷新web网页都会增加一条新的日志记录
### [](#日志文件切分)日志文件切分
**nginx日志文件的切分是一个重要的管理任务，有助于保持日志文件的清晰和可管理性**
**切分方式**
- **使用logrotate工具切分：**
- **自定义nginx配置文件实现文件自切分**
## [](#Location配置)**Location配置**
### [](#location语法规则)location语法规则
- **基本语法**
`location` 块的基本语法结构如下：
1
2
3
location[修饰符] /uri/ &#123;
   #指令...
&#125;
修饰符是是可选的，用于指定匹配方式，如  （`=` 精确匹配， `~` 区分大小写的正则匹配， `~*`不区分大小写的正则匹配，`/` 通用匹配，任何请求都会匹配得到）
`/uri/` 是用于匹配请求url的字符串或正则表达式
- **修饰符说明**
**1. 精确匹配 ：使用 `=` 修饰符，仅当请求 URI 与指定字符串完全相等时匹配**
1
2
3
location = / &#123;  
    # 仅处理根路径/的请求  
&#125;
**2. 正则表达式匹配**
`~`   区分大小写的正则 表达式匹配
`~*` 不区分大小写的正则表达式匹配
1
2
3
4
5
6
7
location ~ /example/ &#123;  
    # 匹配 http://www.example.com/example/ 但不匹配 http://www.example.com/Example/  
&#125;  
location ~* /example/ &#123;  
    # 匹配 http://www.example.com/example/ 和 http://www.example.com/Example/  
&#125;
**3. 匹配以uri开头**
使用 `^~` 修饰符，仅当 URI 以指定的字符串开头时匹配（前缀匹配，以XXX开头）
1
2
3
location ^~ /img/ &#123;  
    # 匹配所有以 /img/ 开头的 URI  
&#125;
**4. 无修饰符**
默认为前缀匹配，匹配uri的前缀
1
2
3
location /api/ &#123;  
    # 匹配所有以 /api/ 开头的 URI  
&#125;
- **优先级规则**
当多个 `location` 块可能会匹配同一个请求时，Nginx会根据以下规则来确定优先使用哪个 `location` 块
1
2
3
4
5
6
7
1. 首先检查是否有精确匹配 =
2. 其次检查是否有以 ^~ 开头的匹配
3. 接下来按照配置文件中的顺序检查正则表达式的匹配 ~ 或 ~*
4. 最后是通用匹配 /（没有修饰符的 location块）
- **注意事项**
`location` 指令必须放在 `server` 指令中，而不能放在 `http` 或其他指令中。
`location` 指令后面必须跟一个大括号 `&#123;&#125;`，用于包含 `location` 相关的配置。
在 `location` 中可以配置一些特殊的变量，如 `$uri` 表示当前请求的 URI，`$args` 表示请求的查询参数等。
`location` 指令可以与 `proxy_pass`、`fastcgi_pass` 等指令结合使用，将请求转发到其他服务器或处理程序中。
## [](#反向代理配置)**反向代理配置**
- **配置nginx反向代理可以实现负载均衡、缓存静态内容，提升服务器的性能**
- **配置反向代理可以隐藏后端服务器、SSL&#x2F;STL终止、控制访问，提升安全性**
**通过编辑nginx服务器的 `nginx.conf` 文件来实现反向代理的配置**
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
#打开nginx服务器的配置文件进行编辑
vi /opt/nginx/nginx-1.26.1_install/conf/nginx.conf
#以下是一个反向代理的示例
server &#123;  
    listen 80;  # 监听80端口（HTTP）  
    server_name example.com;  # 你的域名或IP地址  
    location / &#123;  
        proxy_pass http://192.168.10.100:8080/;  # 后端服务器的地址(这里以tomcat服务器为例)  
        proxy_set_header Host $host;  
        proxy_set_header X-Real-IP $remote_addr;  
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
        proxy_set_header X-Forwarded-Proto $scheme;  
    &#125;  
&#125;
`proxy_pass` 用于指定要到代理的后端服务器的地址。可以使用域名&#x2F;IP地址&#x2F;端口等，例如：`http:192.168.10.100:8080`
`proxy_set_header`用于设置发送到后端服务器的请求头。可以帮助后端服务器识别原始信息的来源（原始信息的IP&#x2F;域名&#x2F;端口号等）
**启用配置并验证是否成功**
1
2
3
4
5
#重新加载nginx.conf配置文件
/opt/nginx/nginx-1.26.1_install/sbin/nginx -s reload
#使用curl命令或者直接在本地服务器上直接访问验证
curl http://192.168.10.100/tomcat/test.jsp
如果你的后端服务器支持HTTPS服务，并且你也希望Nginx服务器也能通过HTTPS代理请求，那么你需要配置SSL&#x2F;STL
## [](#负载均衡配置)负载均衡配置
**nginx服务器配置负载均衡何以实现：提高系统性能、增强系统可用性、实现动态扩展、优化资源利用（提升系统资源利用率）、提高安全性、简化管理（便于管理）**
**实现nginx服务器的负载均衡配置同样可以利用编辑 `conf` 目录中的 `nginx.conf` 文件实现**
1
2
3
4
5
6
7
8
9
# 在nginx.conf文件中添加一个upstream组到http块中
http &#123;  
    upstream tomcats &#123;  
        server 192.168.10.100:8080 weight=1 max_fails=2 fail_timeout=30s;  
        server 192.168.10.100:8090 weight=2 max_fails=2 fail_timeout=30s;  
        server 192.168.1.10 backup;  # 备份服务器  
    &#125;  
    ...  
&#125;
`weight` 用来定义服务器处理请求的权重，默认为1，权重越大收到的请求越多
`backup` 备份服务器，当其他非备份服务器都不可使用时，才会请求该服务器
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
# 在http块中配置一个或多个server块用于处理传入的请求，需要设置listen指令来监听端口，
http &#123;  
    ...  
    server &#123;  
        listen 80;  
        server_name example.com;  # 指定服务器的名称或域名
        location / &#123;  
            proxy_pass http://backend_group;  # 使用前面定义的 upstream 组  
            proxy_set_header Host $host;  
            proxy_set_header X-Real-IP $remote_addr;  
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
        &#125;  
    &#125;  
    ...  
&#125;
`proxy_pass` 将匹配的请求代理转发到前面定义的 upstream 组
`proxy_set_header` 用于修改或添加 HTTP 请求头，以便后端服务器能够正确地处理请求
**启用配置并验证是否成功**
1
2
3
4
#重新加载nginx.conf配置文件
/opt/nginx/nginx-1.26.1_install/sbin/nginx -s reload
#通过浏览器直接访问所设置的域名或IP地址来验证所配置的负载均衡是否能正常工作
**Nginx 提供了多种负载均衡算法，可以根据实际需求选择。常见的算法包括：**
轮询（Round Robin）：默认的负载均衡算法，按顺序将请求分发到每个后端服务器。
加权轮询（Weighted Round Robin）：根据服务器的权重来分发请求，权重越大，接收的请求越多。
最少连接数（Least Connections）：将请求分配给当前连接数最少的服务器。
IP 哈希（IP Hash）：根据客户端 IP 的哈希值来分配请求，确保同一个客户端的请求始终被分发到同一台服务器。
# [](#KeepAlived实现Nginx高可用)KeepAlived实现Nginx高可用
**这里以nginx+keepalived双机主备模式为例**
keepalived实现nginx高可用主要是基于VRRP协议（虚拟路由冗余协议）来实现
## [](#部署KeepAlived)部署KeepAlived
### [](#安装keepalived所需要的第三方工具)安装keepalived所需要的第三方工具
**本次部署专注于构建基于Nginx和Keepalived的高可用集群，已安装的Nginx相关工具将不再重复安装，仅补充构建Keepalived所需的缺失第三方工具**
1
2
# 安装libnl-3开发库
sudo yum install libnl3-devel libnl3-cli-devel
### [](#下载KeepAlived解压包并解压)下载KeepAlived解压包并解压
**进入官网，根据需求选择所需要的版本进行下载，网址：[Keepalived for Linux](https://keepalived.org/download.html)**
**利用远程终端工具（SecureFX、Xftp等），将下载的解压包上传至服务器的 `/opt/nginx` 目录中**
1
2
3
4
5
# 进入/opt/nginx目录中
cd /opt/nginx
#将上传至该目录的keepalived-2.3.1.tar.gz解压包进行解压
tar -xvzf keepalived-2.3.1.tar.gz
### [](#安装KeepAlived)安装KeepAlived
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
# 在nginx目录下创建一个新的keepalived文件夹，用来存放keepalived安装完成后的文件
mkdir /opt/nginx/keepalived
# 进入解压后得到的文件夹keepalived-2.3.1
cd /opt/nginx/keepalived-2.3.1
# 运行脚本程序进行安装
./configure/ --prefix=/opt/nginx/keepalived
# 运行完成后，该文件夹下会多出一个文件--Makefile,可执行以下命令进行编译安装keepalived
make &amp;&amp; make install
### [](#验证-1)验证
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
# 编译安装完成后如果显示以下内容，则表示安装完成
Use IPVS Framework       : Yes
IPVS use libnl           : Yes
IPVS syncd attributes    : No
IPVS 64 bit stats        : No
HTTP_GET regex support   : No
fwmark socket support    : Yes
Use VRRP Framework       : Yes
Use VRRP VMAC            : Yes
Use VRRP authentication  : Yes
With track_process       : Yes
With linkbeat            : Yes
Use NetworkManager       : No
Use BFD Framework        : No
SNMP vrrp support        : No
SNMP checker support     : No
SNMP RFCv2 support       : No
SNMP RFCv3 support       : No
DBUS support             : No
Use JSON output          : No
libnl version            : 3
Use IPv4 devconf         : No
Use iptables             : No
Use nftables             : No
init type                : systemd
systemd notify           : No
Strict config checks     : No
Build documentation      : No
Default runtime options  : -D
`WARNING`  这是一个警告，可能缺少keepalived所需要的第三方库&#x2F;工具
## [](#编写shell脚本监听nginx是否存活)编写shell脚本监听nginx是否存活
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
# 在keepalived目录下创建nginx_check.sh脚本文件
touch /opt/nginx/keepalived/nginx_check.sh
# 编辑nginx_check.sh脚本文件
mv /opt/nginx/keepalaved/nginx_check.sh
#!/bin/bash
A=$(ps -C nginx --no-header | wc -l)  #检查正在运行的nginx进程数量
if [ &quot;$A&quot; -eq 0 ]; then               #如果nginx没有运行，则启动它
    /opt/nginx/nginx-1.26.1_install/sbin/nginx
    sleep 2
    B=$(ps -C nginx --no-header | wc -l)     #再次检查nginx是否启动成功
    if [ &quot;$B&quot; -eq 0 ]; then
        echo &quot;Nginx failed to start, killing keepalived.&quot;
        killall keepalived
    else
        echo &quot;Nginx started successfully.&quot;
    fi
else
    echo &quot;Nginx is already running.&quot;
fi
**赋予 `nginx_check.sh`脚本文件可执行权限**
1
chmod +x /opt/nginx/keepalived/nginx_check.sh
## [](#配置Master节点)配置Master节点
**修改master节点的 `keepalived.conf` 文件**
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
#将keepalived.conf文件内的内容替换为以下内容
! Configuration File for keepalived
global_defs &#123;
   router_id wolfcode     #路由器标志
&#125;
#集群资源监控，组合track_script进行
vrrp_script check_haproxy &#123;
        script &quot;/opt/nginx/keepalived/nginx_check.sh&quot;  #检测nginx状态的脚本路径
        interval 2  #检测时间间隔
        weight -20  #若条件成立，权重减20
&#125;
vrrp_instance PROXY &#123;
        #设置当前主机为主节点，备用节点设置为BACKUP
        state MASTER
        #指定HA监测网络接口
        interface ens32
        #虚拟路由标识，同一个VRRP实例要使用同一个标识，主备机
        virtual_router_id 80
        #主节点时，内容为：
        unicast_src_ip 192.168.10.100
        #设置优先级，确保主节点的优先级高过备用节点
        priority 100
        #用于设定主备节点时间同步检查时间间隔
        advert_int 2
        #设置主备节点间的通信验证类型及密码，同一个VRRP实例中必须一致
        authentication &#123;
                 auth_type PASS
                 auth_pass wolfcode
        &#125;
        #集群资源监控，组合vrrp_script进行
        track_script &#123;
                check_haproxy
        &#125;
        virtual_ipaddress &#123;
                 192.168.10.200   #设置一个虚拟IP进行访问
        &#125;
配置完成后需要刷新keepalived配置并启动：`/opt/nginx/keepalived/sbin/keepalived -s reload`
## [](#配置Selave节点)配置Selave节点
**修改slave节点的 `keepalived.conf` 文件**
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
# 将 keepalived.conf 文件的内容替换为以下内容
! Configuration File for keepalived
global_defs &#123;
   router_id wolfcode     #路由器标志
&#125;
#集群资源监控，组合track_script进行
vrrp_script check_haproxy &#123;
        script &quot;/opt/nginx/keepalived/nginx_check.sh&quot;  #检测nginx状态的脚本路径
        interval 2  #检测时间间隔
        weight -20  #若条件成立，权重减20
&#125;
vrrp_instance PROXY &#123;
        #设置当前主机为备节点，主节点设置为MASTER
        state BACKUP
        #指定HA监测网络接口
        interface ens32
        #虚拟路由标识，同一个VRRP实例要使用同一个标识，主备机
        virtual_router_id 80
        #备节点时，内容为：
        unicast_src_ip 192.168.10.101
        #设置优先级，确保主节点的优先级高过备用节点
        priority 90
        #用于设定主备节点时间同步检查时间间隔
        advert_int 2
        #设置主备节点间的通信验证类型及密码，同一个VRRP实例中必须一致
        authentication &#123;
                 auth_type PASS
                 auth_pass wolfcode
        &#125;
        #集群资源监控，组合vrrp_script进行
        track_script &#123;
                check_haproxy
        &#125;
        virtual_ipaddress &#123;
                 192.168.10.200   #设置一个虚拟IP进行访问
        &#125;
&#125;
配置完成后需要刷新keepalived配置并启动：`/opt/nginx/keepalived/sbin/keepalived -s reload`
# [](#Nginx高可用测试)Nginx高可用测试
**可将master节点关闭，并在selave节点上利用 `ip addr` 命令进行验证，如果出现了配置文件中所配置的虚拟IP，则表示配置成功**
