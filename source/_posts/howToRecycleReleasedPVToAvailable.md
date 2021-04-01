---
title: 如何将已释放的PV回收为可用状态
date: 2021-03-31 20:36:07
tags: [K8S, 配置, 已解决]
category: 运维
---

编辑 需要回收的 PV 的 YAML

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: redis-data-redis-node-0
    namespace: ibootcloud
    resourceVersion: "24344047835"
    uid: 4152f2b7-c473-4975-b60c-6eaa8085b3a9
  csi:
    driver: com.tencent.cloud.csi.cbs
    volumeHandle: disk-8f85b1kw
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cbs-db-sc-test
  volumeMode: Filesystem
status:
  phase: Released
```

手动删除`claimRef`项：

```
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: redis-data-redis-node-0
    namespace: ibootcloud
    resourceVersion: "24344047835"
    uid: 4152f2b7-c473-4975-b60c-6eaa8085b3a9
```

保存生效即可