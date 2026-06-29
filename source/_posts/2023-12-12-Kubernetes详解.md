---
title: Kubernetes详解
date: 2023-12-12 00:00:00
categories:
  - 教程
tags:
  - 运维
  - Kubernetes
---

# [](#pod)pod

## [](#概述)概述

**Pod 是一组（一个或多个）容器的集合（Pod 就像是豌豆荚，而容器就像是豌豆荚中的豌豆）。这些容器共享存储、网络等。**

## [](#pod资源清单)pod资源清单

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
54
55
apiVersion: v1 #api文档版本
kind: Pod #资源对象类型
metadata: #pod相关的元数据
  name: nginx-pod #pod的名称
  labels: #定义Pod的标签
    tybe: app #自定义labe标签，名字tybe,值为app
    test: 1.0.0 #自定义labe标签，tybe版本号
  namespace: &#x27;default&#x27; #命名空间的配置
spec: #期望pod按照这里的配置进行描述
  containers: #对pod中的容器描述
  - name: nginx #容器的名称
    image: &#x27;nginx:1.20.2&#x27; #指定容器的镜像
    imagePullPolicy: IfNotPresent #镜像拉取策略
    startupProbe:  #应用启动探针配置
      exec:
        command:
        - /bin/sh
        - -c
        - &quot;sleep 3;echo success &gt; /inited&quot;
      initialDelaySeconds: 1 #探测延迟，running后指定20秒后才执行探测，默认0秒
      periodSeconds: 5 #执行探测的间隔时间
      failureThreshold: 3 #失败阈值，失败多少次才算是真正失败
      successThreshold: 1 #成功阈值多少次监测成功才算成功
      timeoutSeconds: 5 #请求的超时时间
    readinessProbe: #应用存活探针配置
      httpGet: #探测方式，基于http请求探测
        path: /started.html #http请求路径
     #tcpSocket:
        port: 80 #请求端口
      initialDelaySeconds: 1 #探测延迟，running后指定20秒后才执行探测，默认0秒
      periodSeconds: 5 #执行探测的间隔时间
      failureThreshold: 3 #失败阈值，失败多少次才算是真正失败
      successThreshold: 1 #成功阈值多少次监测成功才算成功
      timeoutSeconds: 5 #请求的超时时间
    command: #指定容器启动时执行的命令
    - nginx
    - -g
    - &#x27;daemon off;&#x27; #nginx -g &#x27;daemon off;&#x27;
    workingDir: /usr/share/nginx/html #定义容器启动后的工作目录
    ports:
    - name: http #端口名称
      containerPort: 80 #描述容器内要暴露什么端口
      protocol: TCP #描述该端口是基于哪种协议通信的
    env: #环境变量
    - name: JVM_OPTS #环境变量名称
      value: &#x27;-Xms128m -Xmx128m&#x27; #环境变量的值
    resources:
      requests: #最少需要多少资源
        cpu: 100m
        memory: 128Mi
      limits: #最多可以使用多少资源
        cpu: 200m
        memory: 256Mi
  restartPolicy: OnFailure #重启策略

# [](#控制器)控制器

## [](#deployment)deployment

![](C:\Users\32625\OneDrive\图片\k8s\24.png)

### [](#配置文件)配置文件

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
apiVersion: apps/v1 # 版本号 
kind: Deployment # 类型 
metadata: # 元数据 
  name: nginx-deploy # rs名称
  labels: #标签 
    controller: deploy 
spec: # 详情描述 
  replicas: 3 # 副本数量 
  revisionHistoryLimit: 5 # 保留历史版本，默认为10 
  paused: false # 暂停部署，默认是false 
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600 
  strategy: # 策略 
    type: RollingUpdate # 滚动更新策略 
    rollingUpdate: # 滚动更新 
      maxSurge: 30% # 最大额外可以存在的副本数
  selector: # 选择器，通过它指定该控制器管理哪些pod 
    matchLabels: # Labels匹配规则 
      app: nginx-pod 
    matchExpressions: # Expressions匹配规则 
      - &#123;key: app, operator: In, values: [nginx-pod]&#125; 
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本 
    metadata: 
      labels: 
        app: nginx-pod 
    spec: 
      containers: 
      - name: nginx 
        image: nginx:1.20.2 
        ports: 
        - containerPort: 80

### [](#版本回退)版本回退

kubetl rollout 参数 deploy xx  # 支持下面的选择

status 显示当前升级的状态

history 显示升级历史记录

pause 暂停版本升级过程

resume 继续已经暂停的版本升级过程

undo 回滚到上一级版本 （可以使用–to-revision回滚到指定的版本）

### [](#滚动更新)滚动更新

- kubectl set image命令

1
kubectl set image deployment nginx-deploy nginx=nginx:1.20.2

直接修改yaml文件

### [](#金丝雀发布)金丝雀发布

金丝雀的发布流程如下：

① 将 service 的标签设置为 app&#x3D;nginx ，这就意味着集群中的所有标签是 app&#x3D;nginx 的 Pod 都会加入负载均衡网络。

② 使用 Deployment v&#x3D;v1 app&#x3D;nginx 去筛选标签是 app&#x3D;nginx 以及 v&#x3D;v1 的 所有 Pod。

③ 同理，使用 Deployment v&#x3D;v2 app&#x3D;nginx 去筛选标签是 app&#x3D;nginx 以及 v&#x3D;v2 的所有 Pod 。

④ 逐渐加大 Deployment v&#x3D;v2 app&#x3D;nginx 控制的 Pod 的数量，根据轮询负载均衡网络的特点，必定会使得此 Deployment 控制的 Pod 的流量增大。

⑤ 当测试完成后，删除 Deployment v&#x3D;v1 app&#x3D;nginx 即可。

## [](#HPA)HPA

![](C:\Users\32625\OneDrive\图片\k8s\50.png)

根据cpu使用率或自定义指标(metrics)自动对pod进行扩&#x2F;缩容

### [](#监控cpu使用率)监控cpu使用率

#### [](#创建deployment和service)创建deployment和service

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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.1
          ports:
            - containerPort: 80
          resources: # 资源限制
            requests:
              cpu: &quot;100m&quot; # 100m 表示100 milli cpu，即 0.1 个CPU
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80 # svc 的访问端口
      name: nginx
      targetPort: 80 # Pod 的访问端口
      protocol: TCP
      nodePort: 30010 # 在机器上开端口，浏览器访问

#### [](#创建HPA)创建HPA

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
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: k8s-hpa
spec:
  minReplicas: 1 # 最小 Pod 数量
  maxReplicas: 10 # 最大 Pod 数量
  targetCPUUtilizationPercentage: 3 # CPU 使用率指标，即 CPU 超过 3%（Pod 的 limit 的 cpu ） 就进行扩容
  scaleTargetRef:  # 指定要控制的Nginx的信息
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy

## [](#扩缩容)扩缩容

- kubectl scale 命令

1
kubectl scale deployment nginx-deploy --replicas=4

## [](#StatefulSet)StatefulSet

deployment部署方式
无状态应用：网络可能会变（IP 地址）、存储可能会变（卷）、顺序可能会变（启动的顺序）。应用场景：业务代码，如：使用 SpringBoot 开发的商城应用的业务代码。无状态的：

statefulset部署方式
有状态应用：网络不变、存储不变、顺序不变。应用场景：中间件，如：MySQL 集群、Zookeeper 集群、Redis 集群、MQ 集群。

![](C:\Users\32625\OneDrive\图片\k8s\56.png)

### [](#配置文件-1)配置文件

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
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-nginx
  namespace: default
spec:
  selector:
    matchLabels:
      app: ss-nginx
  serviceName: nginx-svc # 服务名称，这个字段需要和 service 的 metadata.name 相同
  replicas: 5 # 副本数
  template:
    metadata:
      labels:
        app: ss-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.20.2
---
# 将 StatefulSet 加入网络
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  selector:
    app: ss-nginx
  type: ClusterIP
  clusterIP: None # 不分配 ClusterIP ，形成无头服务，整个集群的 Pod 能直接访问，但是浏览器不可以访问。
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80

### [](#分区更新)分区更新

● StatefulSet 的更新策略：
  ○ OnDelete：删除之后才更新。
  ○ RollingUpdate：滚动更新，如果是此更新策略，可以设置更新的索引（默认值）。

#### [](#示例：)示例：

1
2
3
4
5
6
7
...
spec:
  updateStrategy: # 更新策略
    type: RollingUpdate # OnDelete 删除之后才更新；RollingUpdate 滚动更新
    rollingUpdate:
      partition: 0 # 更新索引 &gt;= partition 的 Pod ，默认为 0
...

### [](#管理策略)管理策略

StatefulSet 的 Pod 的管理策略（podManagementPolicy）分为：

OrderedReady（有序启动，默认值）
Parallel（并发一起启动）

1
2
3
4
...
spec:
  podManagementPolicy: OrderedReady # 控制 Pod 创建、升级以及扩缩容逻辑。Parallel（并发一起启动） 和 
...

### [](#级联删除与非级联删除)级联删除与非级联删除

#### [](#级联删除：)级联删除：

1
2
# 删除statefulset时同时删除pods
kubectl delete sts sts-nginx

#### [](#非级联删除：)非级联删除：

1
2
# 删除statefulset时不会删除pods，sts删除后po依然存在
kubectl delete sts sts-nginx --cascade=false

## [](#DdemoSet)DdemoSet

![](C:\Users\32625\OneDrive\图片\k8s\54.png)

● DaemonSet 控制器确保所有（或一部分）的节点都运行了一个指定的 Pod 副本。
  ○ 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上。
  ○ 当节点从集群中移除时，Pod 也就被垃圾回收了。
  ○ 删除一个 DaemonSet 可以清理所有由其创建的 Pod

### [](#daemonset资源清单)daemonset资源清单

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
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型
metadata: # 元数据
  name: # 名称
  namespace: #命名空间
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的Pod的最大值，可用为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理那些Pod
    matchLabels: # Labels匹配规则 matchLabels 和 matchExpressions 任选其一
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - key: app
        operator: In
        values:
          - nginx-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
     metadata:
       labels:
         app: nginx-pod
     spec:
       containers:
         - name: nginx
           image: nginx:1.17.1
           ports:
             - containerPort: 80

### [](#配置文件-2)配置文件

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
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds
  namespace: default
  labels:
    app: ds
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      # tolerations: # 污点
      # - key: node-role.kubernetes.io/master
      #   effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.20.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
      terminationGracePeriodSeconds: 30
      volumes:
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai

## [](#Job)Job

Job 主要用于负责批量处理短暂的一次性任务。

Job 的特点： 

- 当 Job 创建的 Pod 执行成功结束时，Job 将记录成功结束的 Pod 数量。

- 当成功结束的 Pod 达到指定的数量时，Job 将完成执行。

- 删除 Job 对象时，将清理掉由 Job 创建的 Pod 。

- Job 可以保证指定数量的 Pod 执行完成。

![](C:\Users\32625\OneDrive\图片\k8s\60.png)

### [](#job资源清单)job资源清单

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
apiVersion: batch/v1 # 版本号
kind: Job # 类型
metadata: # 元数据
  name:  # 名称
  namespace:  #命名空间
  labels: # 标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
  parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
  activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
  backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
  manualSelector: true # 是否可以使用selector选择器选择Pod，默认为false
  ttlSecondsAfterFinished: 0 # 如果是 0 表示执行完Job 时马上删除。如果是 100 ，就是执行完 Job ，等待 100s 后删除。TTL 机制由 TTL 控制器 提供，ttlSecondsAfterFinished 字段可激活该特性。当 TTL 控制器清理 Job 时，TTL 控制器将删除 Job 对象，以及由该 Job 创建的所 有 Pod 对象。
  selector: # 选择器，通过它指定该控制器管理那些Pod，非必须字段
    matchLabels: # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - key: app
        operator: In
        values:
          - counter-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
     metadata:
       labels:
         app: counter-pod
     spec:
       restartPolicy: Never # 重启策略只能设置为Never或OnFailure
       containers:
         - name: counter
           image: busybox:1.30
           command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done&quot;]

● 关于模板中的重启策略的说明：
  ○  如果设置为OnFailure，则Job会在Pod出现故障的时候重启容器，而不是创建Pod，failed次数不变。
  ○  如果设置为Never，则Job会在Pod出现故障的时候创建新的Pod，并且故障Pod不会消失，也不会重启，failed次数+1。
  ○  如果指定为Always的话，就意味着一直重启，意味着Pod任务会重复执行，这和Job的定义冲突，所以不能设置为Always。

### [](#CronJob)CronJob

![](C:\Users\32625\OneDrive\图片\k8s\63.png)

- CronJob 控制器以 Job 控制器为其管控对象，并借助它管理 Pod 资源对象，Job 控制器定义的作业任务在其控制器资源创建之后便会立即执行，但 CronJob 可以以类似 Linux 操作系统的周期性任务作业计划的方式控制器运行时间点及重复运行的方式，换言之，CronJob 可以在特定的时间点反复去执行 Job 任务。

#### [](#资源清单)资源清单

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
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型
metadata: # 元数据
  name:  # 名称
  namespace:  #命名空间
  labels:
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点，用于控制任务任务时间执行
  concurrencyPolicy: # 并发执行策略
  failedJobsHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobsHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象，下面其实就是job的定义
    metadata: &#123;&#125;
    spec:
      completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
      parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
      activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
      backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
      template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
        spec:
          restartPolicy: Never # 重启策略只能设置为Never或OnFailure
          containers:
            - name: counter
              image: busybox:1.30
              command: [ &quot;/bin/sh&quot;,&quot;-c&quot;,&quot;for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done&quot; ]

●  schedule：cron表达式，用于指定任务的执行时间。
  ○  *&#x2F;1 * * * *：表示分钟  小时  日  月份  星期。
  ○  分钟的值从 0 到 59 。
  ○  小时的值从 0 到 23 。
  ○  日的值从 1 到 31 。
  ○  月的值从 1 到 12 。
  ○  星期的值从 0 到 6，0 表示星期日。
  ○  多个时间可以用逗号隔开，范围可以用连字符给出：* 可以作为通配符，&#x2F; 表示每…
●  concurrencyPolicy：并发执行策略
  ○  Allow：运行 Job 并发运行（默认）。
  ○  Forbid：禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行。
  ○  Replace：替换，取消当前正在运行的作业并使用新作业替换它。

# [](#网络)网络

## [](#Service)Service

### [](#概述-1)概述

**在 Kubernetes 中，Pod 是应用程序的载体，我们可以通过 Pod 的 IP 来访问应用程序，但是 Pod 的 IP 地址不是固定的，这就意味着不方便直接采用 Pod 的 IP 对服务进行访问。** 

**为了解决这个问题，Kubernetes 提供了 Service 资源，Service 会对提供同一个服务的多个 Pod 进行聚合，并且提供一个统一的入口地址，通过访问 Service 的入口地址就能访问到后面的 Pod 服务。**

![](C:\Users\32625\OneDrive\图片\k8s\2.png)

### [](#OSI-网络模型和-TCP-IP-协议：)OSI 网络模型和 TCP&#x2F;IP 协议：

![](C:\Users\32625\OneDrive\图片\k8s\OSI 网络模型和 TCP-IP 协议.png)

- **Service 默认使用的协议是 TCP 协议，对应的是 OSI 网络模型中的第四层传输层，所以也有人称 Service 为四层网络负载均衡。**

### [](#service资源清单)service资源清单

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
apiVersion: v1 # 版本
kind: Service # 类型
metadata: # 元数据
  name: # 资源名称
  namespace: # 命名空间
spec:
  selector: # 标签选择器，用于确定当前Service代理那些Pod
    app: nginx
  type: NodePort # Service的类型，指定Service的访问方式
  clusterIP: # 虚拟服务的IP地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项，默认值为None
  ports: # 端口信息
    - port: 8080 # Service端口
      protocol: TCP # 协议
      targetPort : # Pod端口
      nodePort:  # 主机端口

### [](#配置文件-3)配置文件

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
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
  labels:
    app: nginx-svc
spec:
  ports:
  - name: http # service 端口配置的名称
    protocol: TCP # 端口绑定的协议，支持 TCP、UDP、SCTP，默认为 TCP
    port: 80 # service 自己的端口
    targetPort: 80 # 目标 pod 的端口
  selector: # 选中当前 service 匹配哪些 pod，对哪些 pod 的东西流量进行代理
    app: nginx
  type: NodePort 随机生成端口

#### [](#type说明：)type说明：

ClusterIP（默认值）：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。

NodePort：通过每个节点上的 IP 和静态端口（NodePort）暴露服务。 NodePort 服务会路由到自动创建的 ClusterIP 服务。 通过请求 `&lt;节点 IP&gt;:&lt;节点端口&gt;`，我们可以从集群的外部访问一个 NodePort 服务。

LoadBalancer：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。

ExternalName：通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容（如：foo.bar.example.com）。 无需创建任何类型代理。

### [](#命令操作：)命令操作：

1
2
3
4
5
#使用其他Pod访问创建的svcvice name
kubectl exec -it nginx-pod -- sh #进入创建的pod中去
curl/wegt http://nginx-svc:80
#如果需要跨namespace(命名空间)访问pod，则在service name后面加上&lt;namespec&gt;即可
curl http://nginx-svc.default

### [](#实现外部访问)实现外部访问

#### [](#实现方式)实现方式

编写service配置文件时，不指定selector属性

自己创建endpoint(ds)

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
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80 # 此时的 targetPort 需要和 Endpoints 的subsets.addresses.ports.port 相同  
---
apiVersion: v1
kind: Endpoints
metadata:
  name: nginx-svc # 此处的 name 需要和 Service 的 metadata 的 name 相同
  namespace: default
subsets:
- addresses:
  - ip: 220.181.38.251 # 百度
  - ip: 221.122.82.30 # 搜狗 
  - ip: 61.129.7.47 # QQ
  ports:
  - name: http # 此处的 name 需要和 Service 的 spec.ports.name 相同
    port: 80
    protocol: TCP域名解析

#### [](#域名解析)域名解析

1
2
3
4
5
6
7
进入pod使用nslookup查询
kubectl exec -it nginx-svc -- sh
# 安装nslookup
apt-get update
apt-get install dnsutils
#其中 nginx-svc 是 Service 的服务名
nslookup -type=a nginx-svc

#### [](#保持会话技术)保持会话技术

- 基于客户端 IP 地址的会话保持模式，即来自同一个客户端发起的所有请求都尽最大可能转发到固定的一个 Pod 上，只需要在 spec 中添加 `sessionAffinity: ClientIP` 选项。

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
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  selector:
    app: nginx
  type: ClusterIP 
  sessionAffinity: &quot;ClientIP&quot;
  sessionAffinityConfig:
    clientIP: 
       timeoutSeconds: 11800 # 30 分钟
  ports:
  - name: nginx
    protocol: TCP
    port: 80 
    targetPort: 80
   

### [](#网络层次)网络层次

![](C:\Users\32625\OneDrive\图片\k8s\14.png)

### [](#Pod-指定自己的主机名)Pod 指定自己的主机名

**Pod 中还可以设置 hostname 和 subdomain 字段，需要注意的是，一旦设置了 hostname ，那么该 Pod 的主机名就被设置为 hostname 的值，而 subdomain  需要和 svc 中的 name 相同。** 

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
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
  namespace: default
spec:
  selector:
    app: nginx
  type: ClusterIP
  clusterIP: None 
  ports:
  - name: foo
    protocol: TCP
    port: 1234
    targetPort: 1234

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  namespace: default
  labels:
    app: nginx
spec:
  hostname: nginx1
  subdomain: default-subdomain # 必须和 svc 的 name 相同，才可以使用 hostname.subdomain.名称空间.svc.cluster.local 访问，否则访问不到指定的 Pod 。
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 250m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort:  80
      name:  nginx
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
  namespace: default
  labels:
    app: nginx
spec:
  hostname: nginx2
  subdomain: default-subdomain # 必须和 svc 的 name 相同，才可以使用 hostname.subdomain.名称空间.svc.cluster.local 访问，否则访问不到指定的 Pod 。
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 250m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort:  80
      name:  nginx
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

## [](#Ingress)Ingress

**Service 可以使用 NodePort 暴露集群外访问端口，但是性能低、不安全并且端口的范围有限。**

**Service 缺少七层（OSI 网络模型）的统一访问入口，负载均衡能力很低，不能做限流、验证非法用户、链路追踪等等。**

**Ingress 公开了从集群外部到集群内 `服务` 的 HTTP 和 HTTPS 路由。流量路由由 Ingress 资源上定义的规则控制。**

**我们使用 Ingress 作为整个集群统一的入口，配置 Ingress 规则转发到对应的 Service** 。

![](C:\Users\32625\OneDrive\图片\k8s\22.png)

### [](#安装Ingress-nginx)安装Ingress-nginx

1
2
# 安装命令：
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/baremetal/deploy.yaml

#### [](#修改deploy-yaml配置文件)修改deploy.yaml配置文件

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
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183
184
185
186
187
188
189
190
191
192
193
194
195
196
197
198
199
200
201
202
203
204
205
206
207
208
209
210
211
212
213
214
215
216
217
218
219
220
221
222
223
224
225
226
227
228
229
230
231
232
233
234
235
236
237
238
239
240
241
242
243
244
245
246
247
248
249
250
251
252
253
254
255
256
257
258
259
260
261
262
263
264
265
266
267
268
269
270
271
272
273
274
275
276
277
278
279
280
281
282
283
284
285
286
287
288
289
290
291
292
293
294
295
296
297
298
299
300
301
302
303
304
305
306
307
308
309
310
311
312
313
314
315
316
317
318
319
320
321
322
323
324
325
326
327
328
329
330
331
332
333
334
335
336
337
338
339
340
341
342
343
344
345
346
347
348
349
350
351
352
353
354
355
356
357
358
359
360
361
362
363
364
365
366
367
368
369
370
371
372
373
374
375
376
377
378
379
380
381
382
383
384
385
386
387
388
389
390
391
392
393
394
395
396
397
398
399
400
401
402
403
404
405
406
407
408
409
410
411
412
413
414
415
416
417
418
419
420
421
422
423
424
425
426
427
428
429
430
431
432
433
434
435
436
437
438
439
440
441
442
443
444
445
446
447
448
449
450
451
452
453
454
455
456
457
458
459
460
461
462
463
464
465
466
467
468
469
470
471
472
473
474
475
476
477
478
479
480
481
482
483
484
485
486
487
488
489
490
491
492
493
494
495
496
497
498
499
500
501
502
503
504
505
506
507
508
509
510
511
512
513
514
515
516
517
518
519
520
521
522
523
524
525
526
527
528
529
530
531
532
533
534
535
536
537
538
539
540
541
542
543
544
545
546
547
548
549
550
551
552
553
554
555
556
557
558
559
560
561
562
563
564
565
566
567
568
569
570
571
572
573
574
575
576
577
578
579
580
581
582
583
584
585
586
587
588
589
590
591
592
593
594
595
596
597
598
599
600
601
602
603
604
605
606
607
608
609
610
611
612
613
614
615
616
617
618
619
620
621
622
623
624
625
626
627
628
629
630
631
632
633
634
635
636
637
638
639
640
641
642
643
644
645
646
647
648
649
650
651
652
653
654
655
656
657
658
659
660
661
662
663
664
665
666
667
668
669
670
671
672
673
674
675
676
677
678
679
680
681
682
683
684
685
686
#GENERATED FOR K8S 1.20
apiVersion: v1
kind: Namespace
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx
---
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx
  namespace: ingress-nginx
rules:
- apiGroups:
  - &quot;&quot;
  resources:
  - namespaces
  verbs:
  - get
- apiGroups:
  - &quot;&quot;
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - &quot;&quot;
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - &quot;&quot;
  resourceNames:
  - ingress-controller-leader
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - &quot;&quot;
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - &quot;&quot;
  resources:
  - events
  verbs:
  - create
  - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
  namespace: ingress-nginx
rules:
- apiGroups:
  - &quot;&quot;
  resources:
  - secrets
  verbs:
  - get
  - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx
rules:
- apiGroups:
  - &quot;&quot;
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - &quot;&quot;
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - &quot;&quot;
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - &quot;&quot;
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
rules:
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - validatingwebhookconfigurations
  verbs:
  - get
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: v1
data:
  allow-snippet-annotations: &quot;true&quot;
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-controller
  namespace: ingress-nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: ClusterIP # NodePort
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  ports:
  - appProtocol: https
    name: https-webhook
    port: 443
    targetPort: webhook
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: ClusterIP
---
apiVersion: apps/v1
kind: DaemonSet # Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      dnsPolicy: ClusterFirstWithHostNet # dns 调整为主机网络 ，原先为 ClusterFirst
      hostNetwork: true # 直接让 nginx 占用本机的 80 和 443 端口，这样就可以使用主机网络
      containers:
      - args:
        - /nginx-ingress-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --report-node-internal-ip-address=true
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.1.2 # 修改 k8s.gcr.io/ingress-nginx/controller:v1.1.2@sha256:28b11ce69e57843de44e3db6413e98d09de0f6688e33d4bd384002a44f78405c
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources: # 资源限制
          requests:
            cpu: 100m
            memory: 90Mi
          limits: 
            cpu: 500m
            memory: 500Mi     
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      nodeSelector:
        node-role: ingress # 以后只需要给某个 node 打上这个标签就可以部署 ingress-nginx 到这个节点上了
        # kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission-create
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.1.2
        helm.sh/chart: ingress-nginx-4.0.18
      name: ingress-nginx-admission-create
    spec:
      containers:
      - args:
        - create
        - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=ingress-nginx-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1 # k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
        imagePullPolicy: IfNotPresent
        name: create
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission-patch
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.1.2
        helm.sh/chart: ingress-nginx-4.0.18
      name: ingress-nginx-admission-patch
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=ingress-nginx-admission
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=ingress-nginx-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1 # k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
        imagePullPolicy: IfNotPresent
        name: patch
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.1.2
    helm.sh/chart: ingress-nginx-4.0.18
  name: ingress-nginx-admission
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: ingress-nginx-controller-admission
      namespace: ingress-nginx
      path: /networking/v1/ingresses
  failurePolicy: Fail
  matchPolicy: Equivalent
  name: validate.nginx.ingress.kubernetes.io
  rules:
  - apiGroups:
    - networking.k8s.io
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    resources:
    - ingresses
  sideEffects: None

#### [](#给node节点打上标签)给node节点打上标签

1
kubectl label node &lt;node-name&gt; node-role=ingress

#### [](#验证)验证

1
2
netstat -ntlp | grep 80
netstat -ntlp | grep 443

![](C:\Users\32625\OneDrive\图片\k8s\30.png)

### [](#ingress配置文件)ingress配置文件

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
apiVersion: networking.k8s.io/v1
kind: Ingress # 资源类型为ingress
metadata:
  name: ingress-http
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: &quot;nginx&quot;
    nginx.ingress.kubernetes.io/backend-protocol: &quot;HTTP&quot;  
spec:
  rules: # 规则
  - host: nginx.xudaxian.com # 指定的监听的主机域名，相当于 nginx.conf 的 server &#123; xxx &#125;
    http: # 指定路由规则
      paths:
      - path: /
        pathType: Prefix # 匹配规则，Prefix 前缀匹配 nginx.xudaxian.com/* 都可以匹配到
        backend: # 指定路由的后台服务的 service 名称
          service:
            name: nginx-svc # 服务名
            port:
              number: 80 # 服务的端口
  - host: tomcat.xudaxian.com # 指定的监听的主机域名，相当于 nginx.conf 的 server &#123; xxx &#125;
    http: # 指定路由规则
      paths:
      - path: /
        pathType: Prefix # 匹配规则，Prefix 前缀匹配 tomcat.xudaxian.com/* 都可以匹配到
        backend: # 指定路由的后台服务的 service 名称
          service:
            name: tomcat-svc # 服务名
            port:
              number: 8080 # 服务的端口

#### [](#pathType-说明：)**[pathType](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#path-types) 说明：**

refix：基于以 `/` 分隔的 URL 路径前缀匹配。匹配区分大小写，并且对路径中的元素逐个完成。 路径元素指的是由 `/` 分隔符分隔的路径中的标签列表。 如果每个 p 都是请求路径 p 的元素前缀，则请求与路径 p 匹配。

Exact：精确匹配 URL 路径，且区分大小写。

ImplementationSpecific：对于这种路径类型，匹配方法取决于 IngressClass。 具体实现可以将其作为单独的 pathType 处理或者与 Prefix 或 Exact 类型作相同处理。

类型
路径
请求路径
匹配与否？

Prefix
`/`
（所有路径）
是

Exact
`/foo`
`/foo`
是

Exact
`/foo`
`/bar`
否

Exact
`/foo`
`/foo/`
否

Exact
`/foo/`
`/foo`
否

Prefix
`/foo`
`/foo`, `/foo/`
是

Prefix
`/foo/`
`/foo`, `/foo/`
是

Prefix
`/aaa/bb`
`/aaa/bbb`
否

Prefix
`/aaa/bbb`
`/aaa/bbb`
是

Prefix
`/aaa/bbb/`
`/aaa/bbb`
是，忽略尾部斜线

Prefix
`/aaa/bbb`
`/aaa/bbb/`
是，匹配尾部斜线

Prefix
`/aaa/bbb`
`/aaa/bbb/ccc`
是，匹配子路径

Prefix
`/aaa/bbb`
`/aaa/bbbxyz`
否，字符串前缀不匹配

Prefix
`/`, `/aaa`
`/aaa/ccc`
是，匹配 `/aaa` 前缀

Prefix
`/`, `/aaa`, `/aaa/bbb`
`/aaa/bbb`
是，匹配 `/aaa/bbb` 前缀

Prefix
`/`, `/aaa`, `/aaa/bbb`
`/ccc`
是，匹配 `/` 前缀

Prefix
`/aaa`
`/ccc`
否，使用默认后端

混合
`/foo` (Prefix), `/foo` (Exact)
`/foo`
是，优选 Exact 类型

### [](#验证-1)验证

#### [](#方案一)方案一

在物理机机上验证:

- 在 win 中的 hosts（C:\Windows\System32\drivers\etc\hosts） 文件中添加如下的信息：

1
2
3
# 用来模拟 DNS ，其中 192.168.10.137 是 ingress 部署的机器，nginx.xudaxian.com 和 tomcat.xudaxian.com 是 ingress 文件中监听的域名
192.168.10.137 nginx.xudaxian.com
192.168.10.137 tomcat.xudaxian.com

- 在物理机终端使用curl命令

1
curl nginx.xudaxian.com

#### [](#方案二)方案二

新建一台虚拟机（centOS 7)并带有桌面：

- 在 hosts （`/etc/hosts`）文件中添加如下的内容：

1
2
3
# 用来模拟 DNS ，其中 192.168.10.137 是 ingress 部署的机器，nginx.xudaxian.com 和 tomcat.xudaxian.com 是 ingress 文件中监听的域名
192.168.10.137 nginx.xudaxian.com
192.168.10.137 tomcat.xudaxian.com

通过浏览器访问：nginx.xudaxian.com

### [](#默认后端)默认后端

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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: &quot;nginx&quot;
    nginx.ingress.kubernetes.io/backend-protocol: &quot;HTTP&quot;  
spec:
  defaultBackend: # 指定所有未匹配的默认后端
    service:
      name: nginx-svc
      port:
        number: 80
  rules: 
    - host: tomcat.com 
      http: 
        paths:
          - path: /abc
            pathType: Prefix 
            backend: 
              service:
                name: tomcat-svc
                port:
                  number: 8080

### [](#限流)限流

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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-ingress
  namespace: default
  annotations: # https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting
    kubernetes.io/ingress.class: &quot;nginx&quot;
    nginx.ingress.kubernetes.io/backend-protocol: &quot;HTTP&quot;  
    nginx.ingress.kubernetes.io/limit-rps: &quot;1&quot; # 限流
spec:
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80

### [](#路径重写（用于前后端分离）)路径重写（用于前后端分离）

![](C:\Users\32625\OneDrive\图片\k8s\33.png)

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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-rewrite
  namespace: default
  annotations:
   nginx.ingress.kubernetes.io/rewrite-target: /$2 # 路径重写
spec:
  ingressClassName: nginx
  rules:
  - host: baidu.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80

### [](#基于-Cookie-的会话保持技术)基于 Cookie 的会话保持技术

- Service 只能基于 ClientIP，但是 ingress 是七层负载均衡，可以基于 Cookie 实现会话保持。

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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-ingress
  namespace: default
  annotations: # https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting
    kubernetes.io/ingress.class: &quot;nginx&quot;
    nginx.ingress.kubernetes.io/backend-protocol: &quot;HTTP&quot;  
    nginx.ingress.kubernetes.io/affinity: &quot;cookie&quot;
spec:
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80

### [](#配置SSL)配置SSL

#### [](#生成证书)生成证书

1
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.cert -subj &quot;/CN=xudaxian.com/O=xudaxian.com&quot;

1
kubectl create secret tls xudaxian-tls --key tls.key --cert tls.cert

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-05-31 125543.png)

#### [](#配置文件-4)配置文件

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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-tls
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: &quot;nginx&quot;  
spec:
  tls:
  - hosts:
      - nginx.xudaxian.com # 通过浏览器访问 https://nginx.xudaxian.com
      - tomcat.xudaxian.com # 通过浏览器访问 https://tomcat.xudaxian.com
    secretName: xudaxian-tls
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  - host: tomcat.xudaxian.com 
    http: 
      paths:
      - path: /
        pathType: Prefix 
        backend:
          service:
            name: tomcat-svc 
            port:
              number: 8080
  #实际开发时，需要购买证书

### [](#ingress金丝雀发布)ingress金丝雀发布

**缺点：不能灰度自定义**

![](C:\Users\32625\OneDrive\图片\k8s\36.png)

#### [](#步骤)步骤

[](#部署service和deployment)部署service和deployment1
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
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
apiVersion: apps/v1
kind: Deployment
metadata:
  name: v1-deployment
  labels:
    app: v1-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: v1-pod
  template:
    metadata:
      name: nginx
      labels:
        app: v1-pod
    spec:
      initContainers:
        - name: alpine
          image: alpine
          imagePullPolicy: IfNotPresent
          command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;echo v1-pod &gt; /app/index.html;&quot;]
          volumeMounts:
            - mountPath: /app
              name: app    
      containers:
        - name: nginx
          image: nginx:1.17.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 250m
              memory: 500Mi 
          volumeMounts:
            - name: app
              mountPath: /usr/share/nginx/html             
      volumes:
        - name: app
          emptyDir: &#123;&#125;       
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: v1-service
  namespace: default
spec:
  selector:
    app: v1-pod
  type: ClusterIP
  ports:
  - name: nginx
    protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: v2-deployment
  labels:
    app: v2-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: v2-pod
  template:
    metadata:
      name: v2-pod
      labels:
        app: v2-pod
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 250m
              memory: 500Mi 
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: v2-service
  namespace: default
spec:
  selector:
    app: v2-pod
  type: ClusterIP
  ports:
  - name: nginx
    protocol: TCP
    port: 80
    targetPort: 80

[](#部署ingress)部署ingress1
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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-v1
  namespace: default
spec:
  ingressClassName: &quot;nginx&quot;
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: v1-service
            port:
              number: 80

[](#测试)测试1
curl -H &quot;Host: nginx.xudaxian.com&quot; http://192.168.10.135

#### [](#流量切分)流量切分

[](#基于-Header-的流量切分)基于 Header 的流量切分1
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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: &quot;true&quot; # 开启金丝雀 
    nginx.ingress.kubernetes.io/canary-by-header: &quot;Region&quot; # 基于请求头
    # 如果 请求头 Region = always ，就路由到金丝雀版本；如果 Region = never ，就永远不会路由到金丝雀版本。
    nginx.ingress.kubernetes.io/canary-by-header-value: &quot;sz&quot; # 自定义值
    # 如果 请求头 Region = sz ，就路由到金丝雀版本；如果 Region != sz ，就永远不会路由到金丝雀版本。
    # nginx.ingress.kubernetes.io/canary-by-header-pattern: &quot;sh|sz&quot;
    # 如果 请求头 Region = sh 或 Region = sz ，就路由到金丝雀版本；如果 Region != sz 并且 Region != sz ，就永远不会路由到金丝雀版本。
spec:
  ingressClassName: &quot;nginx&quot;
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: v2-service
            port:
              number: 80

1
2
3
4
5
6
7
#测试：

curl -H &quot;Host: nginx.xudaxian.com&quot; http://192.168.10.135

curl -H &quot;Host: nginx.xudaxian.com&quot; -H &quot;Region: sz&quot; http://192.10.135

curl -H &quot;Host: nginx.xudaxian.com&quot; -H &quot;Region: sh&quot; http://192.10.135

[](#基于-Cookie-的流量切分)基于 Cookie 的流量切分1
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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: &quot;true&quot; # 开启金丝雀 
    nginx.ingress.kubernetes.io/canary-by-cookie: &quot;vip&quot; # 如果 cookie 是 vip = always ，就会路由到到金丝雀版本；如果 cookie 是 vip = never ，就永远不会路由到金丝雀的版本。
spec:
  ingressClassName: &quot;nginx&quot;
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: v2-service
            port:
              number: 80

1
2
#验证：
curl -H &quot;Host: nginx.xudaxian.com&quot; --cookie &quot;vip=always&quot; http://192.168.10.135

[](#基于服务权重的流量切分)基于服务权重的流量切分1
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
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: &quot;true&quot; # 开启金丝雀 
    nginx.ingress.kubernetes.io/canary-weight: &quot;10&quot; # 基于服务权重
spec:
  ingressClassName: &quot;nginx&quot;
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: v2-service
            port:
              number: 80

1
2
#验证
for i in &#123;1..10&#125;; do curl -H &quot;Host: nginx.xudaxian.com&quot; http://192.168.10.135; done;

## [](#NetworkPolicy（网络策略）)NetworkPolicy（网络策略）

[网络策略](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/)指的是 Pod 间的网络隔离策略，默认情况下是互通的。

Pod  之间的互通是通过如下三个标识符的组合来辨识的： 

- ① 其他被允许的 Pod（例外：Pod 无法阻塞对自身的访问）。

- ② 被允许的名称空间。

- ③ IP 组块（例外：与 Pod 运行所在的节点的通信总是被允许的， 无论 Pod 或节点的 IP 地址）。

![](C:\Users\32625\OneDrive\图片\k8s\42.png)

### [](#pod的隔离与非隔离)pod的隔离与非隔离

默认情况下，Pod 都是非隔离的（non-isolated），可以接受来自任何请求方的网络请求。 

如果一个 NetworkPolicy 的标签选择器选中了某个 Pod，则该 Pod 将变成隔离的（isolated），并将拒绝任何不被 NetworkPolicy 许可的网络连接。

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

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: networkpol-01
  namespace: default
spec:
  podSelector: # Pod 选择器
    matchLabels:
      app: nginx  # 选中的 Pod 就被隔离起来了
  policyTypes: # 策略类型
  - Ingress # Ingress 入站规则、Egress 出站规则
  - Egress 
  ingress: # 入站白名单，什么能访问我
  - from:
    - podSelector: # 选中的 Pod 可以访问 spec.matchLabels 中筛选的 Pod 
        matchLabels:
          access: granted
    ports:
    - protocol: TCP
      port: 80
  egress: # 出站白名单，我能访问什么
    - to:
       - podSelector: # spec.matchLabels 中筛选的 Pod 能访问的 Pod
          matchLabels:
            app: tomcat
       - namespaceSelector: 
          matchLabels:
            kubernetes.io/metadata.name: dev # spec.matchLabels 中筛选的 Pod 能访问 dev 命名空间下的所有
      ports:
      - protocol: TCP
        port: 8080

### [](#场景)场景

[https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#default-policies](https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#default-policies)

# [](#配置和存储)配置和存储

**容器的生命周期可能很短，会被频繁的创建和销毁。那么容器在销毁的时候，保存在容器中的数据也会被清除。这种结果对用户来说，在某些情况下是不乐意看到的。为了持久化保存容器中的数据，Kubernetes 引入了 Volume 的概念。和 Docker 中的卷管理（匿名卷、具名卷、自定义挂载目录，都是挂载在本机，功能非常有限）不同的是，Kubernetes 天生就是集群，所以为了方便管理，Kubernetes 将 `卷` 抽取为一个对象资源，这样可以更方便的管理和存储数据。**

**Volume 是 Pod 中能够被多个容器访问的共享目录，它被定义在 Pod 上，然后被一个 Pod 里面的多个容器挂载到具体的文件目录下，Kubernetes 通过 Volume 实现同一个 Pod 中不同容器之间的数据共享以及数据的持久化存储。Volume 的生命周期不和 Pod 中的单个容器的生命周期有关，当容器终止或者重启的时候，Volume 中的数据也不会丢失**

![](C:\Users\32625\OneDrive\图片\k8s\1.png)

## [](#k8s支持的Volume类型)k8s支持的Volume类型

### [](#非持久性存储：)非持久性存储：

- emptyDir

- HostPath

### [](#网络连接性存储：)网络连接性存储：

- SAN：iSCSI、ScaleIO Volumes、FC (Fibre Channel)

- NFS：nfs，cfs

### [](#分布式存储)分布式存储

- Glusterfs

- RBD (Ceph Block Device)

- CephFS

- Portworx Volumes

- Quobyte Volumes

### [](#云端存储)云端存储

- GCEPersistentDisk

- AWSElasticBlockStore

- AzureFile

- AzureDisk

- Cinder (OpenStack block storage)

- VsphereVolume

- StorageOS

### [](#自定义存储)自定义存储

- FlexVolume

![](C:\Users\32625\OneDrive\图片\k8s\2 (2).png)

## [](#Secrat)Secrat

- Secret 对象类型用来保存敏感信息，如：密码、OAuth2 令牌以及 SSH 密钥等。将这些信息放到 Secret 中比放在 Pod 的定义或者容器镜像中更加安全和灵活。

- 由于创建 Secret 可以独立于使用它们的 Pod， 因此在创建、查看和编辑 Pod 的工作流程中暴露 Secret（及其数据）的风险较小。 Kubernetes 和在集群中运行的应用程序也可以对 Secret 采取额外的预防措施，如：避免将机密数据写入非易失性存储。

- Secret 类似于 [ConfigMap](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/) 但专门用于保存机密数据。

示例：

1
2
3
4
echo &#x27;damin&#x27; | base64 #输出
YWRtaW4K #加密
echo &#x27;YWRtaW4K&#x27; | base64 --decode #解除加密
damin

### [](#类型)类型

内置类型
用法

`Opaque`
用户定义的任意数据

`kubernetes.io/service-account-token`
服务账号令牌

`kubernetes.io/dockercfg`
`~/.dockercfg` 文件的序列化形式

`kubernetes.io/dockerconfigjson`
`~/.docker/config.json` 文件的序列化形式

`kubernetes.io/basic-auth`
用于基本身份认证的凭据

`kubernetes.io/ssh-auth`
用于 SSH 身份认证的凭据

`kubernetes.io/tls`
用于 TLS 客户端或者服务器端的数据

`bootstrap.kubernetes.io/token`
启动引导令牌数据

要使用 Secret，Pod 需要引用 Secret。 Pod 可以用三种方式之一来使用 Secret ： 

- 作为挂载到一个或多个容器上的 [卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/) 中的[文件](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。（volume进行挂载）

- 作为[容器的环境变量](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)（envFrom字段引用）

- 由 [kubelet 在为 Pod 拉取镜像时使用](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-imagepullsecrets)（此时Secret是docker-registry类型的）

Secret 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。 在为创建 Secret 编写配置文件时，你可以设置 `data` 与&#x2F;或 `stringData` 字段。 `data` 和 `stringData` 字段都是可选的。`data` 字段中所有键值都必须是 base64 编码的字符串。如果不希望执行这种 base64 字符串的转换操作，你可以选择设置 `stringData` 字段，其中可以使用任何字符串作为其取值。

### [](#使用命令创建Secret)使用命令创建Secret

1
2
3
kubectl create secret generic secret-1 \
   --from-literal=username=admin \
   --from-literal=password=123456

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-01 164616.png)

### [](#yaml格式创建Secret)yaml格式创建Secret

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
apiVersion: v1
kind: Secret
metadata:
  name: vol-secret
  namespace: default
type:
stringData: #k8s自动加密
      username: admin
      password: &quot;123456&quot;
#如果同时使用deta和stringDta,data会被忽略

#### [](#提取)提取

1
kubectl get  secret vol-secret -o yaml

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-01 181508.png)

### [](#环境变量的引用)环境变量的引用

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
54
55
56
57
58
59
60
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData: 
  username: admin
  password: &quot;1234556&quot;
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: default
  labels:
    app: pod-secret
spec:
  containers:
  - name: alpine
    image: alpine
    command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;sleep 3600&quot;]
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    env:
    - name: SECRET_USERNAME # 容器中的环境变量名称
      valueFrom:
        secretKeyRef: 
          name: my-secret #  指定 secret 的名称
          key: username # secret 中 key 的名称，会自动 base64 解码
    - name: SECRET_PASSWORD # 容器中的环境变量名称
      valueFrom:
        secretKeyRef:
          name: my-secret #  指定 secret 的名称
          key: password # secret 中 key 的名称   
    - name: POD_NAME
      valueFrom: 
        fieldRef:  # 属性引用
          fieldPath: metadata.name
    - name: POD_LIMITS_MEMORY
      valueFrom:
        resourceFieldRef:  # 资源限制引用 
          containerName: alpine  
          resource: limits.memory       
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

### [](#挂载卷)挂载卷

注意：

如果 Secret 以卷挂载的方式，Secret 里面的所有 key 都是文件名，内容就是 key 对应的值。

如果 Secret 以卷挂载的方式，Secret 的内容更新，那么容器对应的值也会被更新（subPath 引用除外）。

如果 Secret 以卷挂载的方式，默认情况下，挂载出来的文件是只读的。

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
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData: 
  username: admin
  password: &quot;1234556&quot;
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: default
  labels:
    app: pod-secret
spec:
  containers:
  - name: alpine
    image: alpine
    command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;sleep 3600&quot;]
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi     
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: app
      mountPath: /app  
  volumes:
    - name: app
      secret:
        secretName: my-secret # secret 的名称，Secret 中的所有 key 全部挂载出来
        items:
          - key: username # secret 中 key 的名称，Secret 中的 username 的内容挂载出来
            path: username.md # 在容器内挂载出来的文件的路径
  restartPolicy: Always

## [](#ConfigMap)ConfigMap

- ConfigMap与Secret相似，但是Secret会将base 64进行编码和解码，而ConfigMap则不会

### [](#yaml格式创建)yaml格式创建

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
54
55
56
57
kind: ConfigMap
apiVersion: v1
metadata:
  name: cm
  namespace: default
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: &quot;3&quot;
  ui_properties_file_name: &quot;user-interface.properties&quot;
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true 
---
apiVersion: v1
kind: Pod
metadata:
  name: &quot;pod-cm&quot;
  namespace: default
  labels:
    app: &quot;pod-cm&quot;
spec:
  containers:
  - name: alpine
    image: &quot;alpine&quot;
    command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;sleep 3600&quot;]
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    env:
    - name: PLAYER_INITIAL_LIVES
      valueFrom:
        configMapKeyRef:
          name: cm
          key: player_initial_lives
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: app
      mountPath: /app  
  volumes:
    - name: app
      configMap:
        name: cm # secret 的名称，Secret 中的所有 key 全部挂载出来
        items:
          - key: ui_properties_file_name # secret 中 key 的名称，Secret 中的 ui_properties_file_name 的内容挂载出来
            path: ui_properties_file_name.md # 在容器内挂载出来的文件的路径
  restartPolicy: Always

注意：

ConfigMap 和 Secret 一样，环境变量引用不会热更新，而卷挂载是可以热更新的。

最新版本的 ConfigMap 和 Secret 提供了不可更改的功能，即禁止热更新，只需要在 Secret 或 ConfigMap 中设置 `immutable = true` 。

### [](#结合-SpringBoot-做到生产配置无感知)结合 SpringBoot 做到生产配置无感知

![](https://cdn.nlark.com/yuque/0/2022/png/513185/1648538977212-715a4ba9-6e3a-45ba-aecc-dd7541dbf870.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_39,text_6K645aSn5LuZ,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

## [](#临时存储)临时存储

Kubernetes 为了不同的目的，支持几种不同类型的临时卷： 

[emptyDir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir)： Pod 启动时为空，存储空间来自本地的 kubelet 根目录（通常是根磁盘）或内存

[configMap](https://kubernetes.io/zh/docs/concepts/storage/volumes/#configmap)、 [downwardAPI](https://kubernetes.io/zh/docs/concepts/storage/volumes/#downwardapi)、 [secret](https://kubernetes.io/zh/docs/concepts/storage/volumes/#secret)： 将不同类型的 Kubernetes 数据注入到 Pod 中

[CSI 临时卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi-ephemeral-volumes)： 类似于前面的卷类型，但由专门[支持此特性](https://kubernetes-csi.github.io/docs/drivers.html) 的指定 [CSI 驱动程序](https://github.com/container-storage-interface/spec/blob/master/spec.md)提供

通用临时卷]([https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes)：](https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes)%EF%BC%9A) 它可以由所有支持持久卷的存储驱动程序提供

`emptyDir`、`configMap`、`downwardAPI`、`secret` 是作为 [本地临时存储](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage) 提供的。它们由各个节点上的 kubelet 管理。

CSI 临时卷 `必须` 由第三方 CSI 存储驱动程序提供。

通用临时卷 `可以`  由第三方 CSI 存储驱动程序提供，也可以由支持动态配置的任何其他存储驱动程序提供。 一些专门为 CSI 临时卷编写的 CSI 驱动程序，不支持动态供应：因此这些驱动程序不能用于通用临时卷。

使用第三方驱动程序的优势在于，它们可以提供 Kubernetes 本身不支持的功能， 例如，与 kubelet 管理的磁盘具有不同运行特征的存储，或者用来注入不同的数据。

### [](#emptyDir)emptyDir

`emptyDir` 的一些用途： 

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留。

- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。

`emptyDir` 卷存储在该节点的磁盘或内存中，如果设置 `emptyDir.medium = Memory` ，那么就告诉 Kubernetes 将数据保存在内存中，并且在 Pod 被重启或删除前，所写入的所有文件都会计入容器的内存消耗，受到容器内存限制约束

![](C:\Users\32625\OneDrive\图片\k8s\17.png)

**Pod 分派到某个 Node 上时，`emptyDir` 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。**

**就像其名称表示的那样，卷最初是空的。**

**尽管 Pod 中的容器挂载 `emptyDir` 卷的路径可能相同也可能不同，这些容器都可以读写 `emptyDir` 卷中相同的文件。**

**当 Pod 因为某些原因被从节点上删除时，`emptyDir` 卷中的数据也会被永久删除**。

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx-empty-dir
  namespace: default
  labels:
    app: nginx-empty-dir
spec:
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi    
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: app
      mountPath: /usr/share/nginx/html
  - name: alpine
    image: alpine
    command: [&quot;/bin/sh&quot;,&quot;-c&quot;,&quot;while true;do sleep 1; date &gt; /app/index.html;done&quot;]
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi    
    volumeMounts:
    - name: app
      mountPath: /app     
  volumes:
    - name: app
      emptyDir: &#123;&#125; # emptyDir 临时存储
  restartPolicy: Always

### [](#hostPath)hostPath

emptyDir 中的数据不会被持久化，它会随着 Pod 的结束而销毁，如果想要`简单`的将数据持久化到主机中，可以选择 hostPath 。

hostPath  就是将 Node 主机中的一个实际目录挂载到 Pod 中，以供容器使用，这样的设计就可以保证 Pod 销毁了，但是数据依旧可以保存在 Node 主机上。

注意：hostPath 之所以被归为临时存储，是因为实际开发中，我们一般都是通过 Deployment 部署 Pod 的，一旦 Pod 被 Kubernetes 杀死或重启拉起等，并不一定会部署到原来的 Node 节点中。

- 除了必需的 `path` 属性之外，用户可以选择性地为 `hostPath` 卷指定 `typ`

取值
行为

空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。

`DirectoryOrCreate`
如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。

`Directory`
在给定路径上必须存在的目录。

`FileOrCreate`
如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。

`File`
在给定路径上必须存在的文件。

`Socket`
在给定路径上必须存在的 UNIX 套接字。

`CharDevice`
在给定路径上必须存在的字符设备。

`BlockDevice`
在给定路径上必须存在的块设备。

注意：hostPath 的典型应用就是时间同步：通常而言，Node 节点的时间是同步的（云厂商提供的云服务器的时间都是同步的），但是，Pod 中的容器的时间就不一定了，有 UTC 、CST 等；同一个 Pod ，如果部署到中国，就必须设置为 CST 了。

#### [](#配置文件-5)配置文件

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx-host-path
  namespace: default
  labels:
    app: nginx
spec:
  containers:
  - name: nginx    
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath: # hostPath
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

## [](#持久化存储)持久化存储

### [](#volume)volume

Kubernetes 支持很多类型的卷。 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以同时使用任意数目的卷类型。

临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长。

当 Pod 不再存在时，Kubernetes 也会销毁临时卷。

Kubernetes 不会销毁 `持久卷` 。

对于给定 Pod 中 `任何类型的卷` ，在容器重启期间数据都不会丢失。

使用卷时, 在 `.spec.volumes` 字段中设置为 Pod 提供的卷，并在 `.spec.containers[*].volumeMounts` 字段中声明卷在容器中的挂载位置。

### [](#subPath)subPath

- 有时，在单个 Pod 中共享卷以供多方使用是很有用的。 `volumeMounts.subPath` 属性可用于指定所引用的卷内的子路径，而不是其根路径。

​       注意：ConfigMap 和 Secret 使用子路径挂载是无法热更新的。

### [](#安装FNS)安装FNS

网络文件系统（Network File System **[ NFS ]** )，是由 [SUN ](https://baike.baidu.com/item/SUN/69463)公司研制的[UNIX](https://baike.baidu.com/item/UNIX/219943)[表示层](https://baike.baidu.com/item/%E8%A1%A8%E7%A4%BA%E5%B1%82/4329716)协议(presentation layer protocol)，能使使用者访问网络上别处的文件就像在使用自己的计算机一样。

![](C:\Users\32625\OneDrive\图片\k8s\20.png)

#### [](#示例)示例

[](#master节点执行的操作)master节点执行的操作以 Master （192.168.10.137）节点作为 NFS 服务端：

1
yum install -y nfs-utils

创建 &#x2F;etc&#x2F;exports 文件

1
2
# * 表示暴露权限给所有主机；* 也可以使用 192.168.0.0/16 代替，表示暴露给所有主机
echo &quot;/nfs/data/ *(insecure,rw,sync,no_root_squash)&quot; &gt; /etc/exports

创建 `/nfs/data/` （共享目录）目录，并设置权限

1
2
3
mkdir -pv /nfs/data/

chmod 777 -R /nfs/data/

启动NFS

1
2
3
4
5
6
7
systemctl enable rpcbind

systemctl enable nfs-server

systemctl start rpcbind

systemctl start nfs-server

在Master节点上加载配置

1
exportfs -r

检查配置是否生效

1
exportfs

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-04 190202.png)

[](#node节点上执行的操作)node节点上执行的操作在node（192.168.10.135 &#x2F; 192.168.10.136)节点上安装nfs-utils

1
2
# 服务器端防火墙开放111、662、875、892、2049的 tcp / udp 允许，否则远端客户无法连接。
yum install -y nfs-utils

执行以下命令检查 nfs 服务器端是否有设置共享目录

1
2
# showmount -e &lt;nfs服务器的ip&gt;
showmount -e 192.168.10.137

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-04 191711.png)

执行以下命令挂载 nfs 服务器上的共享目录到本机路径 `/root/nd`：

1
2
3
4
mkdir /nd

# mount -t nfs &lt;nfs服务器的IP&gt;:/root/nfs_root /root/nfsmount
mount -t nfs 192.168.10.137:/nfs/data /nd

在node节点写入一个测试文件

1
echo &quot;hello nfs server&quot; &gt; /nd/test.txt

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-04 192359.png)

在master节点验证文件是否写入成功

1
cat /nfs/data/test.txt

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-04 192355.png)

### [](#配置文件-6)配置文件

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort: 80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
    - name: html  
      mountPath: /usr/share/nginx/html/ # / 一定是文件夹
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
    - name: html 
      nfs: # 使用 nfs 存储驱动
        path: /nfs/data  # nfs 共享的目录
        server:  192.168.65.100  # nfs 服务端的 IP 地址或 hostname 
  restartPolicy: Always

# [](#调度原理)调度原理

## [](#ResourceQuotas（资源配额）)ResourceQuotas（资源配额）

当多个用户或团队共享具有固定节点数目的集群时，人们会担心 `有人使用超过其基于公平原则所分配到的资源量` 。
  资源配额是帮助管理员解决这一问题的工具。 

资源配额，通过 `ResourceQuota` 对象来定义，`对每个命名空间的资源消耗总量提供限制`。 它可以 `限制` 命名空间中 `某种类型的对象的总数目上限`，`也可以限制命令空间中的 Pod 可以使用的计算资源的总上限`。 

资源配额的工作方式如下： 

- 不同的团队可以在不同的命名空间下工作，目前这是非约束性的，在未来的版本中可能会通过 ACL (Access Control List 访问控制列表) 来实现强制性约束。

- `集群管理员可以为每个命名空间创建一个或多个 ResourceQuota 对象`。

- 当用户在命名空间下创建资源（如 Pod、Service 等）时，Kubernetes 的配额系统会跟踪集群的资源使用情况，`以确保使用的资源用量不超过 ResourceQuota 中定义的硬性资源限额`。

- 如果资源创建或者更新请求违反了配额约束，那么该请求会报错（HTTP 403 FORBIDDEN）， 并在消息中给出有可能违反的约束。

- 如果命名空间下的计算资源 （如 `cpu` 和 `memory`）的`配额被启用`，则`用户必须为 这些资源设定请求值（request）和约束值（limit）`，否则配额系统将拒绝 Pod 的创建。 提示: 可使用 `LimitRanger` 准入控制器来为没有设置计算资源需求的 Pod 设置默认值。

### [](#计算资源配额)计算资源配额

- 用户可以对给定命名空间下的可被请求的 [计算资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/) 总量进行限制。

`limits.cpu`
所有非终止状态的 Pod，其 CPU 限额总量不能超过该值。

`limits.memory`
所有非终止状态的 Pod，其内存限额总量不能超过该值。

`requests.cpu`
所有非终止状态的 Pod，其 CPU 需求总量不能超过该值。

`requests.memory`
所有非终止状态的 Pod，其内存需求总量不能超过该值。

`hugepages-&lt;size&gt;`
对于所有非终止状态的 Pod，针对指定尺寸的巨页请求总数不能超过此值。

`cpu`
与 `requests.cpu` 相同。

`memory`
与 `requests.memory` 相同。

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
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu
  namespace: default # 限制 default 名称空间
spec:
  hard:
    requests.cpu: &quot;1&quot;
    requests.memory: 1Gi
    limits.cpu: &quot;2&quot;
    limits.memory: 2Gi

### [](#扩展资源的资源配额)扩展资源的资源配额

●除上述资源外，在 Kubernetes 1.10 版本中，还添加了对 [扩展资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/#extended-resources) 的支持。
●由于扩展资源不可超量分配，因此没有必要在配额中为同一扩展资源同时指定 requests 和 limits。 对于扩展资源而言，目前仅允许使用前缀为 requests. 的配额项。
●以 GPU 拓展资源为例，如果资源名称为 nvidia.com&#x2F;gpu，并且要将命名空间中请求的 GPU 资源总数限制为 4，则可以如下定义配额：

1
requests.nvidia.com/gpu: 4

**有关更多详细信息，请参阅官方文档[查看和设置配额](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#viewing-and-setting-quotas)**。

### [](#存储资源配额)存储资源配额

用户可以对给定命名空间下的 [存储资源](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/) 总量进行限制。

此外，还可以根据相关的存储类（Storage Class）来限制存储资源的消耗。

资源名称
描述

`requests.storage`
所有 PVC，存储资源的需求总量不能超过该值。

`persistentvolumeclaims`
在该命名空间中所允许的 [PVC](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 总量。

`&lt;storage-class-name&gt;.storageclass.storage.k8s.io/requests.storage`
在所有与 `&lt;storage-class-name&gt;` 相关的持久卷申领中，存储请求的总和不能超过该值。

`&lt;storage-class-name&gt;.storageclass.storage.k8s.io/persistentvolumeclaims`
在与 storage-class-name 相关的所有持久卷申领中，命名空间中可以存在的[持久卷申领](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)总数。

例如，如果一个操作人员针对 `gold` 存储类型与 `bronze` 存储类型设置配额， 操作人员可以定义如下配额： 

- `gold.storageclass.storage.k8s.io/requests.storage: 500Gi`

- `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`

在 Kubernetes 1.8 版本中，本地临时存储的配额支持已经是 Alpha 功能：

资源名称
描述

`requests.ephemeral-storage`
在命名空间的所有 Pod 中，本地临时存储请求的总和不能超过此值。

`limits.ephemeral-storage`
在命名空间的所有 Pod 中，本地临时存储限制值的总和不能超过此值。

`ephemeral-storage`
与 `requests.ephemeral-storage` 相同。

**注意：如果所使用的是 CRI 容器运行时，容器日志会被计入临时存储配额。 这可能会导致存储配额耗尽的 Pods 被意外地驱逐出节点。 参考[日志架构](https://kubernetes.io/zh/docs/concepts/cluster-administration/logging/) 了解详细信息。**

### [](#对象数量配额)对象数量配额

你可以使用以下语法对所有标准的、命名空间域的资源类型进行配额设置： 

- `count/&lt;resource&gt;.&lt;group&gt;`：用于非核心（core）组的资源。

- `count/&lt;resource&gt;`：用于核心组的资源。

这是用户可能希望利用对象计数配额来管理的一组资源示例。 

- `count/persistentvolumeclaims`

- `count/services`

- `count/secrets`

- `count/configmaps`

- `count/replicationcontrollers`

- `count/deployments.apps`

- `count/replicasets.apps`

- `count/statefulsets.apps`

- `count/jobs.batch`

- `count/cronjobs.batch`

相同语法也可用于自定义资源。 例如，要对 `example.com` API 组中的自定义资源 `widgets` 设置配额，请使用 `count/widgets.example.com`。 

当使用 `count/*` 资源配额时，如果对象存在于服务器存储中，则会根据配额管理资源。 这些类型的配额有助于防止存储资源耗尽。例如，用户可能想根据服务器的存储能力来对服务器中 Secret 的数量进行配额限制。 集群中存在过多的 Secret 实际上会导致服务器和控制器无法启动。 用户可以选择对 Job 进行配额管理，以防止配置不当的 CronJob 在某命名空间中创建太多 Job 而导致集群拒绝服务。 

对有限的一组资源上实施一般性的对象数量配额也是可能的。 此外，还可以进一步按资源的类型设置其配额。

资源名称
描述

`configmaps`
在该命名空间中允许存在的 ConfigMap 总数上限。

`persistentvolumeclaims`
在该命名空间中允许存在的 [PVC](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 的总数上限。

`pods`
在该命名空间中允许存在的非终止状态的 Pod 总数上限。Pod 终止状态等价于 Pod 的 `.status.phase in (Failed, Succeeded)` 为真。

`replicationcontrollers`
在该命名空间中允许存在的 ReplicationController 总数上限。

`resourcequotas`
在该命名空间中允许存在的 ResourceQuota 总数上限。

`services`
在该命名空间中允许存在的 Service 总数上限。

`services.loadbalancers`
在该命名空间中允许存在的 LoadBalancer 类型的 Service 总数上限。

`services.nodeports`
在该命名空间中允许存在的 NodePort 类型的 Service 总数上限。

`secrets`
在该命名空间中允许存在的 Secret 总数上限。

 **例如，`pods` 配额统计某个命名空间中所创建的、非终止状态的 `Pod` 个数并确保其不超过某上限值。 用户可能希望在某命名空间中设置 `pods` 配额，以避免有用户创建很多小的 Pod， 从而耗尽集群所能提供的 Pod IP 地址。** 

1
2
3
4
5
6
7
8
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-rq
  namespace: default # 限制 default 名称空间
spec:
  hard:
    pods: &quot;3&quot; # 限制 Pod 的数量

Pod 可以创建为特定的[优先级](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority)。 通过使用配额规约中的 `scopeSelector` 字段，用户可以根据 Pod 的优先级控制其系统资源消耗。 

仅当配额规范中的 `scopeSelector` 字段选择到某 Pod 时，配额机制才会匹配和计量 Pod 的资源消耗。 

如果配额对象通过 `scopeSelector` 字段设置其作用域为优先级类，则配额对象只能 跟踪以下资源： 

- `pods`

- `cpu`

- `memory`

- `ephemeral-storage`

- `limits.cpu`

- `limits.memory`

- `limits.ephemeral-storage`

- `requests.cpu`

- `requests.memory`

- `requests.ephemeral-storage`

 **示例：创建一个配额对象，并将其与具有特定优先级的 Pod 进行匹配** 

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
# 集群中的 Pod 可取三个优先级类之一，即 &quot;low&quot;、&quot;medium&quot;、&quot;high&quot;
# 为每个优先级创建一个配额对象

apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: &quot;1000&quot;
      memory: 200Gi
      pods: &quot;10&quot;
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: [&quot;high&quot;]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: &quot;10&quot;
      memory: 20Gi
      pods: &quot;10&quot;
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: [&quot;medium&quot;]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: &quot;5&quot;
      memory: 10Gi
      pods: &quot;10&quot;
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: [&quot;low&quot;]

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
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: [&quot;/bin/sh&quot;]
    args: [&quot;-c&quot;, &quot;while true; do echo hello; sleep 10;done&quot;]
    resources:
      requests:
        memory: &quot;10Gi&quot;
        cpu: &quot;500m&quot;
      limits:
        memory: &quot;10Gi&quot;
        cpu: &quot;500m&quot;
  priorityClassName: high # priorityClassName 指的是什么，就优先使用这个优先级的配额约束。

## [](#LimitRange（限制范围）)LimitRange（限制范围）

默认情况下， Kubernetes 集群上的容器运行使用的[计算资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/)没有限制。 

使用资源配额，集群管理员可以以[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)为单位，限制其资源的使用与创建。 

在命名空间中，一个 Pod 或 Container 最多能够使用命名空间的资源配额所定义的 CPU 和内存用量。 

有人担心，一个 Pod 或 Container 会垄断所有可用的资源。 LimitRange 是在命名空间内限制资源分配（给多个 Pod 或 Container）的策略对象，例如：资源额度 ResourceQuotas 设置为 requests.cpu 为 1 ，requests.memory 为 1Gi ，但是有人的 Pod 直接设置为 requests.cpu 为 1，requests.memory 为 1Gi，那么其他人就不能再创建 Pod 了，所以需要使用 LimitRange 对资源配额设置区间（最小和最大）。 

一个 LimitRange（限制范围） 对象提供的限制能够做到： 

- 在一个命名空间中实施对每个 Pod 或 Container 最小和最大的资源使用量的限制。

- 在一个命名空间中实施对每个 PersistentVolumeClaim 能申请的最小和最大的存储空间大小的限制。

- 在一个命名空间中实施对一种资源的申请值和限制值的比值的控制。

- 设置一个命名空间中对计算资源的默认申请&#x2F;限制值，并且自动的在运行时注入到多个 Container 中。

#### [](#为命名空间配置-CPU-最小和最大约束)为命名空间配置 CPU 最小和最大约束

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
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
  namespace: default # 限制 default 命名空间
spec:
  limits:
  - max:
      cpu: &quot;800m&quot;
    min:
      cpu: &quot;200m&quot; # Pod 中的容器的 cpu 的范围是 200m~800m
    type: Container

#### [](#配置命名空间的最小和最大内存约束)配置命名空间的最小和最大内存约束

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
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-min-max-demo-lr
  namespace: default # 限制 default 命名空间  
spec:
  limits:
  - max:
      memory: 1Gi
    min:
      memory: 500Mi # Pod 中的容器的 memory 的范围是 500Mi~1Gi
    type: Container

#### [](#为命名空间配置默认的-CPU-请求和限制)为命名空间配置默认的 CPU 请求和限制

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
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
  namespace: default # 限制 default 命名空间    
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 500m # 声明了一个默认的 CPU 请求和一个默认的 CPU 限制
    type: Container

#### [](#为命名空间配置默认的内存请求和限制)为命名空间配置默认的内存请求和限制

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
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: default # 限制 default 命名空间     
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi # 为命名空间配置默认的内存请求和限制
    type: Container

#### [](#限制存储消耗)限制存储消耗

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
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi

1
2
3
4
5
6
7
8
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: &quot;5&quot; # 限制 PVC 数目和累计存储容量
    requests.storage: &quot;5Gi&quot;

## [](#调度原理-1)调度原理

在默认情况下，一个 Pod 在哪个 Node 节点上运行，是由 Scheduler 组件采用相应的算法计算出来的，这个过程是不受人工控制的。但是在实际使用中，这并不满足需求，因为很多情况下，我们想控制某些 Pod 到达某些节点上，那么应该怎么做？这就要求了解 Kubernetes 对 Pod 的调度规则，Kubernetes 提供了四大类调度方式： 

- 自动调度（默认）：运行在哪个 Node 节点上完全由 Scheduler 经过一系列的算法计算得出。

- 定向调度：NodeName、NodeSelector。

- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity。

- 污点（容忍）调度：Taints、Toleration。

### [](#定向调度)定向调度

- 定向调度，指的是利用在 Pod 上声明的 nodeName 或 nodeSelector ，以此将 Pod 调度到期望的 Node 节点上。

**注意：这里的调度是强制的，这就意味着即使要调度的目标 Node 不存在，也会向上面进行调度，只不过 Pod 运行失败而已。**

#### [](#nodeName（不建议）)nodeName（不建议）

`nodeName` 是节点选择约束的最简单方法，但是由于其自身限制，通常不使用它。 `nodeName` 是 PodSpec 的一个字段。 如果它不为空，调度器将忽略 Pod，并且给定节点上运行的 kubelet 进程尝试执行该 Pod。 因此，如果 `nodeName` 在 Pod 的 Spec 中指定了，则它优先于 nodeSelector 。 

使用 `nodeName` 来选择节点的一些限制： 

- 如果指定的节点不存在，

- 如果指定的节点没有资源来容纳 Pod，Pod 将会调度失败并且其原因将显示为， 比如 OutOfmemory 或 OutOfcpu。

- 云环境中的节点名称并非总是可预测或稳定的。

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  nodeName: k8s-node1 # 指定调度到k8s-node1节点上
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

#### [](#nodeSelector)nodeSelector

●nodeSelector 是节点选择约束的最简单推荐形式。nodeSelector 是 PodSpec 的一个字段。 它包含键值对的映射。为了使 pod 可以在某个节点上运行，该节点的标签中 必须包含这里的每个键值对（它也可以具有其他标签）。 最常见的用法的是一对键值对。

●除了自己 [添加](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#step-one-attach-label-to-the-node) 的标签外，节点还预制了一组标准标签。 参见这些[常用的标签，注解以及污点](https://kubernetes.io/zh/docs/reference/labels-annotations-taints/)： 

○[kubernetes.io&#x2F;hostname](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-hostname)

○[failure-domain.beta.kubernetes.io&#x2F;zone](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesiozone)

○[failure-domain.beta.kubernetes.io&#x2F;region](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesioregion)

○[topology.kubernetes.io&#x2F;zone](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)

○[topology.kubernetes.io&#x2F;region](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)

○[beta.kubernetes.io&#x2F;instance-type](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#beta-kubernetes-io-instance-type)

○[node.kubernetes.io&#x2F;instance-type](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#nodekubernetesioinstance-type)

○[kubernetes.io&#x2F;os](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os)

○[kubernetes.io&#x2F;arch](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-arch)

注意：这些标签的值是特定于云供应商的，因此不能保证可靠。 例如，kubernetes.io&#x2F;hostname 的值在某些环境中可能与节点名称相同， 但在其他环境中可能是一个不同的值。

1
2
# 给 k8s-node2 打上标签
kubectl label node k8s-node2 nodeevn=pro

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  nodeSelector:
    nodeevn: pro # 指定调度到 nodeevn = pro 标签的 Node 节点上
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort: 80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  restartPolicy: Always

### [](#亲和性调度)亲和性调度

虽然定向调度的两种方式，使用起来非常方便，但是也有一定的问题，那就是如果没有满足条件的 Node，那么 Pod 将不会被运行，即使在集群中还有可用的 Node 列表也不行，这就限制了它的使用场景。 

基于上面的问题，Kubernetes 还提供了一种亲和性调度（Affinity）。它在 nodeSelector 的基础之上进行了扩展，可以通过配置的形式，实现优先选择满足条件的 Node 进行调度，如果没有，也可以调度到不满足条件的节点上，使得调度更加灵活。 

**Affinity 主要分为三类：** 

- **nodeAffinity（node亲和性）：以 Node 为目标，解决 Pod可 以调度到那些 Node 的问题。**

- **podAffinity（pod亲和性）：以 Pod 为目标，解决 Pod 可以和那些已存在的 Pod 部署在同一个拓扑域中的问题。**

- **podAntiAffinity（pod反亲和性）：以 Pod 为目标，解决 Pod 不能和那些已经存在的 Pod 部署在同一拓扑域中的问题。**

**关于亲和性和反亲和性的使用场景的说明：**

亲和性：如果两个应用频繁交互，那么就有必要利用亲和性让两个应用尽可能的靠近，这样可以较少因网络通信而带来的性能损耗。 

反亲和性：当应用采用多副本部署的时候，那么就有必要利用反亲和性让各个应用实例打散分布在各个 Node 上，这样可以提高服务的高可用性。

#### [](#nodeAffinity)nodeAffinity

nodeAffinity 概念上类似于 `nodeSelector`，它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点。 

和 nodeSelector 的区别： 

- 引入了运算符：In、NotIn、Exists、DoesNotExist、 Gt、 Lt。

- 支持 `硬性过滤` 和 `软性评分` ：

- 硬件过滤支持指定`多条件之间的逻辑或运算`；

- 软件评分规则支持`设置条件权重`。

**nodeAffinity 的可选配置项：**

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
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  # Node节点必须满足指定的所有规则才可以，硬性过滤
    nodeSelectorTerms  # 节点选择列表
      matchFields   # 按节点字段列出的节点选择器要求列表  
      matchExpressions   # 按节点标签列出的节点选择器要求列表(推荐)
        key    # 键
        values # 值
        operator # 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution # 优先调度到满足指定的规则的Node，软性评分 (倾向)   
    preference   # 一个节点选择器项，与相应的权重相关联
      matchFields # 按节点字段列出的节点选择器要求列表
      matchExpressions   # 按节点标签列出的节点选择器要求列表(推荐)
        key # 键
        values # 值
        operator # 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt  
    weight # 倾向权重，在范围1-100。

*注意：*
*●如果我们修改或删除 Pod 调度到节点的标签，Pod 不会被删除；换言之，亲和调度只是在 Pod 调度期间有效。*
*●如果同时定义了 nodeSelector 和 nodeAffinity ，那么必须两个条件都满足，Pod 才能运行在指定的 Node 上。*
*●如果 nodeAffinity 指定了多个 nodeSelectorTerms ，那么只需要其中一个能够匹配成功即可。*
*●如果一个 nodeSelectorTerms 中有多个 matchExpressions ，则一个节点必须满足所有的才能匹配成功。*

**硬性过滤**

1
2
3
# 给 k8s-node1 和 k8s-nodes 打上标签
kubectl label node k8s-node1 disktype=ssd
kubectl label node k8s-node2 disktype=hdd

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
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:
    - containerPort: 80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
  affinity: 
    nodeAffinity:
      # DuringScheduling（调度期间有效）IgnoredDuringExecution（执行期间忽略，执行期间：Pod 运行期间）
      requiredDuringSchedulingIgnoredDuringExecution: # 硬性过滤：Node 节点必须满足指定的所有规则才可以
        nodeSelectorTerms:
          - matchExpressions: # 所有 matchExpressions 满足条件才行
            - key: disktype
              operator: In # 支持 Exists, DoesNotExist, In, NotIn, Gt, Lt
              values: 
                - ssd      
                - hdd
  restartPolicy: Always

**软性评分**

1
2
3
4
# 给 k8s-node1 和 k8s-nodes 打上标签

kubectl label node k8s-node1 disk=50 gpu=1000
kubectl label node k8s-node2 disk=30 gpu=5000

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
54
55
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:

  - name: nginx
    image: nginx:1.20.2
    resources:
      limits:
        cpu: 200m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 200Mi
    ports:

    - containerPort: 80
      name:  http
      volumeMounts:

    - name: localtime
      mountPath: /etc/localtime
      volumes:

    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
      affinity: 
      nodeAffinity:

      # DuringScheduling（调度期间有效）IgnoredDuringExecution（执行期间忽略，执行期间：Pod 运行期间）

      preferredDuringSchedulingIgnoredDuringExecution: # 软性评分：优先调度到满足指定的规则的 Node

        - weight: 90 # 权重
          preference: # 一个节点选择器项，与相应的权重相关联
            matchExpressions:
              - key: disk
                operator: Gt
                values: 
                 - &quot;40&quot;
        - weight: 10 # 权重
          preference: # 一个节点选择器项，与相应的权重相关联
            matchExpressions:
             - key: gpu
               operator: Gt
               values: 
                - &quot;4000&quot;        
                  restartPolicy: Always

#### [](#podAffinity-和-podAntiAffinity)podAffinity 和 podAntiAffinity

podAffinity 主要实现以运行的 Pod 为参照，实现让新创建的 Pod 和参照的 Pod 在一个区域的功能。 

podAntiAffinity 主要实现以运行的 Pod 为参照，让新创建的 Pod 和参照的 Pod 不在一个区域的功能。 

PodAffinity 的可选配置项：

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
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces 指定参照pod的namespace
    topologyKey 指定调度作用域
    labelSelector 标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容  
  preferredDuringSchedulingIgnoredDuringExecution 软限制    
    podAffinityTerm  选项
      namespaces
      topologyKey
      labelSelector
         matchExpressions 
            key    键  
            values 值  
            operator
         matchLabels 
    weight 倾向权重，在范围1-1

注意：topologyKey 用于指定调度的作用域，例如:

如果指定为 kubernetes.io&#x2F;hostname（可以通过 kubectl get node –show-labels 查看），那就是以 Node 节点为区分范围。 

如果指定为 kubernetes.io&#x2F;os，则以 Node 节点的操作系统类型来区分。

**示例：在一个两节点的集群中，部署一个使用 redis 的 WEB 应用程序，并期望 web-server 尽可能和 redis 在同一个节点上。**

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
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deploy
  namespace: default
  labels:
    app: redis-deploy
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity: # 亲和性配置
        podAntiAffinity: # Pod 反亲和性，符合以下指定条件不会被调度过去
          requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
            - labelSelector:
                matchExpressions: 
                  - key: app
                    operator: In
                    values:
                      - store
              topologyKey: kubernetes.io/hostname # 拓扑键，划分逻辑区域。 
              # node 节点以 kubernetes.io/hostname 为拓扑网络，如果 kubernetes.io/hostname 相同，就认为是同一个东西。
              # 亲和就是都放在这个逻辑区域，反亲和就是必须避免放在同一个逻辑区域。
      containers:
      - name: redis-server
        image: redis:5.0.14-alpine
        resources:
           limits:
             memory: 500Mi
             cpu: 1
           requests:
             memory: 250Mi
             cpu: 500m
        ports:
        - containerPort: 2375
          name: redis
        volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
      volumes:
        - name: localtime
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai
      restartPolicy: Always
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx-deploy
  namespace: default
  labels:
    app:  nginx-deploy
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity: # 亲和性配置
        podAffinity: # Pod 亲和性，符合以下指定条件会被调度过去
          requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
            - labelSelector:
                matchExpressions: 
                  - key: app
                    operator: In
                    values:
                      - store
              topologyKey: kubernetes.io/hostname # 拓扑键，划分逻辑区域。 
              # node 节点以 kubernetes.io/hostname 为拓扑网络，如果 kubernetes.io/hostname 相同，就认为是同一个东西。
              # 亲和就是都放在这个逻辑区域，反亲和就是必须避免放在同一个逻辑区域。
        podAntiAffinity: # Pod 反亲和性，符合以下指定条件不会被调度过去
          requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
            - labelSelector:
                matchExpressions: 
                  - key: app
                    operator: In
                    values:
                      - web-store
              topologyKey: kubernetes.io/hostname # 拓扑键，划分逻辑区域。 
              # node 节点以 kubernetes.io/hostname 为拓扑网络，如果 kubernetes.io/hostname 相同，就认为是同一个东西。
              # 亲和就是都放在这个逻辑区域，反亲和就是必须避免放在同一个逻辑区域。              
      containers:
      - name:  nginx
        image:  nginx:1.20.2
        resources:
          limits:
            cpu: 200m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        ports:
        - containerPort:  80
          name:  nginx
        volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
      volumes:
        - name: localtime
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai
      restartPolicy: Always

## [](#污点和容忍调度)污点和容忍调度

### [](#污点)污点

面的调度方式都是站在 Pod 的角度上，通过在 Pod 上添加属性，来确定 Pod 是否要调度到指定的 Node 上，其实我们也可以站在 Node 的角度上，通过在 Node 上添加`污点属性`，来决定是否运行 Pod 调度过来。

Node 被设置了污点之后就和 Pod 之间存在了一种相斥的关系，进而拒绝 Pod 调度进来，甚至可以将已经存在的 Pod 驱逐出去。

污点的格式为：

1
2
3
key=value:effect
# key 和 value 是污点的标签及对应的值
# effect 描述污点的作用

effect 支持如下的三个选项： 

- PreferNoSchedule：Kubernetes 将尽量避免把 Pod 调度到具有该污点的 Node 上，除非没有其他节点可以调度；换言之，尽量不要来，除非没办法。

- NoSchedule：Kubernets 将不会把 Pod 调度到具有该污点的 Node 上，但是不会影响当前 Node 上已经存在的 Pod ;换言之，新的不要来，在这的就不要动。

- NoExecute：Kubernets 将不会将 Pod 调度到具有该污点的 Node 上，同时会将 Node 上已经存在的 Pod 驱逐；换言之，新的不要来，这这里的赶紧走。

**注意：NoExecute 一般用于实际生产环境中的 Node 节点的上下线。**

**设置污点**

1
kubectl taint node xxx key=value:effect

**去除污点**

1
kubectl taint node xxx key:effect-

**去除所有污点**

1
kubectl taint node xxx key-

**查看污点**

1
kubectl describe node xxx | grep -i taints

- **证明：kubeadm 安装的集群上 k8s-master 有污点**

1
kubectl describe node k8s-master | grep -i taints

![](https://cdn.nlark.com/yuque/0/2022/png/513185/1648692456349-9a4ca09a-e5c4-4473-af6a-8c3a298accfe.png?x-oss-process=image/watermark,type_d3F5LW1pY3JvaGVp,size_34,text_6K645aSn5LuZ,color_FFFFFF,shadow_50,t_80,g_se,x_10,y_10)

示例：演示污点效果（为了演示效果更为明显，暂时停止 k8s-node2 节点）

① 为 k8s-node1 设置污点`tag=xudaxian:PreferNoSchedule`，然后创建 Pod1 （Pod1 可以运行）

1
kubectl taint node k8s-node1 tag=xudaxian:PreferNoSchedule

1
kubectl run pod1 --image=nginx:1.20.2

![](https://cdn.nlark.com/yuque/0/2022/gif/513185/1648692464879-6e01bead-ed97-46be-93ef-6cc2cf5c524d.gif)

- ② 修改 k8s-node1 节点的污点为 `tag=xudaxian:NoSchedule`，然后创建Pod2（Pod1 可以正常运行，Pod2 失败）。

1
2
3
4
# 取消污点
kubectl taint node k8s-node1 tag:PreferNoSchedule-
# 设置污点
kubectl taint node k8s-node1 tag=xudaxian:NoSchedule

1
kubectl run pod2 --image=nginx:1.20.2

![](https://cdn.nlark.com/yuque/0/2022/gif/513185/1648692471659-c922a543-2a3e-47cb-9ded-f1a5a0e05a1d.gif)

- ③ 修改 k8s-node1 节点的污点为 `tag=xudaxian:NoExecute`，然后创建 Pod3（Pod1、Pod2、Pod3失败）。

1
2
3
4
# 取消污点
kubectl taint node k8s-node1 tag:NoSchedule-
# 设置污点
kubectl taint node k8s-node1 tag=xudaxian:NoExecute

1
kubectl run pod3 --image=nginx:1.20.2

![](https://cdn.nlark.com/yuque/0/2022/gif/513185/1648692476438-b3547fd6-b251-4982-905a-8e91b663ee86.gif)

### [](#容忍)容忍

上面介绍了污点的作用，我们可以在 Node上 添加污点用来拒绝 Pod 调度上来，但是如果就是想让一个 Pod 调度到一个有污点的 Node 上去，这时候应该怎么做？这就需要使用到容忍。

**污点就是拒绝，容忍就是忽略，Node 通过污点拒绝 Pod 调度上去，Pod 通过容忍忽略拒绝。**

●容忍的详细配置：

1
2
3
4
5
6
7
8
kubectl explain pod.spec.tolerations
......
FIELDS:
  key       # 对应着要容忍的污点的键，空意味着匹配所有的键
  value     # 对应着要容忍的污点的值
  operator  # key-value的运算符，支持Equal和Exists（默认）
  effect    # 对应污点的effect，空意味着匹配所有影响
  tolerationSeconds   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间

污点和容忍的匹配： 

- 当满足如下条件的时候，Kubernetes 认为污点和容忍匹配：

- 键（key）相同。

- 效果（effect）相同。

- 污点的 operator 为：

- Exists ，此时污点中不应该指定 value 。

- Equal，此时容忍的 value 应该和污点的 value 相同。

- 如果不指定 operator ，默认为 Equal 。

特殊情况：

容忍中没有定义 key ，但是定义了 operator 为 Exists ，Kubernetes 则认为此容忍匹配所有的污点，如：

1
2
3
tolerations:
- operator: Exists
# 最终，所有有污点的机器我们都能容忍，Pod 都可以调用

容忍中没有定义 effect，但是定义了 key，Kubernetes 认为此容忍匹配所有 effect ，如：

1
2
3
4
  tolerations: # 容忍
    - key: &quot;tag&quot; # 要容忍的污点的key
      operator: Exists # 操作符
# 最终，有这个污点的机器我们可以容忍，Pod 都可以调度。

**示例**

1
2
# 设置污点
kubectl taint node k8s-node1 tag=xudaxian:NoExecute

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
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
spec:
  containers: # 容器配置
    - name: nginx
      image: nginx:1.20.2
      imagePullPolicy: IfNotPresent
      ports:
        - name: nginx-port
          containerPort: 80
          protocol: TCP
  tolerations: # 容忍
    - key: &quot;tag&quot; # 要容忍的污点的key
      operator: Equal # 操作符
      value: &quot;xudaxian&quot; # 要容忍的污点的value
      effect: NoExecute # 添加容忍的规则，这里必须和标记的污点规则相同

# [](#安全)安全

## [](#访问控制概述)访问控制概述

- 用户使用 `kubectl`、客户端库或构造 REST 请求来访问 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)。 人类用户和 [Kubernetes 服务账户](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/)都可以被鉴权访问 API。 当请求到达 API 时，它会经历多个阶段，如下图所示：

![](C:\Users\32625\OneDrive\图片\k8s\1 (1).png)

### [](#客户端)客户端

在 Kubernetes 集群中，客户端通常由两类： 

- ① User Account：一般是独立于 Kubernetes 之外的其他服务管理的用户账号。

- ② Service Account：Kubernetes 管理的账号，用于为Pod的服务进程在访问 Kubernetes 时提供身份标识。

### [](#认证、授权和准入控制)认证、授权和准入控制

api-server 是访问和管理资源对象的唯一入口。任何一个请求访问 api-server，都要经过下面的三个流程： 

- ① Authentication（认证）：身份鉴别，只有正确的账号才能通过认证。

- ② Authorization（授权）：判断用户是否有权限对访问的资源执行特定的动作。

- ③ Admission Control（准入控制）：用于补充授权机制以实现更加精细的访问控制功能。

### [](#权限流程控制)权限流程控制

用户携带令牌或者证书给 Kubernetes 的 api-server 发送请求，要求修改集群资源。

Kubernetes 开始认证。

Kubernetes 认证通过之后，会查询用户的授权（有哪些权限）。

用户执行操作的过程中（操作 CPU、内存、硬盘、网络……），利用准入控制来判断是否可以执行这样的操作。

## [](#认证管理)认证管理

### [](#k8s的客户端认证方式)k8s的客户端认证方式

Kubernetes 集群安全的关键点在于如何识别并认证客户端身份，它提供了 3 种客户端身份认证方式：

**① HTTP Base 认证：** 

- 通过 `用户名+密码` 的方式进行认证。

- 这种方式是把 `用户名:密码` 用 BASE64 算法进行编码后的字符串放在 HTTP 请求中的 Header 的 Authorization 域里面发送给服务端。服务端收到后进行解码，获取用户名和密码，然后进行用户身份认证的过程。

**② HTTP Token 认证：** 

- 通过一个 Token 来识别合法用户。

- 这种认证方式是用一个很长的难以被模仿的字符串–Token 来表明客户端身份的一种方式。每个 Token 对应一个用户名，当客户端发起 API 调用请求的时候，需要在 HTTP 的 Header 中放入 Token，API Server 接受到 Token 后会和服务器中保存的 Token 进行比对，然后进行用户身份认证的过程。

**③ HTTPS 证书认证：** 

- 基于 CA 根证书签名的双向数字证书认证方式。

- 这种认证方式是安全性最高的一种方式，但是同时也是操作起来最麻烦的一种方式。

### [](#总结)总结

**Kubernetes 允许同时配置多种认证方式，只要其中任意一种方式认证通过即可。**

## [](#授权管理)授权管理

授权发生在认证成功之后，通过认证就可以知道请求用户是谁，然后 Kubernetes 会根据事先定义的授权策略来决定用户是否有权限访问，这个过程就称为授权。

每个发送到 API Server 的请求都带上了用户和资源的信息：比如发送请求的用户、请求的路径、请求的动作等，授权就是根据这些信息和授权策略进行比较，如果符合策略，则认为授权通过，否则会返回错误。

### [](#API-Server目前支持的几种授权策略)API Server目前支持的几种授权策略

**AlwaysDeny**：表示拒绝所有请求，一般用于测试。 

**AlwaysAllow**：允许接收所有的请求，相当于集群不需要授权流程（Kubernetes 默认的策略）。 

**ABAC**：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制。 

**Webhook**：通过调用外部REST服务对用户进行授权。 

**Node**：是一种专用模式，用于对 kubelet 发出的请求进行访问控制。 

**RBAC**：基于角色的访问控制（ kubeadm 安装方式下的默认选项）。

**证明：kubeadm 安装方式的默认授权策略**

1
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -i rbac

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-07 125451.png)

### [](#RBAC)RBAC

**RBAC（Role Based Access Control）：基于角色的访问控制，主要是在描述一件事情：给哪些对象授权了哪些权限。**

**RBAC 模型：**

![](C:\Users\32625\OneDrive\图片\k8s\3 (1).png)

Kubernetes 中的 RBAC 也是基于 RBAC 模型扩展的，涉及到如下的概念： 

- 对象：User、Group、ServiceAccount。

- 角色：代表一组定义在资源上的可操作的动作（权限）的集合。

- 绑定：将定义好的角色和用户绑定在一起，也可以理解为分配角色。

![](C:\Users\32625\OneDrive\图片\k8s\4.png)

RBAC 引入了 4 个顶级资源对象： 

- Role：角色，用于指定一组权限，限定名称空间下的权限。

- ClusterRole：集群角色，用于指定一组权限，限定集群范围下的权限。

- RoleBinding：角色绑定，用于将角色 Role（权限的集合）赋予给对象（User、Group、ServiceAccount）。

- ClusterRoleBinding：集群角色绑定，用于将集群角色 Role（权限的集合）赋予给对象（User、Group、ServiceAccount）。

**说明：为什么 Kubernetes 要设计 Role 、 ClusterRole ？**

**答：**有些资源对象本身就不是 namespace（名称空间）的 ，所以 Kubernetes 增加了 ClusterRole，并且 ClusterRole 也可以管理名称空间下的资源对象。

#### [](#Role-和-ClusterRole资源清单文件)Role 和 ClusterRole资源清单文件

● 一个角色就是一组权限的集合，这里的权限都是许可形式的（白名单）。
● Role 的资源清单文件： 

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
# 名称空间角色
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: xudaxian-role
  namespace: default # 所属的名称空间
rules: # 当前角色的规则
- apiGroups: [&quot;&quot;] # &quot;&quot; 标明 core API 组，默认留空即可。
  resources: [&quot;pods&quot;] 
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;]

Role  只能对名称空间（namespace）进行授权，所以需要指定名称空间（namespace）。 

rules 中的参数说明： 

- apiGroups：`[&quot;&quot;]`，默认留空即可。

- resources：支持的资源对象列表，通过 `kubectl api-resources` 查看

1
kubectl api-resources 

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-07 132153.png)

- verbs：对资源对象的操作方法列表，通过 `kubectl api-resources -o wide` 查看。

![](C:\Users\32625\OneDrive\图片\屏幕快照\屏幕截图 2023-06-07 132407.png)

**ClusterRole 的资源清单文件**

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
# 集群角色
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: xudaxian-clusterrole
rules:
- apiGroups: [&quot;&quot;] # &quot;&quot; 标明 core API 组，默认留空即可。
  resources: [&quot;namespaces&quot;]
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;]
  
  
  # 注意：ClusterRole  不需要设置 namespace。

#### [](#RoleBinding-和-ClusterRoleBinding-资源清单文件)RoleBinding 和 ClusterRoleBinding 资源清单文件

- 角色绑定用来把一个角色绑定到一个目标对象上，绑定目标可以是 User、Group 或者 ServiceAccount 。

[](#RoleBinding-资源清单文件：)RoleBinding  资源清单文件：1
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
# 账号和角色绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: xudaxian-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: xudaxian # &quot;name&quot; 是区分大小写的
roleRef:
  kind: Role
  name: xudaxian-role
  apiGroup: rbac.authorization.k8s.io

[](#ClusterRoleBinding-资源清单文件：)ClusterRoleBinding 资源清单文件：1
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
# 账号和集群角色绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: xudaxian-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: xudaxian # &quot;name&quot; 是区分大小写的
  namespace: default # 如果资源是某个 namespace 下的，那么就需要设置 namespace
roleRef:
  kind: ClusterRole
  name: xudaxian-clusterrole
  apiGroup: rbac.authorization.k8s.io

#### [](#实战（RBAC）)实战（RBAC）

**创建 ServiceAccount 的时候，系统会在底层默认一个含 ServiceAccount 名称的 Secret 。**

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
54
55
56
# 名称空间角色
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: xudaxian-role
  namespace: default # 所属的名称空间
rules: # 当前角色的规则
- apiGroups: [&quot;&quot;] # &quot;&quot; 标明 core API 组，默认留空即可。
  resources: [&quot;pods&quot;] # 指定能操作的资源 ，通过 kubectl api-resources 查看即可。
  # resourceNames: [&quot;&quot;] #  指定只能操作某个名字的资源
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;] # 操作动作，通过 kubectl api-resources -o wide 查看即可。 
---
# 集群角色
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: xudaxian-clusterrole
rules:
- apiGroups: [&quot;&quot;] # &quot;&quot; 标明 core API 组，默认留空即可。
  resources: [&quot;namespaces&quot;]
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;]
---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: xudaxian # ServiceAccount 的名称
  namespace: default
---
# 账号和角色绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: xudaxian-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: xudaxian # &quot;name&quot; 是区分大小写的
roleRef:
  kind: Role
  name: xudaxian-role
  apiGroup: rbac.authorization.k8s.io
---
# 账号和集群角色绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: xudaxian-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: xudaxian # &quot;name&quot; 是区分大小写的
  namespace: default # 如果资源是某个 namespace 下的，那么就需要设置 namespace
roleRef:
  kind: ClusterRole
  name: xudaxian-clusterrole
  apiGroup: rbac.authorization.k8s.io

**温馨提示：**

**① 动态供应（NFS）、DashBoard 等底层都使用了 Role、ClusterRole、RoleBinding、ClusterRoleBinding 。**

**② 我们可以创建一个 ServiceAccount ，并将自定义的 ServiceAccount 绑定 cluster-admin （ClusterRole，Kubernetes 底层提供的），然后通过暴露的 [API](https://kubernetes.io/zh/docs/tasks/administer-cluster/access-cluster-api/) 进行 Kubernetes 管理平台的开发（如：Kubersphere 等）。**

## [](#准入控制)准入控制

通过了前面的认证和授权之后，还需要经过准入控制通过之后，API Server 才会处理这个请求。

准入控制是一个可配置的控制器列表，可以通过在 API Server 上通过命令行设置选择执行哪些注入控制器。

1
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds

- 只有当所有的注入控制器都检查通过之后，API Server 才会执行该请求，否则返回拒绝。

### [](#当前可配置的-Admission-Control)当前可配置的 Admission Control

**AlwaysAdmit：允许所有请求。** 

**AlwaysDeny：禁止所有请求，一般用于测试。** 

**AlwaysPullImages：在启动容器之前总去下载镜像。** 

**DenyExecOnPrivileged：它会拦截所有想在 Privileged Container 上执行命令的请求。** 

**ImagePolicyWebhook：这个插件将允许后端的一个 Webhook 程序来完成 admission control 的功能。** 

**Service Account：实现 ServiceAccount 实现了自动化。** 

**SecurityContextDeny：这个插件将使用 SecurityContext 的 Pod 中的定义全部失效。** 

**ResourceQuota：用于资源配额管理目的，观察所有请求，确保在 namespace 上的配额不会超标。** 

**LimitRanger：用于资源限制管理，作用于 namespace 上，确保对 Pod 进行资源限制。** 

**InitialResources：为未设置资源请求与限制的 Pod，根据其镜像的历史资源的使用情况进行设置。** 

**NamespaceLifecycle：如果尝试在一个不存在的 namespace 中创建资源对象，则该创建请求将被拒绝。当删除一个 namespace 时，系统将会删除该 namespace 中所有对象。** 

**DefaultStorageClass：为了实现共享存储的动态供应，为未指定 StorageClass 或 PV 的 PVC 尝试匹配默认 StorageClass，尽可能减少用户在申请 PVC 时所需了解的后端存储细节。** 

**DefaultTolerationSeconds：这个插件为那些没有设置 forgiveness tolerations 并具有 notready:NoExecute 和 unreachable:NoExecute 两种taints的Pod设置默认的`容忍`时间，为 5min 。** 

**PodSecurityPolicy：这个插件用于在创建或修改 Pod 时决定是否根据 Pod 的 security context 和可用的 PodSecurityPolicy 对 Pod 的安全策略进行控制**

                
