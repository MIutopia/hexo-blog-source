---
title: dockerfile
date: 2023-10-20 00:00:00
categories:
  - 教程
tags:
  - 进阶
  - docker
---

# [](#Dockerfile)Dockerfile

## [](#什么是dockerfile-dockerfile就是用来构建docker镜像的命令)什么是dockerfile?dockerfile就是用来构建docker镜像的命令

**一个完整的docker镜像由以下几步构成：**

**1.编写一个dockerfile文件**

**2.使用docker build命令构建成为一个镜像**

**3.使用docker run命令运行构建好的镜像**

**4.docker push命令发布镜像 （推荐DockerHub、阿里云镜像仓库）**

*注：这里参考[官方文档](https://docs.docker.com/engine/reference/builder/)*

## [](#Dockerfile构建)Dockerfile构建

**构建dockerfile需要注意以下几点：**

**1.每个都要保留关键字（指令）且必须是大写字母！！！**

**2.文件的执行顺序是从上到下的**

**3.#表示注释（如果vim命令那么在虚拟机上是看不见中文的需要手动安装）**

**4.每一个指令都会创建并提交一个新的镜像层，并提交！！**

1
2
3
4
5
注：#vim安装命令
#centos系统
sudo yum install -y vim*
#ubatnu系统
sudo apt install -y vim*

## [](#Dockerfile的命令)Dockerfile的命令

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
FORM              #基础镜像，一切从这里开始
MAINNTATNER       #镜像构建人（姓名+邮箱）
RUN               #镜像构建的时候需要运行的命令
ADD               #步骤：tomcat镜像，这个tomcat压缩包！添加内容
WORKDIR           #镜像的工作目录
VOLUME            #挂载的目录
EXPOSE            #指定暴露的端口号
CMD               #指定这个容器启动的时候要运行的命令，只有最后一个会生效，可被替代
ENTRYOINT         #指定这个容器启动时要运行的命令，可以追加命令
ONBUILD           #当构建一个被继承dockerfile这个时候就会运行ONBUILD的指令。（触发指令）
COPY              #类似ADD，将我们的文件拷贝到镜像中
ENV               #构建的时候设置环境变量

### [](#实例)实例

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
##创建一个自己的centos镜像##
#1.编写dockerfile文件
vi /home/dockerfile/dockerfile.centos  #打开vi编辑器
FROM centos:7
MAINTAINER cf&lt;kisssheep1557@163.com&gt;

ENV MYPATH /usr/local/
WORKDIR $MYPATH

RUN yum -y install vim
RUN yum -y inatall net-tools

EXPOSE 80

CMD echo $MYPATH
CMD echo &quot;-----end-----&quot;
CMD /bin/bash

#2.通过编写好的文件构建镜像
##命令：docker build -f /home/dockerfile/dockerfile文件路径 -t 镜像名：[tag]
docker build -f dockerfile.centos -t mycentos:0.1 .

Successfully built 285c2064af01
Successfully tagged mycentos:0.1

#3.运行测试

![](https://www.z4a.net/images/2023/12/08/docker01.png)

**对比**

官方给的基础镜像centos:7

![](https://www.z4a.net/images/2023/12/08/docker02.png)

自己创建的镜像mycentos:0.1

![](https://www.z4a.net/images/2023/12/08/docker03.png)

*注：一般官方所给出的基础镜像是没有命令的，我们可以在官方给出的基础镜像的基础之上利用dockerfile来进行创建自己所需要的镜像*

1
2
##查看本地进行的变更历史
docker history &lt;镜像ID&gt;

![](https://www.z4a.net/images/2023/12/08/docker04.png)

### [](#CMD和ENTRYPOIN的区别)CMD和ENTRYPOIN的区别

CMD
指定这个容器启动的时候要运行的命令，只有最后一个会生效，可以被代替

ENTRYPOINT
指定这个容器启动的时候要运行的命令，可以追加命令

#### [](#测试CMD)测试CMD

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
#1.编写dockerfile文件
vi /home/dockerfile/mydockerfile-cmd-test

cat /home/dockerfile/mydockerfile-cmd-test
FROM centos:7
CMD [&quot;ls&quot;,&quot;-a&quot;]
#2.构建镜像
docker build -f /home/dockerfile/mydockerfile-cmd-test -t myimage:tag .

#3.run命令运行，发现我们的“ls -a&quot;命令生效、执行

![](https://www.z4a.net/images/2023/12/08/docker05.png)

1
2
3
4
5
#4.追加一个&quot;-l&quot;命令，构成&quot;ls -al&quot;命令，发现报错
[root@docker ~]# docker run 06f2cc65ea4a -l
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: &quot;-l&quot;: executable file not found in $PATH: unknown.
ERRO[0000] error waiting for container:
##原因:在CMD命令的情况下，&quot;-l&quot;替换了CMD[&quot;ls&quot;,&quot;a&quot;]命令而不是追加在里面，&quot;-l&quot;不是命令，所以报错！

#### [](#测试ENTRYPOINT)测试ENTRYPOINT

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
#1.编写dockerfile文件
vi /home/dockerfile/mydockerfile-entrypoint-test

cat /home/dockerfile/mydockerfile-entrypoint-test
FORM centos:7
ENTRYPOINT [&quot;ls&quot;,&quot;-a&quot;]

#2.利用写好的dockerfile文件构建镜像
docker build -f /home/dockerfile/mydockerfile-cmd-test -t myimages:tag .

#3.run命令运行构建好的镜像，发现写入的&quot;ls -a&quot;命令生效、执行

![](https://www.z4a.net/images/2023/12/08/docker06.png)

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
#4.追加一个&quot;l&quot;命令，构成&quot;ls -al&quot;命令，发现命令生效并且被成功执行
[root@docker ~]# docker run 5184c7d459a0 -l
total 12
drwxr-xr-x.   1 root root     6 Oct  4 08:30 .
drwxr-xr-x.   1 root root     6 Oct  4 08:30 ..
-rwxr-xr-x.   1 root root     0 Oct  4 08:30 .dockerenv
-rw-r--r--.   1 root root 12114 Nov 13  2020 anaconda-post.log
lrwxrwxrwx.   1 root root     7 Nov 13  2020 bin -&gt; usr/bin
drwxr-xr-x.   5 root root   340 Oct  4 08:30 dev
drwxr-xr-x.   1 root root    66 Oct  4 08:30 etc
drwxr-xr-x.   2 root root     6 Apr 11  2018 home
lrwxrwxrwx.   1 root root     7 Nov 13  2020 lib -&gt; usr/lib
lrwxrwxrwx.   1 root root     9 Nov 13  2020 lib64 -&gt; usr/lib64
drwxr-xr-x.   2 root root     6 Apr 11  2018 media
drwxr-xr-x.   2 root root     6 Apr 11  2018 mnt
drwxr-xr-x.   2 root root     6 Apr 11  2018 opt
dr-xr-xr-x. 131 root root     0 Oct  4 08:30 proc
dr-xr-x---.   2 root root   114 Nov 13  2020 root
drwxr-xr-x.  11 root root   148 Nov 13  2020 run
lrwxrwxrwx.   1 root root     8 Nov 13  2020 sbin -&gt; usr/sbin
drwxr-xr-x.   2 root root     6 Apr 11  2018 srv
dr-xr-xr-x.  13 root root     0 Oct  4 08:13 sys
drwxrwxrwt.   7 root root   132 Nov 13  2020 tmp
drwxr-xr-x.  13 root root   155 Nov 13  2020 usr
drwxr-xr-x.  18 root root   238 Nov 13  2020 var

##原因：ENTRYPOINT命令的情况下，&quot;-l&quot;追加在ENTRYPOINT [&quot;1s&quot;，&quot;-a&quot;]命令后面，得到&quot;ls -al&quot;的命令，所以命令正常执行！（我们的追加命令，是直接拼接在我们的ENTRYPOINT命令的后面）

## [](#实战：制作tomcat镜像)实战：制作tomcat镜像

### [](#1-准备好centos镜像，tomcat和jdk的压缩包-tomcat和jdk的压缩包安装地址放在下面)1.准备好centos镜像，tomcat和jdk的压缩包(tomcat和jdk的压缩包安装地址放在下面)

1
2
3
4
5
6
docker pull centos:7   #下载centos基础镜像

cd /home/dockerfile  #上传压缩包至该目录
[root@docker dockerfile]# ls
apache-tomcat-8.5.93.tar.gz  jdk-8u381-linux-x64.tar.gz

*注：[tomcat下载地址](https://tomcat.apache.org/download-80.cgi)和 [jdk下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)*

### [](#2-解压，创建dockerfile文件)2.解压，创建dockerfile文件

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
touch readme.txt   #创建readme.txt文件(可以理解为帮助文档)

vi Dockerfile   #官方命名,build命令会自动查找该路径,不需要-f来确定路径
##编写dockerfile文件##
FROM centos:7
MAINTAINER yqy&lt;kisssheep1557@163.com&gt; #构建人信息
#将刚才创建的readme文件复制到该目录下
COPY readme.txt /usr/local/readme.txt
#ADD会自动解压压缩包
ADD jdk-8u381-linux-x64.tar.gz /usr/local  
ADD apache-tomcat-8.5.93.tar.gz /usr/local
#执行添加命令
RUN yum install -y vim
#工作目录
ENV MYPATH /usr/local
WORKDIR $MYPATH
#配置tomcat的运行环境
ENV JAVA_HOME /usr/local/jdk1.8.0_381
ENV CLASS_PATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.93
ENV CATALINA_BASH /usr/local/apache-tomcat-8.5.93
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
#暴露端口
EXPOSE 8080 
#启动后监视日志输出
CMD /usr/local/apache-tomcat-8.5.93/bin/startup.sh &amp;&amp; tail -F /usr/local/apa
che-tomcat-8.5.93/bin/logs/catalina.out

### [](#3-构建镜像，启动，访问测试)3.构建镜像，启动，访问测试

1
2
3
4
5
6
#构建镜像
docker build -t diytomcat .
#启动镜像
docker run -d -p 9090:8080 --name diytomcat01 diytomcat:latest

docker run -d -p 3355:8080 --name yqytomcat -v /home/yqy/build/tomcat/test:/usr/local/apache-tomcat-8.5.93/webapps/test -v /home/yqy/build/tomcat/tomcatlog:/usr/local/apache-tomcat-8.5.93/logs diytomcat   #复杂版

![](https://www.z4a.net/images/2023/12/08/docker07.png)

**访问成功！！！**

### [](#4-发布项目)4.发布项目

1
2
3
#在使用run命令打开容器时做了挂载卷，所以直接在本地test文件下编写项目就可以了
mkdir /home/yqy/build/tomcat/test/WEB-INF
vi /home/yqy/build/tomcat/test/WEB-INF/web.xml

1
2
3
4
5
6
&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;
&lt;web-app xmlns=&quot;http://java.sun.com/xml/ns/javaee&quot;
     xmlns:xsi=&quot;http://www.w3.org/2001/XMLSchema-instance&quot;
     xsi:schemaLocation=&quot;http://java.sun.com/xml/ns/javaee  http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd&quot;
     version=&quot;2.5&quot;&gt;
&lt;/web-app&gt;

1
vi /home/sywl/build/tomcat/test/index.jsp

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
&lt;%@ page language=&quot;java&quot; contentType=&quot;text/html; charset=UTF-8&quot;
 pageEncoding=&quot;UTF-8&quot;%&gt;
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;meta charset=&quot;utf-8&quot;&gt;
&lt;title&gt;hello,sywl&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
Hello World!&lt;br/&gt;
&lt;%
System.out.println(&quot;----my web test----&quot;);
%&gt;
&lt;/body&gt;
&lt;/html&gt;

                
