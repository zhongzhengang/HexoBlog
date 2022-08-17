---
title: Java远程调试各参数说明
categories:
    - Java
tags:
    - 远程调试
---


首先，JAVA自身支持调试功能，并提供了一个简单的调试工具JDB，类似于功能强大的GDB，JDB也是一个字符界面的调试环境，并支持设置断点，支持线程级的调试。

JAVA的调试方法如下：

## 1、首先设置JVM参数，使之工作在DEBUG模式下，加入参数：

```bash
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,address=5432,suspend=n,onthrow=java.io.IOException,launch=/sbin/echo
```

其中，`-Xdebug`是通知JVM工作在DEBUG模式下，`-Xrunjdwp`是通知JVM使用(java debug wire protocol)来运行调试环境。该参数同时了一系列的调试选项：

- `transport`指定了调试数据的传送方式，
- `dt_socket`是指用SOCKET模式，另有`dt_shmem`指用共享内存方式，其中，dt_shmem只适用于Windows平台。
- `server`参数是指是否支持在server模式的VM中.
- `onthrow`指明，当产生该类型的Exception时，JVM就会中断下来，进行调式。该参数可选。
- `launch`指明，当JVM被中断下来时，执行的可执行程序。该参数可选
- `suspend`指明，是否在调试客户端建立起来后，再执行JVM。
- `onuncaught`(=y或n)指明出现uncaught exception 后，是否中断JVM的执行.

<!-- more -->

### 几个例子

example1

```bash
-Xrunjdwp:transport=dt_socket,server=y,address=8000
```

解释：在8000端口监听Socket连接，挂起VM并且不加载运行主函数直到调试请求到达

example2

```bash
-Xrunjdwp:transport=dt_shmem,server=y,suspend=n
```

解释：选择一个可用的共享内存（因为没有指address）并监听该内存连接，同时加载运行主函数

example3

```bash
-Xrunjdwp:transport=dt_socket,address=myhost:8000
```

解释：连接到myhost:8000提供的调试服务（server=n，以调试客户端存在），挂起VM并且不加载运行主函数

example4

```bash
-Xrunjdwp:transport=dt_shmem,address=mysharedmemory
```

解释：通过共享内存的方式连接到调试服务，挂起VM并且不加载运行主函数

example5

```bash
-Xrunjdwp: transport=dt_socket,server=y,address=8000,onthrow=java.io.IOException, launch=/usr/local/bin/debugstub
```

解释：等待java.io.IOException被抛出，然后挂起VM并监听8000端口连接，在接到调试请求后以命令`/usr/local/bin/debugstub dt_socket myhost:8000`执行

example6

```bash
-Xrunjdwp:transport=dt_shmem,server=y,onuncaught=y,launch=d:\bin\debugstub.exe
```

解释：等待一个`RuntimeException`被抛出，然后挂起VM并监听一个可用的共享内存，在接到调试请求后以命令`d:\bin\debugstub.exe dt_shmem <address>`执行,`<address>`是可用的共享内存



## 2、启动调试工具。

最简单的调试工具就是上面提到的JDB，以上述调试用JVM为例，可以用下面的命运行启动JDB：

```bash
$jdb -connect com.sun.jdi.SocketAttach:port=5432,hostname=192.168.11.213
```

另外，还有好多的可视化调试工具，如 eclipse,jsawt等等。Eclipses可用 ant debug来建立一个调试方法。

其实就是使用了JDK的[JPDA](https://blog.csdn.net/qq_38835878/article/details/114539300)，在启动服务器（Jboss或者Tomcat等）的命令行参数里面加上：
```bash
Xdebug -Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n
```



## 参考资料

1、https://www.iteye.com/blog/chainhou-1837059
