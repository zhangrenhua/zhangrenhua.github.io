---
title: Docker入门实战-SSH连接docker容器
description: Docker 是一个开源项目，诞生于 2013 年初，最初是 dotCloud 公司内部的一个业余项目。它基于 Google 公司推出的 Go 语言实现。 项目后来加入了 Linux 基金会，遵从了 Apache 2.0 协议，项目代码在 GitHub 上进行维护。</br></br>Docker 自开源后受到广泛的关注和讨论，以至于 dotCloud 公司后来都改名为 Docker Inc。Redhat 已经在其 RHEL6.5 中集中支持 Docker；Google 也在其 PaaS 产品中广泛应用。</br></br>Docker 项目的目标是实现轻量级的操作系统虚拟化解决方案。 Docker 的基础是 Linux 容器（LXC）等技术。</br></br>在 LXC 的基础上 Docker 进行了进一步的封装，让用户不需要去关心容器的管理，使得操作更为简便。用户操作 Docker 的容器就像操作一个快速轻量级的虚拟机一样简单。</br></br>下面的图片比较了 Docker 和传统虚拟化方式的不同之处，可见容器是在操作系统层面上实现虚拟化，直接复用本地主机的操作系统，而传统方式则是在硬件层面实现。</br><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/docker-jj-1.png"></br><img alt="Alt text" src="http://7xoqbc.com1.z0.glb.clouddn.com/docker-jj-2.png">
tags: [虚拟化,docker]
categories: docker
date: 2016-1-1 13:38:10
---

## 简介

### 什么是Docker

Docker 是一个开源项目，诞生于 2013 年初，最初是 dotCloud 公司内部的一个业余项目。它基于 Google 公司推出的 Go 语言实现。 项目后来加入了 Linux 基金会，遵从了 Apache 2.0 协议，项目代码在 GitHub 上进行维护。

Docker 自开源后受到广泛的关注和讨论，以至于 dotCloud 公司后来都改名为 Docker Inc。Redhat 已经在其 RHEL6.5 中集中支持 Docker；Google 也在其 PaaS 产品中广泛应用。

Docker 项目的目标是实现轻量级的操作系统虚拟化解决方案。 Docker 的基础是 Linux 容器（LXC）等技术。

在 LXC 的基础上 Docker 进行了进一步的封装，让用户不需要去关心容器的管理，使得操作更为简便。用户操作 Docker 的容器就像操作一个快速轻量级的虚拟机一样简单。

下面的图片比较了 Docker 和传统虚拟化方式的不同之处，可见容器是在操作系统层面上实现虚拟化，直接复用本地主机的操作系统，而传统方式则是在硬件层面实现。

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-jj-1.png)

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-jj-2.png)

### 为什么要用Docker

作为一种新兴的虚拟化方式，Docker 跟传统的虚拟化方式相比具有众多的优势。

首先，Docker 容器的启动可以在秒级实现，这相比传统的虚拟机方式要快得多。 其次，Docker 对系统资源的利用率很高，一台主机上可以同时运行数千个 Docker 容器。

容器除了运行其中应用外，基本不消耗额外的系统资源，使得应用的性能很高，同时系统的开销尽量小。传统虚拟机方式运行 10 个不同的应用就要起 10 个虚拟机，而Docker 只需要启动 10 个隔离的应用即可。

具体说来，Docker 在如下几个方面具有较大的优势。

#### 更快速的交付和部署

对开发和运维（devop）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。

开发者可以使用一个标准的镜像来构建一套开发容器，开发完成之后，运维人员可以直接使用这个容器来部署代码。 Docker 可以快速创建容器，快速迭代应用程序，并让整个过程全程可见，使团队中的其他成员更容易理解应用程序是如何创建和工作的。 Docker 容器很轻很快！容器的启动时间是秒级的，大量地节约开发、测试、部署的时间。

#### 更高效的虚拟化

Docker 容器的运行不需要额外的 hypervisor 支持，它是内核级的虚拟化，因此可以实现更高的性能和效率。

#### 更轻松的迁移和扩展

Docker 容器几乎可以在任意的平台上运行，包括物理机、虚拟机、公有云、私有云、个人电脑、服务器等。 这种兼容性可以让用户把一个应用程序从一个平台直接迁移到另外一个。

#### 更简单的管理

使用 Docker，只需要小小的修改，就可以替代以往大量的更新工作。所有的修改都以增量的方式被分发和更新，从而实现自动化并且高效的管理。

#### 对比传统虚拟机总结

特性 | 容器 | 虚拟机
--- | --- | ---
启动 | 秒级 | 分钟即
硬盘使用 | 一般为MB | 一般为GB
性能 | 接近原生 | 弱于
系统支持量 | 单机支持上千个容器 | 一般几十个

## 安装Docker

本来打算在CentOS6上安装Docker，最终由于CentOS6上自带的kernel版本太低导致Docker启动失败而放弃（kernel升级太繁琐）。

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-start-error.png)

<font color="red">下面使用CentOS7安装Docker，本人采用虚拟机的方式来安装，安装好的虚拟机必须保证能访问外网。</font>

CentOS7 系统 CentOS-Extras 库中已带 Docker，可以直接安装：
```
$ sudo yum install docker
```

安装之后启动 Docker 服务，并让它随系统启动自动加载：
```
$ sudo service docker start
$ sudo chkconfig docker on
```


## 获取镜像

可以使用 docker pull 命令来从仓库获取所需要的镜像。

下面的例子将从 Docker Hub 仓库下载一个Centos6并且安装了jdk7的镜像：
```
$ docker pull tcbenkhard/centos6-jdk7
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-pull-image.png)

## 列出本地镜像

使用 docker images 显示本地已有的镜像。
```
$ docker images
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-images.png)

## 启动容器

启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止状态（stopped）的容器重新启动。

因为 Docker 的容器实在太轻量级了，很多时候用户都是随时删除和新创建容器。
下面的命令则启动一个 bash 终端，允许用户进行交互。
```
$ docker run -t -i docker.io/tcbenkhard/centos6-jdk7 /bin/bash
[root@ffe81683c404 /]#
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-run-hello.png)

其中，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上，-i 则让容器的标准输入保持打开。

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：
（1）检查本地是否存在指定的镜像，不存在就从公有仓库下载
（2）利用镜像创建并启动一个容器
（3）分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
（4）从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
（5）从地址池配置一个 ip 地址给容器
（6）执行用户指定的应用程序
（7）执行完毕后容器被终止

可以使用下面命令来查看CentOS版本信息：
```
$ cat /etc/redhat-release
```

## 修改root密码

使用passwd密码来修改密码（如提示没有这个命令行使用yum install passwd安装）：
```
$ passwd
    xxx密码
    xxx确认密码
```

## 安装Openssh

使用下面命令安装ssh server/ssh client：
```
$ sudo yum -y install openssh-server
$ sudo yum -y install openssh-clients
```

修改SSH配置文件以下选项，去掉#注释，将四个选项启用：
```
$ vi /etc/ssh/sshd_config

RSAAuthentication yes #启用 RSA 认证
PubkeyAuthentication yes #启用公钥私钥配对认证方式
AuthorizedKeysFile .ssh/authorized_keys #公钥文件路径（和上面生成的文件同）
PermitRootLogin yes #root能使用ssh登录
```

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-ssh-hello.png)


重启ssh服务，并设置开机启动：
```
$ service sshd restart
$ chkconfig sshd on
```

## 退出容器并保存更改

使用exit命令或者ctrl+C来退出当前运行的容器：
```
[root@ffe81683c404 /]# exit
```

注意：上面`ffe81683c404`是容器的ID，退出后用于保存的唯一ID。

当结束后，我们使用 exit 来退出，现在我们的容器已经被我们改变了，使用 docker commit 命令来提交更新后的副本。
```
$ sudo docker commit -m 'install openssh' -a 'Docker Newbee' ffe81683c404  centos6-jdk7:ssh
4f177bd27a9ff0f6dc2a830403925b5360bfe0b93d476f7fc3231110e7f71b1c
```

其中，-m 来指定提交的说明信息，跟我们使用的版本控制工具一样；-a 可以指定更新的用户信息；之后是用来创建镜像的容器的ID；最后指定目标镜像的仓库名和 tag 信息。创建成功后会返回这个镜像的 ID 信息。

提交后docker中就会多出一个centos6-jdk7:ssh的一个镜像。
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-commit-hello.png)

## 启动新的容器并打通22端口

将新的镜像启动，并将docker服务器的50001端口映射到容器的22端口上：
```
$ docker run -d -p 50001:22 centos6-jdk7:ssh /usr/sbin/sshd -D
```

ssh连接容器：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/docker-ssh-connect.png)

至此SSH连接docker容器成功完成。

## 参考资料
- http://dockerpool.com/static/books/docker_practice/introduction/README.html
- http://blog.sina.com.cn/s/blog_600e56a60102vwjc.html
