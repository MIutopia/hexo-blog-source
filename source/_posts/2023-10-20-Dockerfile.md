---
title: Dockerfile 详解：从基础到实战
date: 2023-10-20 10:00:00
description: Dockerfile 编写指南，从基础指令到多阶段构建，带你掌握镜像构建的核心技能。
categories:
  - 教程
tags:
  - Docker
  - DevOps
---

## 前言

Dockerfile 是用来构建 Docker 镜像的文本文件，包含了一条条指令，告诉 Docker 如何一步步构建镜像。掌握 Dockerfile 是 Docker 进阶的必经之路。

这篇文章从基础概念讲起，通过实例带你理解 Dockerfile 的常用指令和构建流程。

## Dockerfile 构建流程

一个完整的 Docker 镜像由以下几步构成：

1. 编写 Dockerfile 文件
2. 使用 `docker build` 命令构建镜像
3. 使用 `docker run` 命令运行镜像
4. 使用 `docker push` 发布镜像（推荐 DockerHub、阿里云镜像仓库）

> 参考 [Docker 官方文档](https://docs.docker.com/engine/reference/builder/)

## Dockerfile 指令

Dockerfile 中每个指令都会创建一个新的镜像层。

| 指令 | 说明 |
|------|------|
| `FROM` | 基础镜像，一切从这里开始 |
| `MAINTAINER` | 镜像维护者（姓名+邮箱） |
| `RUN` | 构建镜像时执行的命令 |
| `ADD` | 添加文件并自动解压压缩包 |
| `COPY` | 复制文件到镜像中（类似 ADD，但不解压） |
| `WORKDIR` | 设置工作目录 |
| `VOLUME` | 挂载目录 |
| `EXPOSE` | 暴露端口号 |
| `CMD` | 容器启动时执行的命令（仅最后一个生效，可被替代） |
| `ENTRYPOINT` | 容器启动时执行的命令（可追加参数） |
| `ONBUILD` | 当镜像被继承时触发的指令 |
| `ENV` | 设置环境变量 |

### 注意事项

- 每个指令（关键字）必须**大写**
- 文件从上到下顺序执行
- `#` 表示注释
- 每条指令都会创建并提交一个新的镜像层

## 实战：创建自定义 CentOS 镜像

### 1. 编写 Dockerfile

```dockerfile
# 创建文件：/home/dockerfile/dockerfile.centos
FROM centos:7
MAINTAINER cf<kisssheep1557@163.com>

ENV MYPATH /usr/local/
WORKDIR $MYPATH

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80

CMD echo $MYPATH
CMD echo "-----end-----"
CMD /bin/bash
```

### 2. 构建镜像

```bash
# 命令格式：docker build -f <Dockerfile路径> -t <镜像名>:<tag> .
docker build -f dockerfile.centos -t mycentos:0.1 .
```

### 3. 对比测试

官方基础镜像 `centos:7` 通常没有 vim 等常用命令，而通过 Dockerfile 构建的 `mycentos:0.1` 已经预装了这些工具。

```bash
# 查看镜像构建历史
docker history <镜像ID>
```

## CMD 与 ENTRYPOINT 的区别

这两个指令都用于指定容器启动时执行的命令，但行为不同：

- **CMD**：只有最后一个生效，可以被 `docker run` 的参数替代
- **ENTRYPOINT**：会追加参数，不会被替代

### 测试 CMD

```dockerfile
FROM centos:7
CMD ["ls","-a"]
```

```bash
# 构建镜像
docker build -f mydockerfile-cmd-test -t myimage:tag .

# 运行 - 执行 ls -a
docker run 06f2cc65ea4a

# 尝试追加 -l 参数 - 报错！
docker run 06f2cc65ea4a -l
# 原因：-l 替换了 CMD ["ls","-a"]，而不是追加
```

### 测试 ENTRYPOINT

```dockerfile
FROM centos:7
ENTRYPOINT ["ls","-a"]
```

```bash
# 构建镜像
docker build -f mydockerfile-entrypoint-test -t myimages:tag .

# 运行 - 执行 ls -a
docker run 5184c7d459a0

# 追加 -l 参数 - 成功执行 ls -al！
docker run 5184c7d459a0 -l
# 原因：-l 被追加到 ENTRYPOINT 后面
```

## 实战：制作 Tomcat 镜像

### 1. 准备材料

```bash
# 拉取基础镜像
docker pull centos:7

# 准备目录
cd /home/dockerfile

# 下载 Tomcat 和 JDK 压缩包
# Tomcat: https://tomcat.apache.org/download-80.cgi
# JDK: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
ls
# apache-tomcat-8.5.93.tar.gz  jdk-8u381-linux-x64.tar.gz
```

### 2. 编写 Dockerfile

```dockerfile
FROM centos:7
MAINTAINER yqy<kisssheep1557@163.com>

# 复制说明文件
COPY readme.txt /usr/local/readme.txt

# 添加并解压压缩包
ADD jdk-8u381-linux-x64.tar.gz /usr/local  
ADD apache-tomcat-8.5.93.tar.gz /usr/local

# 安装 vim
RUN yum install -y vim

# 设置工作目录
ENV MYPATH /usr/local
WORKDIR $MYPATH

# 配置环境变量
ENV JAVA_HOME /usr/local/jdk1.8.0_381
ENV CLASS_PATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.93
ENV CATALINA_BASH /usr/local/apache-tomcat-8.5.93
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin

# 暴露端口
EXPOSE 8080 

# 启动 Tomcat 并监控日志
CMD /usr/local/apache-tomcat-8.5.93/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.93/logs/catalina.out
```

### 3. 构建并运行

```bash
# 构建镜像
docker build -t diytomcat .

# 运行容器（简单版）
docker run -d -p 9090:8080 --name diytomcat01 diytomcat:latest

# 运行容器（带挂载）
docker run -d -p 3355:8080 --name yqytomcat \
  -v /home/yqy/build/tomcat/test:/usr/local/apache-tomcat-8.5.93/webapps/test \
  -v /home/yqy/build/tomcat/tomcatlog:/usr/local/apache-tomcat-8.5.93/logs \
  diytomcat
```

访问 `http://服务器IP:9090` 即可看到 Tomcat 默认页面。

## 常见问题

### ❌ 构建时 yum install 失败

**原因**：CentOS 7 官方源已停止维护。

**解决**：更换为 vault 源或阿里云镜像源。

### ❌ CMD 命令不生效

**原因**：多个 CMD 指令只有最后一个生效。

**解决**：使用 `&&` 连接多个命令，或使用 ENTRYPOINT。

---

*以上就是 Dockerfile 的核心内容和实战示例。掌握这些指令，你就可以构建自己的 Docker 镜像了。*
