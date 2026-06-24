---
title: 部署KeepAlived
date: 2024-06-18 00:00:00
categories:
  - 负载均衡
tags:
  - 前端
  - keepalived
---

# [](#安装keepalived所需要的第三方工具)安装keepalived所需要的第三方工具

**本次部署专注于构建基于Nginx和Keepalived的高可用集群，已安装的Nginx相关工具将不再重复安装，仅补充构建Keepalived所需的缺失第三方工具**

1
2
# 安装libnl-3开发库
sudo yum install libnl3-devel libnl3-cli-devel

# [](#下载KeepAlived解压包并解压)下载KeepAlived解压包并解压

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

# [](#安装KeepAlived)安装KeepAlived

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

# [](#验证)验证

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

                
                __END__

            
            
                
    
        ![](/images/User/vavtarsheep.jpg)
    
    
        文章作者：CloudSheep
        

        文章出处：[部署KeepAlived](/2024/06/18/%E9%83%A8%E7%BD%B2KeepAlived/)
        

        作者签名：不啻微芒，造炬成阳
        

        关于主题：[hexo-blog](https://github.com/lsSheep/hexo-blog.git)
        

        版权声明：文章除特别声明外，均采用 [BY-NC-SA](https://creativecommons.org/licenses/by-nc-nd/4.0/) 许可协议，转载请注明出处
        

    
    

                
                    
                        分类：
                        
                            [负载均衡](/category/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/)
                        
                    
                
                
                    
                        标签：
                        
                            [前端](/tag/%E5%89%8D%E7%AB%AF/)
                        
                            [keepalived](/tag/keepalived/)
                        
                    
                
                
                    
                    
                        [» ](/2024/06/18/%E9%83%A8%E7%BD%B2Nginx-KeepAlived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4/) 下一篇：    [部署Nginx+KeepAlived高可用集群](/2024/06/18/%E9%83%A8%E7%BD%B2Nginx-KeepAlived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4/)
