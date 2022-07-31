---
title: Docker容器创建过程解析
categories:
    - 编程
        - docker
tags:
    - docker容器创建
---

## 一、docker run命令的背后发生了什么？

平时在创建docker容器的时候会使用`docker run`命令，并给定一些参数，那么在执行`docker run`时docker在背后到底为我们做了哪些事情呢？

### 1、docker常用命令执行流程

先来看官方给出的一张容器创建过程图：

<img src="images/docker创建容器流程图-官方.png" alt="img" style="zoom: 50%;" />

这张图描述了`docker build`、`docker pull`和`docker run`三个命令在Client、Docker daemon和Regitry之间的交互流程。接下来我们着重分析`dockr run`命令是如何执行的。

<!-- more -->

### 2、docker run命令执行流程

先假设我们要创建一个ubuntu容器，那么首先会执行下面的命令：

```bash
docker run -i -t ubuntu /bin/bash
```

执行上述命令后，docker会为我们启动一个ubuntu容器，那么docker在后台都执行了哪些动作呢？

docker创建容器的大致过程可以用下面这幅图来描述：

<img src="images/docker_run大致流程.png" alt="docker_run大致流程" style="zoom: 33%;" />

首先你的操作系统上得要有一个docker daemon后台进程在运行，然后当执行上面这行命令时：

step1  docker client(即：docker终端命令行)会向docker daemon发送请求，启动一个容器；

step2  docker daemon会向host os(一般为linux)请求创建容器；

step3  linux会创建一个空的容器（可以简单理解为：一个未安装操作系统的裸机，只有虚拟出来的CPU、内存等硬件资源）

step4  docker daemon请检查本机是否存在docker镜像文件（可以简单理解为操作系统安装光盘），如果有，则加载到容器中（即：光盘插入裸机，准备安装操作系统）

step5  将镜像文件加载到容器中（即：裸机上安装好了操作系统，不再是裸机状态）

在docker完成上述步骤后，我们就得到了一个ubuntu"虚拟机"，然后就可以执行各种操作了。



如果在本地没有镜像文件，那么docker回到镜像仓库下载指定得镜像，下载到本地后，再进行装载到容器的操作。这种情况下创建容器的完整流程如下图所示：

<img src="images/docker拉取镜像创建容器的大致流程.png" alt="img" style="zoom:33%;" />



经过上面的分析，我们已经知道了docker创建的大致流程，但是有一些细节我们目前还是不知道的。步骤1中client给docker daemon发送了什么请求？是一个请求还是多个请求？步骤2和步骤3可以看作是一个大操作——在操作系统中（一般为Linux）创建容器。创建容器肯定是要分配资源的，那么docker是怎么向操作系统申请资源的？又申请了哪些资源？步骤4比较好理解，就是在本地或远端寻找镜像文件，但是步骤5又是怎么把镜像文件加载到新创建的空容器中的呢？

一番搜索后并没有得到满意的答案，为了清楚的了解上述问题，我们接下来将探索docker源码相关部分。



## 二、docker内部是怎么完成容器创建步骤的？

上一节已经梳理清楚了docker创建容器的流程，这一节我们就来探索一下docker内部是怎么完成容器创建每一个步骤的？

### 1、创建容器时，client向docker daemon发送了什么请求？

首先看一下在docker client模块中处理`docker run`命令的函数代码：

```go
// integration/internal/container/container.go文件
// Run creates and start a container with the specified options
func Run(ctx context.Context, t *testing.T, client client.APIClient, ops ...func(*TestContainerConfig)) string {
	t.Helper()
	id := Create(ctx, t, client, ops...) // step1 创建容器

	err := client.ContainerStart(ctx, id, types.ContainerStartOptions{}) // step2 开启容器
	assert.NilError(t, err)

	return id
}
```





## 参考资料

1、[run执行流详解（以volume，network和libcontainer为线索）](https://www.dockone.io/article/1239)

2、[runc原理概述](https://blog.csdn.net/qq_33339479/article/details/122134279)
