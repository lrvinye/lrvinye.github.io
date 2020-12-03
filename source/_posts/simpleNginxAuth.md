---
title: 给Nginx设置简单访问验证
date: 2020-06-23 16:43:20
category: 运维
tags: [Nginx,运维,配置,安全]
---





## 在 Nginx 的配置文件中添加auth_basic


```conf
# 在反向代理中添加
server {
      listen 80;
      server_name www.test.com;


      location / {
          auth_basic "User Authentication";		#认证信息
          auth_basic_user_file /usr/local/nginx/conf/htpasswd.db;		#密码文件
          
          
          proxy_pass http://xxx;
      }
```



## 使用htpasswd命令生成密码文件

> htpasswd是apache自带的小工具，如果找不到该命令，请使用 `yum install httpd-tools` 安装





```sh
# 生成 /usr/local/nginx/conf/passwd.db 密码库 并添加用户 admin
htpasswd -c /usr/local/nginx/conf/passwd.db admin

#输出
New password:    #输入你的密码
Re-type new password:  #重复你的密码
Adding password for user admin
```

重启 Nginx

再次访问将提示输入基本认证信息 [`用户名`，`密码`]





**大功告成**