---
title: 在K8S上部署ES集群
date: 2020-06-22 14:52:35
category: 运维
tags: [k8s, 运维, 安装, 配置, 搜索引擎, ES, ElasticSearch]
---

# 环境

```yml
#这里的K8S是部署的腾讯云 TKE 容器服务

K8S:
  Master: 1.16.3-tke.6
  Node: 1.16.3-tke.8
  运行时组件: containerd

  #ElasticSearch 7.3.2
docker:
  image: ElasticSearch:7.3.2
```

# 创建 ES 配置文件的 configmap：

```bash
# cat es-cm.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: es
  namespace: ns
data:
  elasticsearch.yml: |
    cluster.name: "${NAMESPACE}"
    node.name: "${POD_NAME}"
    network.host: 0.0.0.0
    discovery.seed_hosts: "es-in" #内部通信服务
    cluster.initial_master_nodes: "es-0,es-1,es-2" #三个pod
```

# YAML 构建

## Service

```yaml
# 集群内部节点通信使用
apiVersion: v1
kind: Service
metadata:
  name: es-in
  namespace: ns
  labels:
    k8s-app: es
spec:
  selector:
    k8s-app: es
  clusterIP: None
  ports:
    - name: in
      port: 9300
      protocol: TCP
```

```yaml
#外部调用服务使用
apiVersion: v1
kind: Service
metadata:
  name: es-out
  namespace: ns
  labels:
    k8s-app: es
spec:
  selector:
    k8s-app: es
  ports:
    - name: out
      port: 9200
      protocol: TCP
```

## StatefulSet

```yaml
# es-statefulset.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es
  namespace: ns
  labels:
    k8s-app: es
spec:
  replicas: 3 #3个节点
  serviceName: es
  selector:
    matchLabels:
      k8s-app: es
  template:
    metadata:
      labels:
        k8s-app: es
    spec:
      containers:
      - name: es
        image: elasticsearch:7.3.2
        command: #修改 vm.max_map_count 以达到最低使用标准
        - bash
        - -c
        - ulimit -l unlimited && sysctl -w vm.max_map_count=262144 && chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data && exec su elasticsearch docker-entrypoint.sh
        env:
          - name: NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: ES_HEAP_SIZE
          	value: 512m
          - name: ES_JAVA_OPTS
          	value: -Xms512m -Xmx512m
          - name: path.data #持久化储存 多节点的权限问题：es的数据目录默认只允许一个节点访问，但在k8s上采用了持久卷，所有节点的数据都存储在这个卷上，这会导致es的访问权限问题。可以通过更改es的配置max_local_storage_nodes来允许多个节点访问同一个数据目录，但es官方不推荐这样做。所以我们的方案是更改每个节点的数据存储目录来解决，指定es配置项path.data来实现
          	value: /usr/share/elasticsearch/data/$(POD_NAME)
        resources:
          limits:
            cpu: 256m
            memory: 1Gi
          requests:
            cpu: 10m
            memory: 128Mi
        readinessProbe: #可读检查
          failureThreshold: 3
          httpGet:
            path: /_cluster/health?local=true
            port: 9200
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        securityContext: #特权级容器，需要获取root权限，否则无法在上面修改 vm.max_map_count
          privileged: true
        volumeMounts:
          - name: es-config
            mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
            subPath: elasticsearch.yml
          - mountPath: /usr/share/elasticsearch/data
          	name: es-data
          	subPath: data
      volumes:
        - name: es-config
          configMap:
            name: es
        - name: es-data
        	nfs:
          		path: /elasticsearch
          		server: 172.16.0.15
```

通过 `es-out:9200` 访问可得

```json
{
  "name": "es-1",
  "cluster_name": "ns",
  "cluster_uuid": "oWW0e-FbTTaskiWU2_BFxg",
  "version": {
    "number": "7.3.2",
    "build_flavor": "default",
    "build_type": "docker",
    "build_hash": "1c1faf1",
    "build_date": "2019-09-06T14:40:30.409026Z",
    "build_snapshot": false,
    "lucene_version": "8.1.0",
    "minimum_wire_compatibility_version": "6.8.0",
    "minimum_index_compatibility_version": "6.0.0-beta1"
  },
  "tagline": "You Know, for Search"
}
```

**\*\*大功告成！\*\***
