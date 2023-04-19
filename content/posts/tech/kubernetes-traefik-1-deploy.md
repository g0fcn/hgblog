---
title: "traefik系列之一 | 简介、部署和配置" 
date: 2023-04-17
lastmod: 2023-04-17
tags: 
- traefik
- kubernetes
keywords:
- kubernetes
- traefik
- ingress
description: "kubernetes 中 Traefik ingress 的简介、部署及配置。" 
cover:
    image: "https://image.lvbibir.cn/blog/traefik.png"
    hidden: true
    hiddenInSingle: true
---

# 0. 前言

基于 `centos7.9`，`docker-ce-20.10.18`，`kubelet-1.22.3-0`， `traefik-2.9.10`

参考：https://www.cuiliangblog.cn/detail/section/29427812

# 1. 简介

## 1.1 Traefik 简介

Traefik 是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 ([Docker](https://www.docker.com/), [Swarm](https://docs.docker.com/swarm), [Kubernetes](http://kubernetes.io/), [Marathon](https://mesosphere.github.io/marathon/), [Mesos](https://github.com/apache/mesos), [Consul](https://www.consul.io/), [Etcd](https://coreos.com/etcd/), [Zookeeper](https://zookeeper.apache.org/), [BoltDB](https://github.com/boltdb/bolt), Rest API, file…) 来自动化、动态的应用它的配置文件设置。

它是一个边缘路由器，它会拦截外部的请求并根据逻辑规则选择不同的操作方式，这些规则决定着这些请求到底该如何处理。Traefik 提供自动发现能力，会实时检测服务，并自动更新路由规则。

![traefik-architecture](https://image.lvbibir.cn/blog/traefik-architecture.png)

## 1.2 Traefik 核心组件



![img](https://image.lvbibir.cn/blog/v2-b67cb6ed0ac457009296459fe974cef4_r.jpg)

从上图可知，当请求 Traefik 时，请求首先到 `entrypoints`，然后分析传入的请求，查看他们是否与定义的 `Routers` 匹配。如果匹配，则会通过一系列 `middlewares` 处理，再到 `traefikServices` 上做流量转发，最后请求到 `kubernetes的services上` 。

这就涉及到以下几个重要的核心组件:

- **[Providers](https://doc.traefik.io/traefik/providers/overview/)** 是基础组件，Traefik 的配置发现是通过它来实现的，它可以是协调器，容器引擎，云提供商或者键值存储。Traefik 通过查询 `Providers` 的 `API` 来查询路由的相关信息，一旦检测到变化，就会动态的更新路由。 
- **[Entrypoints](https://doc.traefik.io/traefik/routing/entrypoints/)** 是 `Traefik` 的网络入口，它定义接收请求的接口，以及是否监听TCP或者UDP。
- **[Routers](https://doc.traefik.io/traefik/routing/routers/)** 主要用于分析请求，并负责将这些请求连接到对应的服务上去，在这个过程中，Routers还可以使用Middlewares来更新请求，比如在把请求发到服务之前添加一些Headers。 
- **[Services](https://doc.traefik.io/traefik/routing/services/)** 负责配置如何到达最终将处理传入请求的实际服务。 
- **[Middlewares](https://doc.traefik.io/traefik/middlewares/overview/)** 用来修改请求或者根据请求来做出一些判断（authentication, rate limiting, headers, ...），中间件被附件到路由上，是一种在请求发送到你的**服务**之前（或者在服务的响应发送到客户端之前）调整请求的一种方法。

## 1.3 Traefik CRD资源

> 参考文档：https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/

traefik通过自定义资源实现了对traefik资源的创建和管理，支持的crd资源类型如下所示：

| kind                                                         | 功能                        |
| ------------------------------------------------------------ | --------------------------- |
| [IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-ingressroute) | HTTP路由配置                |
| [Middleware](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-middleware) | HTTP中间件配置              |
| [TraefikService](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-traefikservice) | HTTP负载均衡/流量复制配置   |
| [IngressRouteTCP](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-ingressroutetcp) | TCP路由配置                 |
| [MiddlewareTCP](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-middlewaretcp) | TCP中间件配置               |
| [IngressRouteUDP](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-ingressrouteudp) | UDP路由配置                 |
| [TLSOptions](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-tlsoption) | TLS连接参数配置             |
| [TLSStores](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-tlsstore) | TLS存储配置                 |
| [ServersTransport](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-serverstransport) | traefik与后端之间的传输配置 |

# 2. Traefik 部署

traefik 是支持 helm 部署的，但是查看 helm 包的 value.yaml 配置发现总共有 500 多行配置，当需要修改配置项或者对 traefik 做一下自定义配置时，并不灵活。如果只是使用 traefik 的基础功能，推荐使用 helm 部署。如果想深入研究使用 traefik 的话，推荐使用自定义方式部署。

## 2.1 crd rbac serviceaccount

crd

```bash
[root@k8s-node1 ~]# mkdir /opt/traefik
[root@k8s-node1 ~]# cd /opt/traefik
[root@k8s-node1 traefik]# wget https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
[root@k8s-node1 traefik]# kubectl apply -f kubernetes-crd-definition-v1.yml
```

rbac

```bash
[root@k8s-node1 traefik]# wget https://raw.githubusercontent.com/traefik/traefik/v2.9/docs/content/reference/dynamic-configuration/kubernetes-crd-rbac.yml
[root@k8s-node1 traefik]# kubectl apply -f kubernetes-crd-rbac.yml
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created
```

serviceaccount.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: traefik
  name: traefik-ingress-controller
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: traefik
```

## 2.2 configmap 

> 在 Traefik 中有三种方式定义静态配置：在配置文件中、在命令行参数中、通过环境变量传递，由于 Traefik 配置很多，通过 CLI 定义不是很方便，一般时候选择将其配置选项放到配置文件中，然后存入 ConfigMap，将其挂入 traefik 中。参考文档：https://doc.traefik.io/traefik/getting-started/configuration-overview/

`configmap.yml` 文件内容：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-config
  namespace: traefik
data:
  traefik.yaml: |-
    global:
      checkNewVersion: false    # 周期性的检查是否有新版本发布
      sendAnonymousUsage: false # 周期性的匿名发送使用统计信息
    serversTransport:
      insecureSkipVerify: true  # Traefik忽略验证代理服务的TLS证书
    api:
      insecure: true            # 允许HTTP 方式访问API
      dashboard: true           # 启用Dashboard
      debug: false              # 启用Debug调试模式
    metrics:
      prometheus:               # 配置Prometheus监控指标数据，并使用默认配置
        addRoutersLabels: true  # 添加routers metrics
        entryPoint: "metrics"   # 指定metrics监听地址
    entryPoints:
      web:
        address: ":80"          # 配置80端口，并设置入口名称为 web
        forwardedHeaders: 
          insecure: true        # 信任所有的forward headers
      websecure:
        address: ":443"         # 配置443端口，并设置入口名称为 websecure
        forwardedHeaders: 
          insecure: true
      traefik:
        address: ":9000"        # 配置9000端口为 dashboard 的端口，不设置默认值为 8080
      metrics:
        address: ":9101"        # 配置9101端口，作为metrics收集入口
      tcpep:
        address: ":9200"        # 配置9200端口，作为tcp入口
      udpep:
        address: ":9300/udp"    # 配置9300端口，作为udp入口
    providers:
      kubernetesCRD:            # 启用Kubernetes CRD方式来配置路由规则
        ingressClass: ""        # 指定traefik的ingressClass名称
        allowCrossNamespace: true   #允许跨namespace
        allowEmptyServices: true    #允许空endpoints的service
    log:
      filePath: "/etc/traefik/logs/traefik.log" # 设置调试日志文件存储路径，如果为空则输出到控制台
      level: "DEBUG"            # 设置调试日志级别
      format: "common"          # 设置调试日志格式
    accessLog:
      filePath: "/etc/traefik/logs/access.log" # 设置访问日志文件存储路径，如果为空则输出到控制台
      format: "common"          # 设置访问调试日志格式
      bufferingSize: 0          # 设置访问日志缓存行数
      filters:
        # statusCodes: ["200"]  # 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     # 设置代理访问重试失败时，保留访问日志
        minDuration: 20         # 设置保留请求时间超过指定持续时间的访问日志
      fields:                   # 设置访问日志中的字段是否保留（keep保留、drop不保留）
        defaultMode: keep       # 设置默认保留访问日志字段
        names:                  # 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop
          StartUTC: drop        # 禁用日志timestamp使用UTC
        headers:                # 设置Header中字段是否保留
          defaultMode: keep     # 设置默认保留Header中字段
          names:                # 针对Header中特别字段特别配置保留模式
            # User-Agent: redact# 可以针对指定agent
            Authorization: drop
            Content-Type: keep
```

设置节点 label，用于控制在哪些节点部署 Traefik，这里我们使用 k8s-node1(master) 节点作为边缘节点部署

```bash
kubectl label node k8s-node1  IngressProxy=true
```

## 2.3 daemonset service

> 使用 DeamonSet 或者 Deployment 均可部署，此处使用 DaemonSet 方式部署 Traefik，调度至IngressProxy=true 的那台 master 边缘节点，并使用 host 网络模式，提高网络入口的网络性能

`daemonset.yml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik
  namespace: traefik
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 5       # 等待容器优雅退出的时长
      containers:
      - name: traefik
        image: traefik:v2.9.10
        env:
        - name: KUBERNETES_SERVICE_HOST       # 手动指定k8s api,避免网络组件不稳定。
          value: "1.1.1.1"
        - name: KUBERNETES_SERVICE_PORT_HTTPS # API server端口
          value: "6443"
        - name: KUBERNETES_SERVICE_PORT       # API server端口
          value: "6443"
        - name: TZ                            # 指定时区
          value: "Asia/Shanghai"
        ports:
          - name: web
            containerPort: 80
            hostPort: 80                      # 将容器端口绑定所在服务器的 80 端口
          - name: websecure
            containerPort: 443
            hostPort: 443                     # 将容器端口绑定所在服务器的 443 端口
          - name: dashboard
            containerPort: 9000               # Traefik Dashboard 端口
          - name: metrics
            containerPort: 9101               
            hostPort: 9101                    # metrics 端口，由于 node_exporter 占据了 9100 端口，改到 9101  
          - name: tcpep
            containerPort: 9200               # tcp端口
          - name: udpep
            containerPort: 9300               # udp端口
        securityContext:                      # 只开放网络权限  
          capabilities:
            drop:
              - ALL
            add:
              - NET_BIND_SERVICE
        args:
          - --configfile=/etc/traefik/config/traefik.yaml
        volumeMounts:
        - mountPath: /etc/traefik/config
          name: config
        - mountPath: /etc/traefik/logs
          name: logdir
        - mountPath: /etc/localtime
          name: timezone
          readOnly: true
        resources:
          requests:
            memory: "5Mi"
            cpu: "10m" 
          limits:
            memory: "256Mi"
            cpu: "1000m"
      volumes:
        - name: config                         # traefik配置文件
          configMap:
            name: traefik-config 
        - name: logdir                         # traefik日志目录
          hostPath:
            path: /var/log/traefik
            type: "DirectoryOrCreate"
        - name: timezone                       # 挂载时区文件
          hostPath:
            path: /etc/localtime
            type: File
      tolerations:                             # 设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      hostNetwork: true                        # 开启host网络，提高网络入口的网络性能
      dnsConfig:                               # 开启host网络后，默认使用主机的dns配置，添加coredns配置
        nameservers:
          - 10.96.0.10
        searches:
          - traefik.svc.cluster.local 
          - svc.cluster.local 
          - cluster.local
        options:
          - name: ndots
            value: "5"
      nodeSelector:                            # 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"                   # 调度至IngressProxy: "true"的节点
```

service  `kubectl apply -f service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace: traefik
spec:
  type: NodePort    ## 官网示例为LoadBalancer,为方便演示，此处改为NodePort
  selector:
    app: traefik
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 80
    - name: websecure
      protocol: TCP
      port: 443
      targetPort: 443
    - name: dashboard
      protocol: TCP
      port: 9000
      targetPort: 9000
    - name: metrics
      protocol: TCP
      port: 9101
      targetPort: 9101
    - name: tcpep
      protocol: TCP
      port: 9200
      targetPort: 9200
    - name: udpep
      protocol: UDP
      port: 9300
      targetPort: 9300
```

## 2.4 验证

```bash
# traefik pod 映射的主机端口
[root@k8s-node1 traefik]# ss -tnlp | grep traefik
LISTEN 0  128  [::]:9000   [::]:*   users:(("traefik",pid=36505,fd=11))
LISTEN 0  128  [::]:9101   [::]:*   users:(("traefik",pid=36505,fd=12))
LISTEN 0  128  [::]:9200   [::]:*   users:(("traefik",pid=36505,fd=13))
LISTEN 0  128  [::]:80     [::]:*   users:(("traefik",pid=36505,fd=8))
LISTEN 0  128  [::]:443    [::]:*   users:(("traefik",pid=36505,fd=9))
[root@k8s-node1 traefik]# kubectl get pod,cm,sa,svc -n traefik |grep traefik
pod/traefik-c29hs   1/1     Running   0          4m36s
configmap/traefik-config     1      24m
serviceaccount/traefik-ingress-controller   1         36m
service/traefik   NodePort   10.109.10.18   <none>  80:57197/TCP,443:36218/TCP,9000:47179/TCP,9101:25785/TCP,9200:44736/TCP,9300:13220/UDP   2m14s
```

可以直接通过 http://1.1.1.1:47179 或者 http://1.1.1.1:9000 访问到 dashboard

![image-20230419105303492](https://image.lvbibir.cn/blog/image-20230419105303492.png)

## 2.5 其他配置

### 2.5.1 强制使用TLS v1.2+

> 如今，TLS v1.0 和 v1.1 因为存在安全问题，现在已被弃用。为了保障系统安全，所有入口路由都应该强制使用TLS v1.2 或更高版本。
>
> 参考文档：https://doc.traefik.io/traefik/user-guides/crd-acme/#force-tls-v12

```bash
[root@k8s-node1 traefik]# tee traefik-tlsoption.yml <<-'EOF'
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: default
  namespace: traefik
spec:
  minVersion: VersionTLS12
  cipherSuites:
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384   # TLS 1.2
    - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305    # TLS 1.2
    - TLS_AES_256_GCM_SHA384                  # TLS 1.3
    - TLS_CHACHA20_POLY1305_SHA256            # TLS 1.3
  curvePreferences:
    - CurveP521
    - CurveP384
  sniStrict: true
EOF
[root@k8s-node1 traefik]# kubectl apply -f traefik-tlsoption.yml
tlsoption.traefik.containo.us/default created
```

### 2.5.2 日志切割

> 官方并没有日志轮换的功能，但是traefik收到USR1信号后会重建日志文件，因此可以通过logrotate实现日志轮换
>
> 参考文档：https://doc.traefik.io/traefik/observability/logs/

```bash
mkdir -p /etc/logrotate.d/traefik
tee /etc/logrotate.d/traefik/config <<-'EOF'
/var/log/traefik/*.log {
  daily
  rotate 15
  missingok
  notifempty
  compress
  dateext
  dateyesterday
  dateformat .%Y-%m-%d
  create 0644 root root
  postrotate
   docker kill --signal="USR1" $(docker ps | grep traefik |grep -v pause| awk '{print $1}')
  endscript
 }
EOF
```

创建定时任务

```bash
crontab -e
0 0 * * * /usr/sbin/logrotate -f /etc/logrotate.d/traefik/config >/dev/null 2>&1
```

## 2.6 多控制器

有的业务场景下可能需要在一个集群中部署多个 traefik，例如：避免单个traefik配置规则过多导致加载处理缓慢。每个namespace部署一个traefik。或者traefik生产与测试环境区分等场景，需要不同的实例控制不同的 IngressRoute 资源对象，要实现该功能有两种方法

### 2.6.1 annotations 注解筛选

> 参考文档：https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass

首先在traefik配置文件中的providers下增加Ingressclass参数，指定具体的值

```yaml
    providers:
      kubernetesCRD:            # 启用Kubernetes CRD方式来配置路由规则
        ingressClass: "traefik-v2.9" # 指定traefik的ingressClass实例名称
        allowCrossNamespace: true   #允许跨namespace
        allowEmptyServices: true    #允许空endpoints的service
```

接下来在 IngressRoute 资源对象中的 annotations 参数中添加 `kubernetes.io/ingress.class: traefik-v2.9` 即可

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
  namespace: traefik
  annotations:
    kubernetes.io/ingress.class: traefik-v2.9 #  因为静态配置文件指定了ingressclass，所以这里的annotations 要指定，否则访问会404
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.test.com`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
          namespace: traefik
```

### 2.6.2 label 标签选择器筛选

> 参考文档：https://doc.traefik.io/traefik/providers/kubernetes-crd/#labelselector

首先在traefik配置文件中的providers下增加labelSelector参数，指定具体的标签键值。

```yaml
    providers:
      kubernetesCRD:            # 启用Kubernetes CRD方式来配置路由规则
        # ingressClass: "traefik-v2.9"    # 指定traefik的ingressClass名称
        labelSelector: "app=traefik-v2.9" # 通过标签选择器指定traefik标签 
        allowCrossNamespace: true   #允许跨namespace
        allowEmptyServices: true    #允许空endpoints的service
```

然后在 IngressRoute 资源对象中添加labels标签选择器，选择 `app: traefik-v2.9` 这个标签即可

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
  labels:     # 通过标签选择器，该IngressRoute资源由配置了app=traefik-v2.9的traefik处理
    app: traefik-v2.9
  # annotations:
    # kubernetes.io/ingress.class: traefik-v2.9
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.test.com`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
          namespace: traefik
```

