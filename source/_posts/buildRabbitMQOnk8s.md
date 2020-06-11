---
title: 在K8S上部署RabbitMQ集群
date: 2020-06-11 17:43:37
category: 运维
tags: [k8s,运维,安装,配置,消息队列,RabbitMQ]
---



# 环境

```yml
#这里的K8S是部署的腾讯云 TKE 容器服务
K8S:
	Master: 1.16.3-tke.6
	Node: 1.16.3-tke.7
	运行时组件: containerd

	#RabbitMQ 3.8.2 on Erlang 22.2.8#
docker:
	image: rabbitmq:3.8.2-management
	
```

 

# 创建唯一erlang.cookie



```bash
# 创建erlang.cookie
echo $(openssl rand -base64 32) > erlang.cookie
# k8s 注入 secret
kubectl create secret generic erlang.cookie --from-file=erlang.cookie -n r
```





# YAML构建





## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
  namespace: ns
```

## Role

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq
  namespace: ns
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
```

## RoleBinding

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq
  namespace: ns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
  namespace: r
```

## Service

```yaml
#rabbit外部服务使用
---
kind: Service
apiVersion: v1
metadata:
  name: rabbitmq-service
  namespace: ns
spec:
  type: NodePort
  ports:
    - name: mangement
      protocol: TCP
      port: 15672
      nodePort: 31002
    - name: smp 
      protocol: TCP
      port: 5672
      nodePort: 31001
  selector:
    app: rabbitmq
```

```yaml
# rabbit内部节点互相通信使用
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  namespace: ns
  labels:
    app: rabbitmq
spec:
  clusterIP: None
  ports:
  - port: 5672
    name: amqp
  selector:
    app: rabbitmq
```



## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: ns
spec:
  serviceName: rabbitmq
  updateStrategy:
    type: RollingUpdate
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      name: rabbitmq
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      containers:
      - name: rabbitmq
        image: rabbitmq:3.8.2-management
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            memory: "800Mi"
            cpu: "0.4"
          limits:
            memory: "900Mi"
            cpu: "0.6"
        volumeMounts:
          - name: rabbitmq-data
            mountPath: /var/lib/rabbitmq
            subPath: mnesia
        ports:
        - containerPort: 5672
          name: amqp
        env:
          - name: RABBITMQ_DEFAULT_USER
            value: user
          - name: RABBITMQ_DEFAULT_PASS
            value: password
          - name: RABBITMQ_ERLANG_COOKIE
            value: erlang.cookie #使用前面生成的cookie
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: SERVICE_INTERNAL_NAME
            value: "rabbitmq"
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          - name: RABBITMQ_NODENAME
            value: "rabbit@$(POD_NAME).$(SERVICE_INTERNAL_NAME).$(NAMESPACE).svc.cluster.local"
      volumes:
        - name: rabbitmq-data
          nfs:
            path: /rabbitmq
            server: 172.16.0.15
```



# 加入集群

进入 `从节点 1 与 2` 的容器

```bash
#查看 erlang.cookie
cat $HOME/.erlang.cookie

#停止应用
rabbitmqctl --erlang-cookie erlang.cookie stop_app
#加入节点0
rabbitmqctl --erlang-cookie erlang.cookie join_cluster rabbit@rabbitmq-0
#启动应用
rabbitmqctl --erlang-cookie erlang.cookie start_app
```

再进入 `主节点0` 的容器

```bash
#查看集群状态
rabbitmqctl --erlang-cookie erlang.cookie cluster_status
```

成功则能在集群状态中看到以下内容

```bash
----Running Nodes
----rabbit@rabbitmq-0.rabbitmq.ns.svc.cluster.local
----rabbit@rabbitmq-1.rabbitmq.ns.svc.cluster.local
----rabbit@rabbitmq-2.rabbitmq.ns.svc.cluster.local
```



通过 `rabbitmq:15672` 登录管理web界面成功后

能看到：

![](https://pic.cherryez.com/lrvinye/2020/6/11/TSBrowser_uV63m3BeIp.png)

**大功告成！**