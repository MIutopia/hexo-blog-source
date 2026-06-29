---
title: "Tomcat 部署实战：从安装到生产环境优化"
date: 2024-06-15 10:00:00
categories: ["基础设施"]
tags: ["Tomcat", "Java", "Web服务", "部署"]
cover: /images/hero/11.jpg
summary: "详细讲解 Tomcat 的安装和生产环境配置，包括性能优化和安全加固。"
---

# Tomcat 部署实战：从安装到生产环境优化

## 目录

- [引言](#引言)
- [一、安装 Tomcat](#一安装-tomcat)
- [二、基础配置](#二基础配置)
- [三、生产环境优化](#三生产环境优化)
- [四、虚拟主机配置](#四虚拟主机配置)
- [五、日志管理](#五日志管理)
- [六、监控和告警](#六监控和告警)
- [总结](#总结)

## 引言

Apache Tomcat 是 Java Web 开发中最流行的 Servlet 容器，广泛用于部署 Java Web 应用。在生产环境中，合理的配置和优化是保证 Tomcat 性能和稳定性的关键。

本文将详细讲解 Tomcat 的安装、配置和优化，帮助你构建高性能的 Java Web 服务。

## 一、安装 Tomcat

### 1.1 环境准备

```bash
# 安装 Java
dnf install java-11-openjdk-devel -y

# 验证 Java
java -version
```

### 1.2 下载安装 Tomcat

```bash
# 创建用户和组
groupadd tomcat
useradd -M -s /bin/nologin -g tomcat tomcat

# 下载 Tomcat
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.17/bin/apache-tomcat-10.1.17.tar.gz

# 解压
tar -xzf apache-tomcat-10.1.17.tar.gz -C /opt/
ln -s /opt/apache-tomcat-10.1.17 /opt/tomcat

# 设置权限
chown -R tomcat:tomcat /opt/tomcat/
chmod +x /opt/tomcat/bin/*.sh
```

### 1.3 创建系统服务

```bash
cat > /etc/systemd/system/tomcat.service << EOF
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start tomcat
systemctl enable tomcat
```

## 二、基础配置

### 2.1 server.xml 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
    <Service name="Catalina">
        <Connector port="8080" protocol="HTTP/1.1"
                   connectionTimeout="20000"
                   redirectPort="8443"
                   maxThreads="200"
                   minSpareThreads="25"
                   maxSpareThreads="75"
                   enableLookups="false"
                   acceptCount="100"
                   disableUploadTimeout="true"/>

        <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
                   maxThreads="200"
                   SSLEnabled="true">
            <SSLHostConfig>
                <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                             type="RSA"/>
            </SSLHostConfig>
        </Connector>

        <Engine name="Catalina" defaultHost="localhost">
            <Host name="localhost"  appBase="webapps"
                  unpackWARs="true" autoDeploy="true">
            </Host>
        </Engine>
    </Service>
</Server>
```

### 2.2 context.xml 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <Resource name="jdbc/MyDB"
              auth="Container"
              type="javax.sql.DataSource"
              maxTotal="100"
              maxIdle="30"
              maxWaitMillis="10000"
              username="dbuser"
              password="dbpass"
              driverClassName="com.mysql.cj.jdbc.Driver"
              url="jdbc:mysql://localhost:3306/mydb"/>
</Context>
```

## 三、生产环境优化

### 3.1 JVM 优化

```bash
# /opt/tomcat/bin/setenv.sh
export JAVA_OPTS="\
  -Xms2g\
  -Xmx2g\
  -XX:MetaspaceSize=256m\
  -XX:MaxMetaspaceSize=512m\
  -XX:+UseG1GC\
  -XX:MaxGCPauseMillis=200\
  -XX:+HeapDumpOnOutOfMemoryError\
  -XX:HeapDumpPath=/var/log/tomcat/heapdump.hprof\
"
```

> **JVM 优化要点**：
> - `-Xms` 和 `-Xmx` 设置为相同值，避免堆内存动态调整
> - 使用 G1GC 垃圾回收器，适合大内存场景
> - 配置堆转储，便于分析内存问题

### 3.2 连接池优化

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           maxThreads="500"
           minSpareThreads="50"
           maxSpareThreads="200"
           acceptCount="100"
           maxConnections="1000"
           keepAliveTimeout="30000"
           maxKeepAliveRequests="100"/>
```

### 3.3 安全加固

```bash
# 删除默认应用
rm -rf /opt/tomcat/webapps/ROOT/
rm -rf /opt/tomcat/webapps/docs/
rm -rf /opt/tomcat/webapps/examples/
rm -rf /opt/tomcat/webapps/host-manager/
rm -rf /opt/tomcat/webapps/manager/

# 配置访问控制
cat > /opt/tomcat/conf/tomcat-users.xml << EOF
<user username="admin" password="secret" roles="manager-gui,admin-gui"/>
EOF

# 限制管理界面访问
cat > /opt/tomcat/webapps/manager/META-INF/context.xml << EOF
<Context antiResourceLocking="false" privileged="true">
    <Valve className="org.apache.catalina.valves.RemoteAddrValve"
           allow="127.0.0.1"/>
</Context>
EOF
```

## 四、虚拟主机配置

```xml
<Engine name="Catalina" defaultHost="localhost">
    <Host name="example.com" appBase="/var/lib/tomcat/example.com" unpackWARs="true" autoDeploy="true">
        <Alias>www.example.com</Alias>
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs"
               prefix="example_access_log"
               suffix=".txt"
               pattern="%h %l %u %t \"%r\" %s %b"/>
    </Host>

    <Host name="blog.com" appBase="/var/lib/tomcat/blog.com" unpackWARs="true" autoDeploy="true">
        <Alias>www.blog.com</Alias>
    </Host>
</Engine>
```

## 五、日志管理

```bash
# 配置日志轮转
cat > /etc/logrotate.d/tomcat << EOF
/opt/tomcat/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 tomcat tomcat
    postrotate
        /bin/kill -USR1 `cat /opt/tomcat/temp/tomcat.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
EOF
```

## 六、监控和告警

```bash
# 启用 JMX
cat >> /opt/tomcat/bin/setenv.sh << EOF
export CATALINA_OPTS="\
  -Dcom.sun.management.jmxremote\
  -Dcom.sun.management.jmxremote.port=9090\
  -Dcom.sun.management.jmxremote.rmi.port=9090\
  -Dcom.sun.management.jmxremote.authenticate=false\
  -Dcom.sun.management.jmxremote.ssl=false\
  -Djava.rmi.server.hostname=10.0.0.10\
"
EOF
```

## 总结

Tomcat 的生产环境配置需要综合考虑多个方面：

1. **JVM 优化**：合理配置堆内存和垃圾回收器
2. **连接池配置**：调整线程池参数适应并发需求
3. **安全加固**：删除默认应用，限制管理界面访问
4. **日志管理**：配置日志轮转和定期清理
5. **监控告警**：启用 JMX 便于监控

通过合理的配置，可以构建高性能、高安全的 Java Web 服务。

---

**参考资料**
- [Tomcat 官方文档](https://tomcat.apache.org/)
- [Tomcat 性能优化指南](https://tomcat.apache.org/tomcat-10.1-doc/performance.html)
