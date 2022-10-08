---
title: H2database快速入门
categories:
    - Java
        - 数据库
tags:
    - H2database
---

## 在一个应用中嵌入H2数据库

这个数据库可以被用在嵌入式模式或者服务器模式。在嵌入式模式中使用H2，你需要做下面的事情：

- 添加`h2*.jar`到类路径（**H2**数据库没有任何的依赖）
- 使用JDBC驱动类：`org.h2.Driver`
- 数据库连接URL`jdbc:h2:~/test`会在你的用户家目录下打开一个数据库`test`
- 一个新的数据库会被自动的创建



## H2命令行应用

命令行可以让你使用浏览器接口访问SQL数据库。

![Web Browser - H2 Console Server - H2 Database](images/console-2.png)

<!-- more -->

如果你没有Windows系统，或者某些事情不能如期工作，请参考[Tutorial](http://www.h2database.com/html/tutorial.html)。

### 实践步骤

#### 安装

使用windows installer安装**H2**软件。下载地址为：http://www.h2database.com/html/download.html

#### 打开命令行

点击[start]，[All Programs], [H2], 和 [H2 Console(Command Line)]:

![image-20221008210903375](images/quickstart-1.png)

一个新的命令行窗口就出现了。

![image-20221008211831815](images/image-20221008211831815.png)

同时一个新的浏览器页面也会打开URL:  http://localhost:8082。你也许会被防火墙告警。如果你不想网络上的其他电脑访问你机器上的数据库，你可以让防火墙拦截那些请求。在本次教程中只需要本地连接就可以了。

#### 登录

选择[Generic H2]并且点击[Connect]:

![quickstart-2](images/quickstart-2.png)

现在你就登录到H2数据库了。

#### 示例

点击[Sample SQL Script]:

![image-20221008212611418](images/image-20221008212611418.png)

SQL命令会出现命令行区域。

#### 执行SQL

点击[Run]按钮。

![image-20221008212814235](images/image-20221008212814235.png)

在页面的左侧，一个新的`TEST`入口会被添加到数据库图标下面。操作和语句的结果展示在下面的脚本中。

![image-20221008213039039](images/image-20221008213039039.png)

#### 关闭连接

点击[Disconnect]来关闭与H2数据库的连接。

#### 结束

关闭命令行窗口就退出了整个H2数据库。



## 参考资料

1、Quickstart：http://www.h2database.com/html/quickstart.html