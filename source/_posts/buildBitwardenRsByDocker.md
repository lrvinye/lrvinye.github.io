---
title: 通过Docker部署 Bitwarden_rs
date: 2020-12-03 11:58:36
tags: [Docker, 运维, 安装, 配置, 密码]
category: 运维
---

# 环境

- Docker Ce 、 Docker Compose 使用 centos 7 yum 安装
- Linux Centos 7 1C 1G
- bitwardenrs 镜像

# 具体步骤

## 拉取所需镜像

- bitwardenrs 镜像是由[第三方](https://github.com/dani-garcia/bitwarden_rs "第三方")使用 Rust 实现的服务端代码，使得启动所需配置降低，并实现 docker 自动部署

```
docker pull bitwardenrs/server
```

## 部署容器

- 首先我们使用`openssl`生成一个管理员密钥，用于开启后台管理功能

```
openssl rand -base64 48
```

- 部署 bitwarden

```
# 运行 bitwarden_rs 容器
docker run -d --name bitwarden \
    -e SIGNUPS_ALLOWED=true \ #是否允许注册
    -e INVITATIONS_ALLOWED=true \ #是否允许邀请注册
    -e ADMIN_TOKEN=[REPLACE] \ #管理员密钥，可使用上一步生成的长密钥
    -e DOMAIN=[REPLACE]  \ #访问域名
    -p 8780:80  \ #访问端口绑定
    -p 8782:3012  \ #websocker端口绑定，如不开启ws功能可不绑定
    -v /opt/bitwarden/bw-data/:/data/ \ #挂载数据目录
    bitwardenrs/server:latest

```

更多参数可参考 [官方 Wiki](https://github.com/dani-garcia/bitwarden_rs/wiki "官方 Wiki")

# 需要注意的地方

- 使用 docker 部署的 nginx 进行反代访问时，使用宿主服务器的内网 ip + bw 绑定的端口 的形式进行转发，可实现跨容器访问
- bitwarden 需要使用 **https** 访问，否则无法正常使用
- bitwarden 备份或更新、重新部署，只需要保证数据目录相同即可
