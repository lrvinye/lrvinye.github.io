---
title: 通过 Helm 部署 Redis 集群
date: 2021-03-31 18:50:45
tags: [Helm ,K8S, Redis, Bitnami, 安装, 配置]
category: 运维
---

下面将介绍，在K8S集群中通过 Bitnami 的 Helm Chart 搭建部署一个3节点的 一主二从架构 Redis集群

# 环境

- 腾讯云 kubernetes 1.18.4-tke.6
- Redis™ Bitnami 12.10.0 [Chart下载](https://charts.bitnami.com/bitnami/redis-12.10.0.tgz)

# 具体步骤

## 准备持久化

这里使用腾讯云的云硬盘作为持久化卷
1. 手动创建 所需的云硬盘，这里我们有3个节点，需要3个云硬盘，每个10GB
2. 创建与云硬盘对应的 `storageClass` 由于是数据库持久化型的数据，我们应该在生产环境中指定回收策略为 `Retain`，卷绑定模式应该设置为`立即绑定`
3. 创建指定`storageClass`的 PV卷 并在控制台中与上一步创建的云硬盘操作进行绑定
4. PV 需要打上labels

主节点一个

```yaml
labels: 
  app: redis-master
```

从节点两个
```yaml
labels: 
  app: redis-slave
```

这里的label将在下面的步骤中用来对PVC进行匹配

## 准备 Values.yaml 

> 下面的Values配置中，仅指出需要根据[模板](#Values原模板配置)的原配置信息进行自定义的选项！！！

### **全局参数**
- 指定 `storageClass` 为上面我们已经创建的SC
```yaml
global:
  storageClass: cbs-db-sc-test
  redis:
    password: xxx123456
```

### **集群设置**
启用集群并设置`slaveCount`节点数为2

```yaml
cluster:
  enabled: true
  slaveCount: 2
```

---

**为何不手动创建PVC并在下面的选项中设置`existingClaim` ?** <br/>

```yaml
persistence:
  ## A manually managed Persistent Volume and Claim
  ## Requires persistence.enabled: true
  ## If defined, PVC must be created manually before volume will be bound
  ##
  existingClaim:
```

如果在此处指定了`existingClaim`,那么我们的3个节点将同时挂载在一个PVC上，这将影响我们的持久化数据隔离

---

### **节点设置**

```yaml
master:
    #这里根据自己的可用资源合理配置
  resources:
    requests:
      memory: 100Mi
      cpu: 50m
    limits:
      memory: 256Mi
      cpu: 150m
  nodeSelector: 
  #这里是用来约束节点的所在node，因为腾讯云的云硬盘只能挂载同一可用区下的CVM
    topology.com.tencent.cloud.csi.cbs/zone: ap-guangzhou-4
  persistence:
    enabled: true
    path: /data
    subPath: ""
    accessModes:
      - ReadWriteOnce
      #最少10G，且为10的倍数
    size: 10Gi
    # 匹配创建的PV
    matchLabels:
      app: redis-master
slave:
    #这里根据自己的可用资源合理配置
  resources:
    requests:
      memory: 100Mi
      cpu: 50m
    limits:
      memory: 256Mi
      cpu: 150m
  nodeSelector: 
  #这里是用来约束节点的所在node，因为腾讯云的云硬盘只能挂载同一可用区下的CVM
    topology.com.tencent.cloud.csi.cbs/zone: ap-guangzhou-4
  persistence:
    enabled: true
    path: /data
    subPath: ""
    accessModes:
      - ReadWriteOnce
      #最少10G，且为10的倍数
    size: 10Gi
    # 匹配创建的PV
    matchLabels:
      app: redis-slave
```

### 初始化时目录权限配置

为了防止权限不足`Permission deny`
```yaml
volumePermissions:
  enabled: true
```

### redis.conf 配置

```yaml
configmap: |-
  # Enable AOF https://redis.io/topics/persistence#append-only-file
  # 这一项需为 no 否则无法进行主从复制
  appendonly no
  # Disable RDB persistence, AOF persistence already enabled.
  save ""
```

---


Chart 准备完毕即可进行部署


# Values原模板配置
```yaml
## Global Docker image parameters
## Please, note that this will override the image parameters, including dependencies, configured to use the global value
## Current available global Docker image parameters: imageRegistry and imagePullSecrets
##
global:
  # imageRegistry: myRegistryName
  # imagePullSecrets:
  #   - myRegistryKeySecretName
  # storageClass: myStorageClass
  redis: {}

## Bitnami Redis(TM) image version
## ref: https://hub.docker.com/r/bitnami/redis/tags/
##
image:
  registry: docker.io
  repository: bitnami/redis
  ## Bitnami Redis(TM) image tag
  ## ref: https://github.com/bitnami/bitnami-docker-redis#supported-tags-and-respective-dockerfile-links
  ##
  tag: 6.0.12-debian-10-r3
  ## Specify a imagePullPolicy
  ## Defaults to 'Always' if image tag is 'latest', else set to 'IfNotPresent'
  ## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
  ##
  pullPolicy: IfNotPresent
  ## Optionally specify an array of imagePullSecrets.
  ## Secrets must be manually created in the namespace.
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  ##
  # pullSecrets:
  #   - myRegistryKeySecretName

## String to partially override redis.fullname template (will maintain the release name)
##
# nameOverride:

## String to fully override redis.fullname template
##
# fullnameOverride:

## Cluster settings
##
cluster:
  enabled: true
  slaveCount: 2

## Use redis sentinel in the redis pod. This will disable the master and slave services and
## create one redis service with ports to the sentinel and the redis instances
##
sentinel:
  enabled: false
  ## Require password authentication on the sentinel itself
  ## ref: https://redis.io/topics/sentinel
  ##
  usePassword: true
  ## Bitnami Redis(TM) Sentintel image version
  ## ref: https://hub.docker.com/r/bitnami/redis-sentinel/tags/
  ##
  image:
    registry: docker.io
    repository: bitnami/redis-sentinel
    ## Bitnami Redis(TM) image tag
    ## ref: https://github.com/bitnami/bitnami-docker-redis-sentinel#supported-tags-and-respective-dockerfile-links
    ##
    tag: 6.0.12-debian-10-r0
    ## Specify a imagePullPolicy
    ## Defaults to 'Always' if image tag is 'latest', else set to 'IfNotPresent'
    ## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
    ##
    pullPolicy: IfNotPresent
    ## Optionally specify an array of imagePullSecrets.
    ## Secrets must be manually created in the namespace.
    ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
    ##
    # pullSecrets:
    #   - myRegistryKeySecretName
  masterSet: mymaster
  initialCheckTimeout: 5
  quorum: 2
  downAfterMilliseconds: 20000
  failoverTimeout: 18000
  parallelSyncs: 1
  port: 26379

  ## Delay seconds when cleaning nodes IPs
  ## When starting it will clean the sentinels IP (RESET "*") in all the nodes
  ## This is the delay time before sending the command to the next node
  ##
  cleanDelaySeconds: 5

  ## Additional Redis(TM) configuration for the sentinel nodes
  ## ref: https://redis.io/topics/config
  ##
  configmap:
  ## Enable or disable static sentinel IDs for each replicas
  ## If disabled each sentinel will generate a random id at startup
  ## If enabled, each replicas will have a constant ID on each start-up
  ##
  staticID: false
  ## Configure extra options for Redis(TM) Sentinel liveness and readiness probes
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#configure-probes)
  ##
  livenessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 5
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 5
  customLivenessProbe: {}
  customReadinessProbe: {}
  ## Redis(TM) Sentinel resource requests and limits
  ## ref: http://kubernetes.io/docs/user-guide/compute-resources/
  # resources:
  #   requests:
  #     memory: 256Mi
  #     cpu: 100m
  ## Redis(TM) Sentinel Service properties
  ##
  service:
    ##  Redis(TM) Sentinel Service type
    ##
    type: ClusterIP
    sentinelPort: 26379
    redisPort: 6379

    ## External traffic policy (when service type is LoadBalancer)
    ## ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    ##
    externalTrafficPolicy: Cluster

    ## Specify the nodePort value for the LoadBalancer and NodePort service types.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
    ##
    # sentinelNodePort:
    # redisNodePort:

    ## Provide any additional annotations which may be required. This can be used to
    ## set the LoadBalancer service type to internal only.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer
    ##
    annotations: {}
    labels: {}
    loadBalancerIP:

  ## Additional commands to run prior to starting Redis(TM) node with sentinel
  ##
  preExecCmds: ""

  ## An array to add extra env var to the sentinel node configurations
  ## For example:
  ## extraEnvVars:
  ##  - name: name
  ##    value: value
  ##  - name: other_name
  ##    valueFrom:
  ##      fieldRef:
  ##        fieldPath: fieldPath
  ##
  extraEnvVars: []

  ## ConfigMap with extra env vars:
  ##
  extraEnvVarsCM: []

  ## Secret with extra env vars:
  ##
  extraEnvVarsSecret: []

## Specifies the Kubernetes Cluster's Domain Name.
##
clusterDomain: cluster.local

networkPolicy:
  ## Specifies whether a NetworkPolicy should be created
  ##
  enabled: false

  ## The Policy model to apply. When set to false, only pods with the correct
  ## client label will have network access to the port Redis(TM) is listening
  ## on. When true, Redis(TM) will accept connections from any source
  ## (with the correct destination port).
  ##
  # allowExternal: true

  ## Allow connections from other namespaces. Just set label for namespace and set label for pods (optional).
  ##
  ingressNSMatchLabels: {}
  ingressNSPodMatchLabels: {}

serviceAccount:
  ## Specifies whether a ServiceAccount should be created
  ##
  create: false
  ## The name of the ServiceAccount to use.
  ## If not set and create is true, a name is generated using the fullname template
  ##
  name:
  ## Add annotations to service account
  # annotations:
  #   iam.gke.io/gcp-service-account: "sa@project.iam.gserviceaccount.com"

rbac:
  ## Specifies whether RBAC resources should be created
  ##
  create: false

  role:
    ## Rules to create. It follows the role specification
    # rules:
    #  - apiGroups:
    #    - extensions
    #    resources:
    #      - podsecuritypolicies
    #    verbs:
    #      - use
    #    resourceNames:
    #      - gce.unprivileged
    rules: []

## Redis(TM) pod Security Context
##
securityContext:
  enabled: true
  fsGroup: 1001
  ## sysctl settings for master and slave pods
  ##
  ## Uncomment the setting below to increase the net.core.somaxconn value
  ##
  # sysctls:
  # - name: net.core.somaxconn
  #   value: "10000"

## Container Security Context
## ref: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
##
containerSecurityContext:
  enabled: true
  runAsUser: 1001

## Use password authentication
##
usePassword: true
## Redis(TM) password (both master and slave)
## Defaults to a random 10-character alphanumeric string if not set and usePassword is true
## ref: https://github.com/bitnami/bitnami-docker-redis#setting-the-server-password-on-first-run
##
password: ""
## Use existing secret (ignores previous password)
# existingSecret:
## Password key to be retrieved from Redis(TM) secret
##
# existingSecretPasswordKey:

## Mount secrets as files instead of environment variables
##
usePasswordFile: false

## Persist data to a persistent volume (Redis(TM) Master)
##
persistence:
  ## A manually managed Persistent Volume and Claim
  ## Requires persistence.enabled: true
  ## If defined, PVC must be created manually before volume will be bound
  ##
  existingClaim:

# Redis(TM) port
redisPort: 6379

##
## TLS configuration
##
tls:
  # Enable TLS traffic
  enabled: false
  #
  # Whether to require clients to authenticate or not.
  authClients: true
  #
  # Name of the Secret that contains the certificates
  certificatesSecret:
  #
  # Certificate filename
  certFilename:
  #
  # Certificate Key filename
  certKeyFilename:
  #
  # CA Certificate filename
  certCAFilename:
  #
  # File containing DH params (in order to support DH based ciphers)
  # dhParamsFilename:

##
## Redis(TM) Master parameters
##
master:
  ## Redis(TM) command arguments
  ##
  ## Can be used to specify command line arguments, for example:
  ## Note `exec` is prepended to command
  ##
  command: "/run.sh"
  ## Additional commands to run prior to starting Redis(TM)
  ##
  preExecCmds: ""
  ## Additional Redis(TM) configuration for the master nodes
  ## ref: https://redis.io/topics/config
  ##
  configmap:
  ## Deployment pod host aliases
  ## https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/
  ##
  hostAliases: []
  ## Redis(TM) additional command line flags
  ##
  ## Can be used to specify command line flags, for example:
  ## extraFlags:
  ##  - "--maxmemory-policy volatile-ttl"
  ##  - "--repl-backlog-size 1024mb"
  ##
  extraFlags: []
  ## Comma-separated list of Redis(TM) commands to disable
  ##
  ## Can be used to disable Redis(TM) commands for security reasons.
  ## Commands will be completely disabled by renaming each to an empty string.
  ## ref: https://redis.io/topics/security#disabling-of-specific-commands
  ##
  disableCommands:
    - FLUSHDB
    - FLUSHALL

  ## Redis(TM) Master additional pod labels and annotations
  ## ref: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
  ##
  podLabels: {}
  podAnnotations: {}

  ## Redis(TM) Master resource requests and limits
  ## ref: http://kubernetes.io/docs/user-guide/compute-resources/
  # resources:
  #   requests:
  #     memory: 256Mi
  #     cpu: 100m
  ## Use an alternate scheduler, e.g. "stork".
  ## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
  ##
  # schedulerName:

  # Enable shared process namespace in a pod.
  # If set to false (default), each container will run in separate namespace, redis will have PID=1.
  # If set to true, the /pause will run as init process and will reap any zombie PIDs,
  # for example, generated by a custom exec probe running longer than a probe timeoutSeconds.
  # Enable this only if customLivenessProbe or customReadinessProbe is used and zombie PIDs are accumulating.
  # Ref: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/
  shareProcessNamespace: false
  ## Configure extra options for Redis(TM) Master liveness and readiness probes
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#configure-probes)
  ##
  livenessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 5
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 5

  ## Configure custom probes for images other images like
  ## rhscl/redis-32-rhel7 rhscl/redis-5-rhel7
  ## Only used if readinessProbe.enabled: false / livenessProbe.enabled: false
  ##
  # customLivenessProbe:
  #  tcpSocket:
  #    port: 6379
  #  initialDelaySeconds: 10
  #  periodSeconds: 5
  # customReadinessProbe:
  #  initialDelaySeconds: 30
  #  periodSeconds: 10
  #  timeoutSeconds: 5
  #  exec:
  #    command:
  #    - "container-entrypoint"
  #    - "bash"
  #    - "-c"
  #    - "redis-cli set liveness-probe \"`date`\" | grep OK"
  customLivenessProbe: {}
  customReadinessProbe: {}

  ## Redis(TM) Master Node selectors and tolerations for pod assignment
  ## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
  ## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-tolerations-beta-feature
  ##
  # nodeSelector: {"beta.kubernetes.io/arch": "amd64"}
  # tolerations: []
  ## Redis(TM) Master pod/node affinity/anti-affinity
  ##
  affinity: {}

  ## Redis(TM) Master Service properties
  ##
  service:
    ##  Redis(TM) Master Service type
    ##
    type: ClusterIP
    port: 6379

    ## External traffic policy (when service type is LoadBalancer)
    ## ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    ##
    externalTrafficPolicy: Cluster

    ## Specify the nodePort value for the LoadBalancer and NodePort service types.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
    ##
    # nodePort:

    ## Provide any additional annotations which may be required. This can be used to
    ## set the LoadBalancer service type to internal only.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer
    ##
    annotations: {}
    labels: {}
    loadBalancerIP:
    # loadBalancerSourceRanges: ["10.0.0.0/8"]

  ## Enable persistence using Persistent Volume Claims
  ## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  persistence:
    enabled: true
    ## The path the volume will be mounted at, useful when using different
    ## Redis(TM) images.
    ##
    path: /data
    ## The subdirectory of the volume to mount to, useful in dev environments
    ## and one PV for multiple services.
    ##
    subPath: ""
    ## redis data Persistent Volume Storage Class
    ## If defined, storageClassName: <storageClass>
    ## If set to "-", storageClassName: "", which disables dynamic provisioning
    ## If undefined (the default) or set to null, no storageClassName spec is
    ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
    ##   GKE, AWS & OpenStack)
    ##
    # storageClass: "-"
    accessModes:
      - ReadWriteOnce
    size: 8Gi
    ## Persistent Volume selectors
    ## https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector
    ##
    matchLabels: {}
    matchExpressions: {}
    volumes:
    #  - name: volume_name
    #    emptyDir: {}

  ## Update strategy, can be set to RollingUpdate or onDelete by default.
  ## https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets
  ##
  statefulset:
    labels: {}
    annotations: {}
    updateStrategy: RollingUpdate
    ## Partition update strategy
    ## https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#partitions
    # rollingUpdatePartition:
    volumeClaimTemplates:
      labels: {}
      annotations: {}

  ## Redis(TM) Master pod priorityClassName
  ##
  priorityClassName: null

  ## An array to add extra env vars
  ## For example:
  ## extraEnvVars:
  ##  - name: name
  ##    value: value
  ##  - name: other_name
  ##    valueFrom:
  ##      fieldRef:
  ##        fieldPath: fieldPath
  ##
  extraEnvVars: []

  ## ConfigMap with extra env vars:
  ##
  extraEnvVarsCM: []

  ## Secret with extra env vars:
  ##
  extraEnvVarsSecret: []

##
## Redis(TM) Slave properties
## Note: service.type is a mandatory parameter
## The rest of the parameters are either optional or, if undefined, will inherit those declared in Redis(TM) Master
##
slave:
  ## Slave Service properties
  ##
  service:
    ## Redis(TM) Slave Service type
    ##
    type: ClusterIP
    ## Redis(TM) port
    ##
    port: 6379

    ## External traffic policy (when service type is LoadBalancer)
    ## ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    ##
    externalTrafficPolicy: Cluster

    ## Specify the nodePort value for the LoadBalancer and NodePort service types.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
    ##
    # nodePort:

    ## Provide any additional annotations which may be required. This can be used to
    ## set the LoadBalancer service type to internal only.
    ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer
    ##
    annotations: {}
    labels: {}
    loadBalancerIP:
    # loadBalancerSourceRanges: ["10.0.0.0/8"]

  ## Redis(TM) slave port
  ##
  port: 6379
  ## Deployment pod host aliases
  ## https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/
  ##
  hostAliases: []
  ## Can be used to specify command line arguments, for example:
  ## Note `exec` is prepended to command
  ##
  command: "/run.sh"
  ## Additional commands to run prior to starting Redis(TM)
  ##
  preExecCmds: ""
  ## Additional Redis(TM) configuration for the slave nodes
  ## ref: https://redis.io/topics/config
  ##
  configmap:
  ## Redis(TM) extra flags
  ##
  extraFlags: []
  ## List of Redis(TM) commands to disable
  ##
  disableCommands:
    - FLUSHDB
    - FLUSHALL

  ## Redis(TM) Slave pod/node affinity/anti-affinity
  ##
  affinity: {}

  ## Kubernetes Spread Constraints for pod assignment
  ## ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
  ##
  # - maxSkew: 1
  #   topologyKey: node
  #   whenUnsatisfiable: DoNotSchedule
  spreadConstraints: {}

  # Enable shared process namespace in a pod.
  # If set to false (default), each container will run in separate namespace, redis will have PID=1.
  # If set to true, the /pause will run as init process and will reap any zombie PIDs,
  # for example, generated by a custom exec probe running longer than a probe timeoutSeconds.
  # Enable this only if customLivenessProbe or customReadinessProbe is used and zombie PIDs are accumulating.
  # Ref: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/
  shareProcessNamespace: false
  ## Configure extra options for Redis(TM) Slave liveness and readiness probes
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#configure-probes)
  ##
  livenessProbe:
    enabled: true
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 5
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 10
    successThreshold: 1
    failureThreshold: 5

  ## Configure custom probes for images other images like
  ## rhscl/redis-32-rhel7 rhscl/redis-5-rhel7
  ## Only used if readinessProbe.enabled: false / livenessProbe.enabled: false
  ##
  # customLivenessProbe:
  #  tcpSocket:
  #    port: 6379
  #  initialDelaySeconds: 10
  #  periodSeconds: 5
  # customReadinessProbe:
  #  initialDelaySeconds: 30
  #  periodSeconds: 10
  #  timeoutSeconds: 5
  #  exec:
  #    command:
  #    - "container-entrypoint"
  #    - "bash"
  #    - "-c"
  #    - "redis-cli set liveness-probe \"`date`\" | grep OK"
  customLivenessProbe: {}
  customReadinessProbe: {}

  ## Redis(TM) slave Resource
  # resources:
  #   requests:
  #     memory: 256Mi
  #     cpu: 100m

  ## Redis(TM) slave selectors and tolerations for pod assignment
  # nodeSelector: {"beta.kubernetes.io/arch": "amd64"}
  # tolerations: []

  ## Use an alternate scheduler, e.g. "stork".
  ## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
  ##
  # schedulerName:

  ## Redis(TM) slave pod Annotation and Labels
  ##
  podLabels: {}
  podAnnotations: {}

  ## Redis(TM) slave pod priorityClassName
  priorityClassName: null

  ## Enable persistence using Persistent Volume Claims
  ## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  persistence:
    enabled: true
    ## The path the volume will be mounted at, useful when using different
    ## Redis(TM) images.
    ##
    path: /data
    ## The subdirectory of the volume to mount to, useful in dev environments
    ## and one PV for multiple services.
    ##
    subPath: ""
    ## redis data Persistent Volume Storage Class
    ## If defined, storageClassName: <storageClass>
    ## If set to "-", storageClassName: "", which disables dynamic provisioning
    ## If undefined (the default) or set to null, no storageClassName spec is
    ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
    ##   GKE, AWS & OpenStack)
    ##
    # storageClass: "-"
    accessModes:
      - ReadWriteOnce
    size: 8Gi
    ## Persistent Volume selectors
    ## https://kubernetes.io/docs/concepts/storage/persistent-volumes/#selector
    ##
    matchLabels: {}
    matchExpressions: {}

  ## Update strategy, can be set to RollingUpdate or onDelete by default.
  ## https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets
  ##
  statefulset:
    labels: {}
    annotations: {}
    updateStrategy: RollingUpdate
    ## Partition update strategy
    ## https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#partitions
    # rollingUpdatePartition:
    volumeClaimTemplates:
      labels: {}
      annotations: {}

  ## An array to add extra env vars
  ## For example:
  ## extraEnvVars:
  ##  - name: name
  ##    value: value
  ##  - name: other_name
  ##    valueFrom:
  ##      fieldRef:
  ##        fieldPath: fieldPath
  ##
  extraEnvVars: []

  ## ConfigMap with extra env vars:
  ##
  extraEnvVarsCM: []

  ## Secret with extra env vars:
  ##
  extraEnvVarsSecret: []

## Prometheus Exporter / Metrics
##
metrics:
  enabled: false

  image:
    registry: docker.io
    repository: bitnami/redis-exporter
    tag: 1.17.1-debian-10-r12
    pullPolicy: IfNotPresent
    ## Optionally specify an array of imagePullSecrets.
    ## Secrets must be manually created in the namespace.
    ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
    ##
    # pullSecrets:
    #   - myRegistryKeySecretName

  # A way to specify an alternative redis hostname, if you set a local endpoint in hostAliases for example
  # Useful for certificate CN/SAN matching
  redisTargetHost: "localhost"

  ## Metrics exporter resource requests and limits
  ## ref: http://kubernetes.io/docs/user-guide/compute-resources/
  ##
  # resources: {}

  ## Extra arguments for Metrics exporter, for example:
  ## extraArgs:
  ##   check-keys: myKey,myOtherKey
  # extraArgs: {}

  ## Metrics exporter pod Annotation and Labels
  ##
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
  # podLabels: {}

  # Enable this if you're using https://github.com/coreos/prometheus-operator
  serviceMonitor:
    enabled: false
    ## Specify a namespace if needed
    # namespace: monitoring
    # fallback to the prometheus default unless specified
    # interval: 10s
    ## Defaults to what's used if you follow CoreOS [Prometheus Install Instructions](https://github.com/bitnami/charts/tree/master/bitnami/prometheus-operator#tldr)
    ## [Prometheus Selector Label](https://github.com/bitnami/charts/tree/master/bitnami/prometheus-operator#prometheus-operator-1)
    ## [Kube Prometheus Selector Label](https://github.com/bitnami/charts/tree/master/bitnami/prometheus-operator#exporters)
    ##
    selector:
      prometheus: kube-prometheus

    ## RelabelConfigs to apply to samples before scraping
    ## ref: https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#relabelconfig
    ## Value is evalued as a template
    ##
    relabelings: []

    ## MetricRelabelConfigs to apply to samples before ingestion
    ## ref: https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#relabelconfig
    ## Value is evalued as a template
    ##
    metricRelabelings: []
    #  - sourceLabels:
    #      - "__name__"
    #    targetLabel: "__name__"
    #    action: replace
    #    regex: '(.*)'
    #    replacement: 'example_prefix_$1'

  ## Custom PrometheusRule to be defined
  ## The value is evaluated as a template, so, for example, the value can depend on .Release or .Chart
  ## ref: https://github.com/coreos/prometheus-operator#customresourcedefinitions
  ##
  prometheusRule:
    enabled: false
    additionalLabels: {}
    namespace: ""
    ## Redis(TM) prometheus rules
    ## These are just examples rules, please adapt them to your needs.
    ## Make sure to constraint the rules to the current redis service.
    # rules:
    #   - alert: RedisDown
    #     expr: redis_up{service="{{ template "redis.fullname" . }}-metrics"} == 0
    #     for: 2m
    #     labels:
    #       severity: error
    #     annotations:
    #       summary: Redis(TM) instance {{ "{{ $labels.instance }}" }} down
    #       description: Redis(TM) instance {{ "{{ $labels.instance }}" }} is down
    #    - alert: RedisMemoryHigh
    #      expr: >
    #        redis_memory_used_bytes{service="{{ template "redis.fullname" . }}-metrics"} * 100
    #        /
    #        redis_memory_max_bytes{service="{{ template "redis.fullname" . }}-metrics"}
    #        > 90
    #      for: 2m
    #      labels:
    #        severity: error
    #      annotations:
    #        summary: Redis(TM) instance {{ "{{ $labels.instance }}" }} is using too much memory
    #        description: |
    #          Redis(TM) instance {{ "{{ $labels.instance }}" }} is using {{ "{{ $value }}" }}% of its available memory.
    #    - alert: RedisKeyEviction
    #      expr: |
    #        increase(redis_evicted_keys_total{service="{{ template "redis.fullname" . }}-metrics"}[5m]) > 0
    #      for: 1s
    #      labels:
    #        severity: error
    #      annotations:
    #        summary: Redis(TM) instance {{ "{{ $labels.instance }}" }} has evicted keys
    #        description: |
    #          Redis(TM) instance {{ "{{ $labels.instance }}" }} has evicted {{ "{{ $value }}" }} keys in the last 5 minutes.
    rules: []

  ## Metrics exporter pod priorityClassName
  priorityClassName: null
  service:
    type: ClusterIP

    ## External traffic policy (when service type is LoadBalancer)
    ## ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    ##
    externalTrafficPolicy: Cluster

    ## Use serviceLoadBalancerIP to request a specific static IP,
    ## otherwise leave blank
    # loadBalancerIP:
    annotations: {}
    labels: {}

##
## Init containers parameters:
## volumePermissions: Change the owner of the persist volume mountpoint to RunAsUser:fsGroup
##
volumePermissions:
  enabled: false
  image:
    registry: docker.io
    repository: bitnami/bitnami-shell
    tag: "10"
    pullPolicy: Always
    ## Optionally specify an array of imagePullSecrets.
    ## Secrets must be manually created in the namespace.
    ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
    ##
    # pullSecrets:
    #   - myRegistryKeySecretName
  resources: {}
  # resources:
  #   requests:
  #     memory: 128Mi
  #     cpu: 100m

  ## Init container Security Context
  ## Note: the chown of the data folder is done to containerSecurityContext.runAsUser
  ## and not the below volumePermissions.securityContext.runAsUser
  ## When runAsUser is set to special value "auto", init container will try to chwon the
  ## data folder to autodetermined user&group, using commands: `id -u`:`id -G | cut -d" " -f2`
  ## "auto" is especially useful for OpenShift which has scc with dynamic userids (and 0 is not allowed).
  ## You may want to use this volumePermissions.securityContext.runAsUser="auto" in combination with
  ## podSecurityContext.enabled=false,containerSecurityContext.enabled=false
  ##
  securityContext:
    runAsUser: 0

## Redis(TM) config file
## ref: https://redis.io/topics/config
##
configmap: |-
  # Enable AOF https://redis.io/topics/persistence#append-only-file
  appendonly yes
  # Disable RDB persistence, AOF persistence already enabled.
  save ""

## Sysctl InitContainer
## used to perform sysctl operation to modify Kernel settings (needed sometimes to avoid warnings)
##
sysctlImage:
  enabled: false
  command: []
  registry: docker.io
  repository: bitnami/bitnami-shell
  tag: "10"
  pullPolicy: Always
  ## Optionally specify an array of imagePullSecrets.
  ## Secrets must be manually created in the namespace.
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  ##
  # pullSecrets:
  #   - myRegistryKeySecretName
  mountHostSys: false
  resources: {}
  # resources:
  #   requests:
  #     memory: 128Mi
  #     cpu: 100m

## PodSecurityPolicy configuration
## ref: https://kubernetes.io/docs/concepts/policy/pod-security-policy/
##
podSecurityPolicy:
  ## Specifies whether a PodSecurityPolicy should be created
  ##
  create: false

## Define a disruption budget
## ref: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
##
podDisruptionBudget:
  enabled: false
  minAvailable: 1
  # maxUnavailable: 1

```