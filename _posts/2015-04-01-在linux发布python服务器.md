---
date: 2015-04-01 15:27:31
layout: post
title: 在linux（ubantu）上部署django工程
category: python
tags: python
keywords: python wsgi linux ubantu django
description:
---

###门槛

文章是在默认你已经掌握以下基础知识编写，如果在阅读过程中发现对以下基础知识不太了解，可以先查看相关资料

1.基本的linux命令行，如基本的`cd`、`cat`、`vi`、`sudo`操作

2.懂得使用git进行代码管理，如基本的`git pull`、`git push`、`git clone`操作
>服务器会通过git来同步代码

3.会使用ssh连接服务器，如`ssh root@[hostname]`
>服务器都连不上的话，还谈什么部署...

4.有一个能运行django框架,能使用`python manage.py runserver`运行
>那怕是输出一个hello world也够了，因为目的是部署，不是实现服务器的功能

###前言
先介绍用到的几个术语

	pip				python包管理工具
	django 			多功能python框架
	git				代码管理神器
	gunicorn		高性能python服务器
    gevent			python并发框架
    supervisor		管理python服务器的开启、关闭、查看状态
	nginx			高性能的 HTTP 和 反向代理
    fabric			自动化部署框架

>这里只是简单说明一下用到的术语，更多深入详细的了解读者可以自行google

##快速上手
我的服务器环境是Ubantu 64位，Django1.6.6，python2.7，给各位参照下，如果版本不同的话或许会出现其他异常情况
####1.前期准备
>如果你已经准备了一个可执行的django， 用git管理代码了，可以跳过这一步了

准备一个可以运行的django项目

	python manage.py runserver

用git管理代码，初始化git

	git init

添加远程仓库

    git remote add origin [remote git url]

代码加入git管理`git add .`

提交更新`git commit -am '[change log]'`

推上服务器`git push`

好，在开发机器的工作完成了，下面进入服务器部分。

####1.配置服务器环境
首先用ssh连接服务器

	ssh root@[hostname]

安装需要用到环境

```
`apt-get install pip`

`apt-get install nginx`

`apt-get install git`

`pip install gunicorn`

`pip install supervisor`

`pip install django==1.6.6`

`pip install gunicon`
```

安装环境后把代码同步下来

	git clone [git url]

或者

	git pull

在项目路径下，尝试运行django

	gunicorn [project name].wsgi

如果运行成功，说明django项目和gunicron都没有问题


>如果需要root权限的在命令前面加上sudo，如 `sudo apt-get install pip`

>python库用pip安装，其他库用apt-get安装

####2.写配置文件
编写nginx配置文件`nginx.conf`，保存到`/etc/nginx/`

```
user www-data;
worker_processes 2;
pid /var/run/nginx.pid;

events {
	worker_connections 768;
}

http {
    server {
        listen       80;
        #server_name  localhost;

        #access_log  logs/host.access.log  main;

        location ~ ^\/static\/.*$ {
            root /home/ScraperService;
        }

        location / {
            proxy_pass http://127.0.0.1:9000;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

检测一下配置有没有问题

	/etc/init.d/nginx configtest

>nginx配置详细请移步[nginx官网文档](http://nginx.org/en/docs/beginners_guide.html)

编写supervisor配置文件`[project name].conf`，保存到`/etc/supervisor/conf.d`

```
[program:[project name]]
command     = gunicorn_django [project path]/[project name]/settings.py -b 0.0.0.0:9000 -w 2 -k gevent
directory   = [project path]
user        = www-data
startsecs   = 3

redirect_stderr         = true
stdout_logfile_maxbytes = 50MB
stdout_logfile_backups  = 10
stdout_logfile          = [log path]
```

其中[project name]、[project path]、[log path]要改成对应的路径

>gunicorn参数 -w 线程数 -b 127.0.0.1:8000 监听的端口 -k gevent 运行方式，详细配置请移步[gunicorn官网文档](http://docs.gunicorn.org/en/latest/run.html)

>supervisor详细配置请移步[supervisor官网](http://supervisord.org/)

####3.启动服务

更新nginx配置

	nginx reload

重启nginx

	nginx restart

[](blogimage/013.png)

更新supervisor配置

	supervisorctl update

停止已经开启的项目

	supervisorctl stop [project name]

重新启动项目

	supervisorctl start [project name]

没有报错就启动成功了，报错的可以去[log path]查看log

>supervisor默认log在/var/log/supervisor，nginx默认log在/var/log/nginx

启动以后可以随时查看项目状态

	supervisorctl status


流程
用supervisor负责管理监控gunicorn
用gunicorn启动web service
启动nginx做反向代理和静态服务


####4.自动部署
通过以上步骤已经可以部署成功，但是每次修改代码后，都重新登录服务更新代码、重启supervisor服务、重启nginx服务，非常麻烦。就没有更加geek的方法吗？当然有！
在开发机器安装fabric

	pip install fabric

编写fabfile.py放在项目里

```
# -*- coding:utf-8 -*-

from fabric.api import *

# 服务器登录用户名:
env.user = 'root'
# 服务器地址，可以有多个，依次部署:
env.hosts = ['hostname', ]

env.passwords = {
    'root@[hostname]': '[password]',
}

#不用密码也可以用ssh登录
# env.key_filename = '~/.ssh/id_rsa'

_REMOTE_BASE_DIR = '[project path]'

def deploy():
    # 切换到项目目录
    with cd(_REMOTE_BASE_DIR):
        run('git pull origin dev')

    # 重启supervisor服务和nginx服务器:
    with settings(warn_only=True):
        run('supervisorctl stop ScraperService')
        run('supervisorctl start ScraperService')
        run('/etc/init.d/nginx reload')

```

以后更改代码后，不需要再登录服务器，直接运行fab命令，就可以自动部署了

	fab deploy

![](/blog_image/013.png)

>fabric会自动帮你登录服务器，并执行预先设定的命令

什么？每次更改后都要执行`fab deploy`，还是觉得不够geek?还有更懒（shuǎng）的方法，git hook

###5.git hook自动部署

git hook
web hook
待续...

>还可以用linux软链接的方式来管理部署路径，主要作用是方便切换版本，这里只是给个引子，主页菌也还没有详细研究，有兴趣的读者们可以自行了解。








