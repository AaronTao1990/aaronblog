---
title: Docker初探系列（一）
date: 2016-05-13 10:46:49
tag: Docker
category : Docker
---

## 简介
### Docker能用来干什么？
* 更快的发布应用
* 让部属和负载均衡更加轻松
* 更加弹性，能够支持更多负载


### Docker主要有哪些组件
* Docker Engine:开源的容器平台
* [Docker Hub](https://hub.docker.com/):SaaS平台

## 准备步骤or环境搭建
* [install docker](https://docs.docker.com/mac/step_one/)
* [run an image in a container](https://docs.docker.com/mac/step_two/)
* [build your own image](https://docs.docker.com/mac/step_four/)



## Docker架构

Docker使用的是客服端服务器的架构，Docker client与Docker daemon直接通信， Docker daemon负责build、run、distribute所有的Docker容器。client和daemon既可以运行在同一台机器上，也可以吧daemon部署在远程服务器，然后用client连接远程daemon进行工作，在远程工作模式下，client和daemon通过socket或者RESTful API进行通信。

![Docker architecture](/images/docker_architecture.svg)


### 几个关键概念
* Docker images, images是一个只读的模版，可以包含一个部属了代码的ubuntu操作系统。images被用来创建container。
* Docker registries，registries管理着images，是Docker hub的一部分，里面包含了大量现成的images供使用。
* Docker container， container就像一个文件夹，里面包含了程序运行所需要的一切。


### Docker image的工作机制

每个image包含很多层，Docker用[union file system](http://en.wikipedia.org/wiki/UnionFS)把这些层封装成单个image。

这种层级结构使得Docker非常的轻量级，当你改变了一个image（比如升级代码到最新版本）之后，Docker就会build出一个新的层，但是不会像虚拟机的整个image替换那样，而只用发布更新的部分，这种机制使得发布Docker image非常的方便。

每个image都是从base image衍生出来，例如 ubuntu、fedora

Docker都是根据指令集从base image衍生出新的层，每个指令生成新的一层。

* Run a command
* Add a file or directory
* Create an environment variable
* What process to run when launching the container from this image

这些指令都保存在一个叫做Dockerfile的文件中



### Docker registry的工作机制

Docker registry是用来存放docker images的地方。我们创建的docker image都可以push到Docker registry里。既可以是[Docker Hub](https://hub.docker.com/)这种公共的registry，也可以是内网的registry。


### Docker container的工作机制

一个container包含了操作系统、用户自己添加的文件、meta-data。Docker image是只读的，当Docker在image上运行了一个container的时候，docker在image之上增加了一个读写层，我们的应用程序就运行在这个读写层里。


### 当我们运行一个container的时候发生了什么

我们通过以下命令来跟docker daemon交互，告诉daemon来执行命令

$ docker run -i -t ubuntu /bin/bash


* Pulls the ubuntu image
* creates a new container
* Allocates a filesystem and mounts a read-write layer
* Allocates a network/bridge interface
* Sets up an IP address
* Executes a process that you specify
* Captures and provides application output
