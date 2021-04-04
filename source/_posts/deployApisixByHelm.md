---
title: 通过 Helm 部署 Apisix
date: 2021-04-04 15:01:07
tags: [Helm, K8S, Etcd, Apisix, 网关, 安装, 配置]
category: 运维
---

下面将介绍，在 K8S 集群中通过 Helm Chart 搭建部署一个 可水平伸缩(HPA)架构的 Apisix 网关 ,并附带可视化管理面板

# 环境

- 腾讯云 kubernetes 1.18.4-tke.8
- Bitnami Etcd-6.1.3 [Chart 下载](https://charts.bitnami.com/bitnami/etcd-6.1.3.tgz)
- 官方 Apisix Helm Chart 0.2.1
- 官方 Apisix-dashboard Helm Chart 0.1.1

# 具体步骤

## 准备数据层 Etcd

Apisix 的数据库使用的是 Etcd，所以我们需要先部署一个 Etcd 集群，关于 Etcd 集群在 K8S 中的部署，请戳[这里](http://www.lrvinye.cn/blog/2021/04/04/deployEtcdByHelm)

## 打包官方的Chart

APISIX官方的HelmChart地址 → [https://github.com/apache/apisix-helm-chart]

官方的仓库中已经准备了 Apisix 、Apisix-dashboard 、Apisix-ingress 这三个Chart的描述文件

我们需要将仓库克隆后自行压缩为`tgz`格式的压缩文件

### Apisix Chart 自定义

官方给出的Chart描述文件，如果之间打包则无法提供腾讯云控制台正常可视化部署，其只能提供Helm的命令行进行部署，我们这里需要对其进行一些修改，让我们自己打包后可用。

在 `/charts/apisix/Chart.yaml` 文件的最后一段

```yaml
### /charts/apisix/Chart.yaml ...

dependencies:
  - name: etcd
    version: 5.2.1
    repository: https://charts.bitnami.com/bitnami
    condition: etcd.enabled
  - name: apisix-dashboard
    version: 0.1.0
    repository: file://../apisix-dashboard
    condition: dashboard.enabled
    alias: dashboard
  - name: apisix-ingress-controller
    version: 0.2.0
    repository: file://../apisix-ingress-controller
    condition: ingress-controller.enabled
    alias: ingress-controller
```

这是一段关于部署时的依赖 Helm Chart 的描述。由于我们不使用命令行进行部署（我们将自行部署相关的Chart），因此需要删去后再打包。
其它两个Chart暂无需要修改的地方。

---

打包后，我们得到了三个Chart的tgz文件。
分别是 `apisix-0.2.1.tgz`、`apisix-dashboard-0.1.1.tgz`、`apisix-ingress-controller-0.3.0.tgz`


## 自定义 Values

> 下面的 Values 配置中，仅指出需要根据[模板](#Values原模板配置)的原配置信息进行自定义的选项！！！

### **Apisix参数**

这里我们将对部分需要自定义的参数进行解析

```yaml
## 集群副本数
replicaCount: 1


## 开放网关配置
gateway:

  ## 部署的Service类型

  type: NodePort
  # type: LoadBalancer
  # annotations:
  #   service.beta.kubernetes.io/aws-load-balancer-type: nlb

  ## 网关的入口
  http:
    enabled: true
    servicePort: 80
    containerPort: 9080

    ## ssl入口端口
  tls:
    enabled: false
    servicePort: 443
    containerPort: 9443
    http2:
      enabled: true

  ## ingress 端口配置
  ingress:
    enabled: false
    annotations:
      # kubernetes.io/ingress.class: nginx
      # kubernetes.io/tls-acme: "true"
    hosts:
      - host: apisix.local
        paths: []
    tls: []
  #  - secretName: apisix-tls
  #    hosts:
  #      - chart-example.local

## etcd 配置
etcd:
  # install etcd(v3) by default, set false if do not want to install etcd(v3) together
  ## 这里我们已经有自己部署的etcd了，所以不使用它自带的
  enabled: false

  ## 指定etcd的连接配置，这里使用你自己的etcd连接地址
  host:
    - http://etcd:2379 # host or ip e.g. http://172.20.128.89:2379
  prefix: "/apisix"
  timeout: 30

  # if etcd.enabled is true, set more values of bitnami/etcd helm chart
  auth:
    rbac:
      # No authentication by default
      enabled: false

## 这里的dashboard和ingress，我们都自行部署，所以不使用它自带的部署
dashboard:
  enabled: false
ingress-controller:
  enabled: false

# 资源
resources:
  limits:
    cpu: 250m
    memory: 256Mi
  requests:
    cpu: 50m
    memory: 128Mi

## HPA配置
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

```

> 为什么不使用自带的部署信息（etcd、dashboard...）？ <br/> 
由于其自带的部署需要依赖于helm的命令行部署，而我们这里将使用打包tgz的形式部署，所以其自带的附属工具部署将在我们这种部署中无效，因此我们选择自行部署

### **可视化面板配置**

> 通过Values文件我们可以发现，apisix-dashboard其实是通过对etcd的数据进行管理，以达到管理apisix的目的

```yaml
# 集群副本数
replicaCount: 1


config:
  conf:
    ## 这里为固定写法 ，官方解释：仅当conf.listen.host为0.0.0.0时，外部网络才能访问容器中的服务。
    listen:
      host: 0.0.0.0
      ## GUI端口
      port: 9000
    etcd:
      # 定义etcd的连接信息， Supports defining multiple etcd host addresses for an etcd cluster
      endpoints:
        - etcd:2379
      # Etcd basic auth 认证配置，这里是无
      username: ~
      password: ~
    log:
      errorLog:
        level: warn
        filePath: /dev/stderr
      accessLog:
        filePath: /dev/stdout

  authentication:
    secert: secert
    expireTime: 3600
    ## GUI登录用户，可用写多个
    users:
      - username: lrvinye
        password: xy000409

## service部署
service:
  type: ClusterIP
  port: 9000

## ingress 连接配置
ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: apisix-dashboard.local
      paths: []
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

## 资源
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

## HPA配置
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

```

---

以上的相关配置，笔者已经整理好，戳[这里](https://github.com/ibootcloud/apisix-helm-chart)

Chart 准备完毕即可按顺序进行部署，部署完毕后，访问上面dashboard部署时指定的GUI端口即可登录可视化Web管理页面，
网关的入口将作为正式的服务入口
