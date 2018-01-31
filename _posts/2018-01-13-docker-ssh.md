---
layout: post
title:  "本地容器化管理(Using Docker)"
subtitle: "Local Service Management By Docker"
date:   2018-01-13 23:59:59 +0800
background: '/img/posts/06.jpg'
---

## 本地容器化管理——前言

与其说引入Docker，不如说是引入了一种思想。围绕一件尽量内聚的事情，将运行环境和代码“封装”在一个环境里，以服务的形式对外透出。熟悉面向对象思想和编程原则的朋友一定觉得非常熟悉，只是原则的范围扩大到了整个运行环境，甚至是操作系统。这不仅提高了可移植性，更是一种组织服务的方式。

从现实来看，我们总是对自己的本地环境(开发机器)容易忽视，疏于对它的治理。

设想一下，在我们是如何在本地写代码、做研发甚至是学技术的：了解自己的系统(Mac或是其他)；通过brew/yum/apt-get安装系统工具包、安装技术栈运行环境，还有数据库、IDE及其他编程辅助程序、版本控制软件、工程化套件。。。直到新的需求出现，前面的程序将周而复始。慢慢地，你将拥有一台很“强大”的电脑，用起来非常趁手，编写的程序都可以完美运行，简直无所不能。但有两个问题需要思考。

### 交付——换台机器，我写的代码还能跑吗？

举个例子，在本地的Mac系统进行开发，使用了NodeJS Latest版本，完成了一个webapp。然后我把代码部署到测试环境——一个装有NodeJS LTS版本的CentOS系统。这里至少已经出现了两个明显的风险点，1.不同系统的NodeJS安装；2.LTS环境无法实用Latest的某些新特性。一句话：我们交付的只是代码，而不是一个对代码+环境的完整描述。可以通过增加运行环境说明(readme)来一定程度解决这个问题，但这并不可靠和优雅。

### 混乱——多版本、多技术栈

举个例子，使用jekyll搭建个人博客，需要Ruby运行环境(2.5)。Mac自带Ruby运行环境，但版本较低。对系统版本进行升级不可取(系统程序依赖低版本)，只能使用RVM做多版本管理。当我们需要用最新的Python版本完成一个爬虫程序时，同样的事情发生在Python(系统2.7)和Python3。多个技术栈和多版本(或是版本管理工具)，让本地这个“应用”非常臃肿和混乱，一段时间以后很难记得哪些技术组合解决哪些开发问题。

综上，借由Docker，我们需要对自己的本地环境做一次全新的梳理和升级。Docker帮我们做好了准备工作。它的基本概念可以参考[官方文档](https://docs.docker.com/reference/)，写的非常的清晰明白。这里对几个比较重要的环节做一些分享和探讨。


## 容器连接

进入容器有很多种方式，第一类是docker本身提供的工具。

### docker run -ti
```
docker run -t -i ubuntu:latest /bin/bash
```
-t是开启一个tty伪终端(pseudo-TTY)，将容器的标准输出展示到这里；-i则表示打开标准输入(STDIN)。这种做法简单直接，缺点也很明显：只能在创建时进入。而且显式地指定/bin/bash会覆盖镜像的CMD指令。

### attach
```
docker attach containerId   # containerId：已经运行的容器id
```
attach进入容器会有几个问题：

1. 只能进入以/bin/bash为启动命令的容器
2. 多个窗口使用该命令进入同一容器的时候所有的窗口都会同步显示。如果有一个窗口阻塞了，那么其他的窗口再也无法进行其他的操作
3. 会出现莫名其妙的卡死情况(实际使用中发现的)

### exec
官方提供的另一种方式进入容器，且简单灵活:
```
docker exec -it containerId /bin/bash # containerId：已经运行的容器id
```
这条指令很好理解，exec允许在已经运行的容器中自定义参数(如-it)和执行命令(/bin/bash)，对于docker自带工具来说，exec通常是最好的选择。

### ssh

ssh(Secure Shell)：远程登录会话协议，重点两个字：远程，这是docker上述自带工具所不具备的。而通过远程登录来管理应用或服务，对很多研发人员来说，更是一种习惯。

在笔者的工作环境，无论应用是否Docker化，从运维的角度讲，都不建议(甚至是禁止)直接远程登录服务器进行日志查询、甚至是重启等操作，取而代之的是各产品化的解决方案：比如B/S形态更产品化的日志查询、应用生命周期管理等等。但程序员对“掌控感”的渴求与生俱来，运行环境上下文中有什么、发生了什么、结果是什么，这些问题一定要到“现场”才最痛快直接。所以在测试环境等一切允许的场景，都是“登”上去一顿操作。

为容器“植入”ssh能力，从理念来看，业界也存在争论。Docker的理念是一个容器只运行一个服务，如果需要远程连接，每个容器运行一个额外的SSH守护进程服务，有违这一理念。此外也意味着引入了被更多攻击的可能。

本文讨论的更倾向本地Docker化服务管理，所以理念和风险可以适当弱化，而通过多个ip+port的方式定位服务、管理服务，是更熟悉且易于切换到分布式环境的方式。

ssh分为口令登录和免密登录(秘钥登录)，[基本概念](http://blog.csdn.net/yimingsilence/article/details/52161412)参考这里。
而秘钥登录，又涉及到公钥和私钥，且公钥私钥在普通加密和认证这两种场景的使用方式是有区别的，强烈建议阅读[这里](http://blog.51cto.com/bingdian/313319)

下文中采用的是口令登录，免密登录笔者也进行了尝试，但遇到了**Permission denied**的问题，一直无法解决(Linux权限苦手)。
ssh可以作为核心功能集成到基础镜像，在本人的[docker-libs](https://github.com/nanda221/docker-libs/blob/master/base/Dockerfile)中已经有一份样例，里面保留了很多采坑和测试的注释，可以辅助理解在镜像构建时的一些考量。下面是几行核心指令：
```
From ...
...
RUN apt-get update
# 安装sshd
RUN apt-get install -y openssh-server
# 创建运行目录
RUN mkdir /var/run/sshd
# 修改root密码
RUN echo 'root:pswd1234' | chpasswd
# 允许root登录
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
# 官方解释：SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
# 创建启动脚本
RUN echo '#!/bin/bash\n/usr/sbin/sshd -D' >> /root/bootstrap.sh && chmod 755 /root/bootstrap.sh
# 添加启动进入点
ENTRYPOINT ["/root/bootstrap.sh"]
# 暴露端口，ssh约定监听端口22
EXPOSE 22
...
```
使用Dockerfile构建镜像，假设tag为sshd:0.1，然后就可以运行：
```
# 方式一：随机端口映射
docker run -d -P sshd:0.1
# 方式二：指定端口映射
docker run -d -p 10122:22 sshd:0.1
```
方式一执行后可以通过**docker port containerId查看映射的随机端口号**。
然后就可以通过**ssh root@0.0.0.0 -p portnumber**进入容器，注意，上面指定的密码为**pswd1234**

## 编程语言运行环境

根据服务的需要，要准备编程语言需要的运行环境。本人的[docker-libs](https://github.com/nanda221/docker-libs)中已经准备了一些较新且常用的语言环境镜像作为参考。

## 生命周期管理

待施工：重点阐述如何通过更产品化、界面化的方式来管理容器的启停生命周期。



















