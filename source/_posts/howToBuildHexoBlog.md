---
title: 哪个男孩不像拥有一款属于自己的博客呢？
date: 2020-05-29 23:13:35
updated: 2020-05-30 22:22:00
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



### 目录结构

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



------



### 常用的Hexo命令 (我会经常用到的)

```
npm update hexo -g 	#全局升级 HEXO
hexo init 			#初始化博客

hexo n post "name" 		  #新建文件名为 name 的文章
hexo n [temp] "name"		#以 temp 为模板 新建文件名为 name 的文章

hexo g == hexo generate 	#生成静态文件

hexo s == hexo server 		#启动本地热更新服务器
	-s #静态模式
	-p 5000 #更改端口为5000
	-i 192.168.1.1 #自定义 IP
	
hexo d == hexo deploy 		#根据配置文件中的部署规则进行构建部署
	-g #直接进行生成与部署 相当于 hexo g & hexo d
	
hexo clean 					#清除生成的静态文件
```



------



### 部署配置

这里我只进行 GitHub 部署的演示，也是我现在用的；

还记得在上面准备好的GitHub仓库吗

在hexo 的配置文件 `_config.yml` 中如下配置

```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/[你的用户名]/[你的用户名].github.io.git
  branch: master 		#一定要是master分支，因为GitHub page 只会识别你的master 分支
```



保存后，还要安装一个用于部署的插件

```
npm install hexo-deployer-git --save
```



再输入上面提到的部署命令

```
hexo clean 
hexo g 
hexo d
```



博客就成功上线了，只要访问 `[你的用户名].github.io` 就能通过GitHub page 访问了！👌



### 配置自定义域名

 自己的博客，自然得有自己的域名，不然怎么够(zhuang)酷(bi)😎



1. 设置 访问域名的 CNAME 解析记录 值为 `[你的用户名].github.io`
2. 在工程中的 source 路径下，新建名为 `CNAME ` 的文件，输入 你的访问域名，保存

它在这里：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   ├── _posts
|   └── CNAME 👈
└── themes
```

再 重新部署即可



## 换个主题



官方传送门 👉 `https://hexo.io/themes/` 

找个自己喜欢的主题，下下来

```
git clone https://github.com/xxx themes/xxx
```

**记住要放在 `themes` 路径下！**



接下来在配置文件 `_config.yml` 中配置

```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: xxx
theme_config: 			#主题的配置文件。在这里放置的配置会覆盖主题目录下的 _config.yml 中的配置
```

也就是设置使用 xxx 作为博客的主题

主题的路径下， 一般都有自己的 `_config.yml` 是主题的配置文件，需要根据不同的主题自行钻研摸索**个性化**



## 添加页面

当然，hexo 中不止可以添加文章，你也可以新建一个页面，

```
hexo new page "页面名"
```

基本上和文章差不多，这个一般是一些主题会用到，比如新建一个标签云的页面，专门展示所有的文章标签

可以通过 `域名/页面名` 的路径访问到，所以和文章略有不同



 

------



# 配置文件示例



```yml
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: lrvinye's blogs
subtitle: 'Cherryez'
description: 'this is a personal website'
keywords: 'lrvinye'
author: lrvinye
language: zh-CN
timezone: 'Asia/Shanghai'

# URL
## 如果您的网站存放在子目录中，例如 http://yoursite.com/blog，则请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/。
url: https://lrvinye.cn
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: true # 是否在永久链接中保留尾部的 index.html，设置为 false 时去除
  trailing_html: true # 是否在永久链接中保留尾部的 .html, 设置为 false 时去除 (对尾部的 index.html无效)

# Directory
source_dir: source #资源文件夹，这个文件夹用来存放内容。
public_dir: public #公共文件夹，这个文件夹用于存放生成的站点文件。
tag_dir: tags #标签文件夹
archive_dir: archives #归档文件夹
category_dir: categories #分类文件夹
code_dir: downloads/code #Include code 文件夹，source_dir 下的子目录
i18n_dir: :lang #国际化（i18n）文件夹
skip_render:  #跳过指定文件的渲染。匹配到的文件将会被不做改动地复制到 public 目录中。您可使用 glob 表达式来匹配路径。

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post #预设布局
auto_spacing: false #在中文和英文之间加入空格
titlecase: false # 把标题转换为 title case
external_link:
  enable: true # 在新标签中打开链接
  field: post # 对整个网站（site）生效或仅对文章（post）生效
  exclude: '' #需要排除的域名。主域名和子域名如 www 需分别配置
filename_case: 0 #把文件名称转换为 (1) 小写或 (2) 大写
render_drafts: false #显示草稿
post_asset_folder: false #启动 Asset 文件夹
relative_link: false #把链接改为与根目录的相对位址
future: true  #显示未来的文章
highlight:
  enable: true  #开启代码块高亮
  line_number: true  #显示行数
  auto_detect: false  #如果未指定语言，则启用自动检测
  tab_replace: ''  #用 n 个空格替换 tabs；如果值为空，则不会替换 tabs
  wrap: true 
  hljs: false 

# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
  path: ''
  per_page: 10 
  order_by: -date

# Category & Tag
default_category: uncategorized #默认分类
category_map: #分类别名
tag_map: #标签别名

# Metadata elements
## https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta
meta_generator: true #Meta generator 标签。 值为 false 时 Hexo 不会在头部插入该标签

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss
## 启用以后，如果 Front Matter 中没有指定 updated， post.updated 将会使用 date 的值而不是文件的创建时间。在 Git 工作流中这个选项会很有用
use_date_for_updated: true

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10 #每页显示的文章量 (0 = 关闭分页功能)
pagination_dir: page #分页目录

# Include / Exclude file(s)
## include:/exclude: options only apply to the 'source/' folder
#Hexo 默认会忽略隐藏文件和文件夹（包括名称以下划线和 . 开头的文件和文件夹，Hexo 的 _posts 和 _data 等目录除外）。通过设置此字段将使 Hexo 处理他们并将它们复制到 source 目录下。
include: 
#Hexo 会忽略这些文件和目录
exclude: 
ignore:

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: Chic
theme_config: #主题的配置文件。在这里放置的配置会覆盖主题目录下的 _config.yml 中的配置

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/lrvinye/lrvinye.github.io.git
  branch: master
```

