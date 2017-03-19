---
title: 在 Ubuntu 14.04 服务器上部署 Hexo 博客
tags:
  - Ubuntu
  - Hexo
categories:
  - Hexo
comments: true
date: 2017-03-19 15:31:00
---
> 开源的博客平台多如牛毛，而且不乏优秀之作，如Hexo、Octopress、Jekyll、Wordpress。
本系列文章将分享如何利用各种博客引擎在云端搭建属于自己的个人博客。
今天介绍如何在Ubuntu14.04上部署Hexo博客。

## 准备工作

1. 一台安装了Ubuntu 14.04的vps（推荐vultr,有日本机房，优化后速度可稳定在150ms,速度还不错）。
2. 耐性&google.

此外，还要在云服务器上安装Git和Nginx两个必备的软件包。Git用于版本管理和部署，Nginx用于静态博客托管。

```
sudo apt-get update
sudo apt-get install git nginx -y

```

##  1.本地Hexo安装及初始化

我们使用 Node.js 的包管理器npm安装hexo-cli和hexo-server.

```
npm install hexo-cli hexo-server -g

```

初始化hexo目录

```
hexo init hexo
npm install hexo-deployer-git --save  #部署git必须

```

在国内环境下执行该命令，速度会有些慢。建议使用淘宝cnpm或者使用代理加速。
这时，我们就已经有了一个写作、管理博客的环境。

### 安装 NexT 主题

Hexo 安装主题的方式非常简单，只需要将主题文件拷贝至站点目录的 themes 目录下， 然后修改下配置文件即可。具体到 NexT 来说，安装步骤如下。

```
cd your-hexo-site
git clone https://github.com/iissnan/hexo-theme-next themes/next

```

关于 Next 的配置请参考[Next官网](http://theme-next.iissnan.com/getting-started.html)。


## 2.云端服务器配置

完成本地端的操作之后，暂时回到服务器的配置。在下面的操作之前，请确保已经购买了一台云服务器。

在这部分，要完成以下件事情：

1. 为本地的hexo配置一个部署静态文件的远程仓库。
2. 配置 Nginx 托管博客文件目录。
3. 配置远程仓库自动更新到博客文件目录的钩子。

### 2.1创建私有git仓库

在/var/repo/下，创建一个名为hexo_static的裸仓库（bare repo）。

如果没有 /var/repo 目录，需要先创建；然后修改目录的所有权和用户权限，之后 ubuntu 用户都具备 /var/repo 目录下所有新生成的目录和文件的权限。

```
sudo mkdir /var/repo/
sudo chown -R $USER:$USER /var/repo/
sudo chmod -R 755 /var/repo/

```

然后，执行如下命令：

```
cd /var/repo/
git init --bare hexo_static.git

```
### 2.2配置nginx托管文件目录

```
sudo mkdir -p /usr/share/nginx/hexo	
sudo chown -R $USER:$USER /usr/share/nginx/hexo
sudo chmod -R 755 /usr/share/nginx/hexo

```
然后，修改 Nginx 的 default 设置：

```
sudo vim /etc/nginx/sites-available/default

```
将其中的 root 指令指向 /usr/share/nginx/hexo 目录。

```
...

server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;

	root /usr/share/nginx/hexoo; # 需要修改的部分
	index index.html index.htm;
	server_name hujianlong.top;  # 改为你的域名

...

```
保存并退出文件。将配置中的 hujianlong.top 修改为你的域名。

最后，重启 Nginx 服务，使得改动生效。

```
sudo /etc/init.d/nginx restart

```

### 2.3创建 Git 钩子

接下来，在服务器上的裸仓库 hexo_static 创建一个钩子，在满足特定条件时将静态 HTML 文件传送到 Web 服务器的目录下。

在自动生成的 hooks 目录下创建一个新的钩子文件：

```
vim /var/repo/hexo_static.git/hooks/post-receive

```

在该文件中添加两行代码，指定 Git 的工作树（源代码）和 Git 目录（配置文件等）。

```
#!/bin/bash

git --work-tree=/usr/share/nginx/hexo --git-dir=/var/repo/hexo_static.git checkout -f

```

保存并退出文件，并让该文件变为可执行文件。

```
chmod +x /var/repo/hexo_static.git/hooks/post-receive

```

## 3.完成本地 Hexo 配置

在第三部分的操作中，我们将完成以下任务：

1.修改 Hexo 配置中的 URL 和默认文章版式；
2.新建博客草稿并发布
3.配置自动部署到服务器端的 hexo_static 裸仓库；

### 3.1修改 Hexo 部分默认配置

进入 hexo_blog 目录，_config.yml 为 Hexo 的主配置文件。我们首先修改博客的 URL 地址。

```
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'

url: http://server-ip # 没有绑定域名时填写服务器的实际 IP 地址。
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

```

接下来，修改 default_layout，该字段位于在 Writing 部分。将其从 post 修改为 draft ，表示每篇博文默认都是草稿，必须经过发布之后才能在博客站点上访问。

```
# Writing
new_post_name: :title.md # File name of new posts
default_layout: draft # 原来的值是 post
titlecase: false # Transform title into titlecase

```

通过 Git 部署，继续编辑 _config.yml 文件，找到 Deployment 部分，按照如下情况修改：

```
deploy:
    type: git
	repo: root@云服务器的IP地址:/var/repo/hexo_static
	branch: master

```
保存并退出文件。

### 3.2新建博客草稿并发布

主要命令总结为：

```
hexo new <博文title>
hexo publish <博文title>
hexo generate && hexo deploy

```

## 总结

本文较为完整地介绍了 Hexo 博客的安装及初始化，服务端如何配置通过 Git 部署等。
通过 Git 钩子，将 Hexo 生成的博客静态文件，快速地推送到 Web 服务的托管目录。
使用Next主题。
