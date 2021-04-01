---
title: 通过 Docker 或 K8S 部署 Kong 网关
tags: [API,Kong , 网关,K8S, Docker, 运维, 安装, 配置]
category: 运维
date: 2020-12-11 17:07:13
---

# 环境

- Docker or K8S
- 一台正常运行可访问的 Postgres 数据库

# 具体步骤

## postgres 准备

- 创建 kong 的使用用户

|          | value |
| -------- | ----- |
| username | kong  |
| password | kong  |

> 注意此处创建的用户需要勾选允许登录选项

- 创建名为 `kong` 的数据库，数据库的拥有者为刚刚创建的`kong`用户

## 迁移初始化数据

> 这里演示 K8S 的 yaml 写法

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kong-migrations
  namespace: test
spec:
  template:
    metadata:
      name: kong-migrations
    spec:
      containers:
        - name: kong-migrations
          image: kong:latest
          env:
            - name: KONG_DATABASE
              value: "postgres"
            - name: KONG_PG_HOST
              value: "pgsql"
            - name: KONG_PG_PASSWORD
              value: "kong"
            - name: KONG_PG_USER
              value: "kong"
          args:
            - /bin/sh
            - -c
            - kong migrations bootstrap
      restartPolicy: Never
```

> 只用执行一次，所以使用 Job

## 创建运行 kong

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kong
  template:
    metadata:
      labels:
        app: kong
    spec:
      containers:
        - name: kong
          image: kong:latest
          env:
            - name: KONG_DATABASE
              value: "postgres"
            - name: KONG_PG_HOST
              value: "pgsql"
            - name: KONG_PG_PASSWORD
              value: "kong"
            - name: KONG_PG_USER
              value: "kong"
            - name: KONG_PROXY_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_ADMIN_ACCESS_LOG
              value: "/dev/stdout"
            - name: KONG_PROXY_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_ADMIN_ERROR_LOG
              value: "/dev/stderr"
            - name: KONG_ADMIN_LISTEN
              value: "0.0.0.0:8001, 0.0.0.0:8444 ssl"
          ports:
            - containerPort: 8000
              name: web
            - containerPort: 8001
              name: admin
            - containerPort: 8443
              name: ssl
            - containerPort: 8444
              name: adminssl
          livenessProbe:
            exec:
              command:
                - kong
                - health
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
```

- KONG_ADMIN_LISTEN 用于指定管理员接口地址，分别为 http 与 https 地址

## 搭建 Konga 管理面板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: konga
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: konga
  template:
    metadata:
      labels:
        app: konga
    spec:
      containers:
        - name: konga
          image: pantsel/konga
          env:
            - name: DB_ADAPTER
              value: postgres
            - name: DB_HOST
              value: pgsql
            - name: DB_PORT
              value: "5432:5432"
            - name: DB_PASSWORD
              value: kong
            - name: DB_USER
              value: kong
            - name: DB_DATABASE
              value: konga
          ports:
            - containerPort: 1337
              name: web
```

> yaml 中指定的 konga 数据库不用提前创建，容器会自行创建

至此，konga 与 kong 都搭建完成，访问 konga 的 web 地址，便可注册登录连接 kongAdmin 进行管理
