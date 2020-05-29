---
title: 哪个男孩不像拥有一款属于自己的博客呢？
date: 2020-05-29 23:13:35
category: 运维
tags: [博客,网站,运维,安装,配置]
---

# 前言（随便写点）

其实早在之前，我已经使用wordPress搭建过CMS系统，工作室博客等等，

但是用来用去，都感觉太繁琐了，可视化的界面用起来不像想象中的友好

Hexo 最后还是成为了我的选择，短小精悍！这是一款基于Node.js的静态博客框架，👉 依赖少易于安装使用

可以非常方便的部署在GitHub等平台上

各种主题都不错，上手也比其它来的要快，而且作者是位台湾同胞，对中文的支持想必也是很香的👍

好了，话不多说，今天想来写一写如何来用Hexo快速搭建一个属于你自己的博客！

 

老规矩，  [官方中文文档](https://hexo.io/zh-cn/docs/)   👈戳这里

------

 

[TOC]

 

------





# 环境（你需要用到的）

- [x] GIt 最基本的不再赘述
- [x] 能使用Npm 的 Node.js
- [x] 能使用 Github Page 的github 账号
- [x] 会使用Markdown 的你

由于今天就打算写写Hexo有关的，上面提到的环境，下面就不再教如何使用了。

------





# 具体实施



### 准备Github Page

使用自己的github账号创建一个名为 `[你的账户名].github.io`  的==公开==仓库！不公开怎么访问呢？🙃

上面的仓库名是固定格式的写法，为的是能通过 `github page` 进行访问。



### NPM安装Hexo



这里展示 的是全局安装，适用于快速上手

```bash
npm install -g hexo-cli 
```

进阶操作应该是单项目下安装

```bash
npm install hexo
```



安装以后，可以使用以下两种方式执行 Hexo：

1. `npx hexo <command> ` 

2. 将 Hexo 所在的目录下的 `node_modules` 添加到环境变量之中即可直接使用 `hexo ` ,用法与全局安装一样：

   ```
   echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile
   ```



### 初始化Hexo



新建一个文件夹，用于放置博客，在该文件夹下执行：

```bash
hexo init blog
```



其实

这个时候以及ok了，你可以执行：

```bash
#生成文件
hexo g 
#启动本地开发服务器
hexo s
```

然后你就能在默认提示的 路径进行访问了 ，

这个时候，文件夹里的目录姐勾应该是这样的：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

`_config.yml` 是Hexo的配置文件，可以在里面自定义配置博客的相关信息

`scaffolds` 是模板文件夹，可以自定义用于一键生成的模板Markdown文件

`source` 里面保存的所有的页面和文章,除 `_posts` 文件夹之外，开头命名为 `_` (下划线)的文件 / 文件夹和隐藏的文件将会被忽略

`themes` 里面是下载的主题文件夹,Hexo 会根据主题来生成静态页面



### 一些常用的Hexo命令