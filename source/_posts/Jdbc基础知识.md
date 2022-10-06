---
title: Jdbc基础知识
categories:
    - Java
        - 数据库
tags:
    - Jdbc
---


## 一、JDBC的定义

Java 数据库连接 (JDBC) API 提供来自 Java 编程语言的通用数据访问。使用 JDBC API，您几乎可以访问任何数据源，从关系数据库到电子表格和平面文件（flat files）。 JDBC 技术还提供了一个通用基础，可以在此基础上构建工具和替代接口。

JDBC Api由下面两个包组成：

- [`java.sql`](https://docs.oracle.com/javase/8/docs/api/java/sql/package-summary.html)
- [`javax.sql`](https://docs.oracle.com/javase/8/docs/api/javax/sql/package-summary.html)

当你下载Java SE 8 的时候就会自动获取这两个包。

要将 JDBC API 与特定的数据库管理系统一起使用，您需要基于 JDBC 技术的驱动程序在 JDBC 技术和数据库之间进行调解。根据各种因素，驱动程序可能纯粹用 Java 编程语言编写，也可能混合使用 Java 编程语言和 Java 本机接口 (JNI) 本机方法。要获取特定数据库管理系统的 JDBC 驱动程序，请参阅 [JDBC 数据访问 API](http://www.oracle.com/technetwork/java/javase/tech/index-jsp-136101.html)

<!-- more -->

## 二、JDBC执行流程

执行流程分为三步，如下所示：

- 连接数据源，如：数据库。
- 为数据库传递查询和更新指令。
- 处理数据库响应并返回的结果。


## 三、JDBC架构

分为双层架构和三层架构。

### 双层架构

![Two-tier-Architecture-for-Data-Access](images/Two-tier-Architecture-for-Data-Access.gif)

作用：此架构中，Java Applet 或应用直接访问数据源。

条件：要求 Driver 能与访问的数据库交互。

机制：用户命令传给数据库或其他数据源，随之结果被返回。

部署：数据源可以在另一台机器上，用户通过网络连接，称为 C/S配置（可以是内联网或互联网）

### 三层架构

![Three-tier-Architecture-for-Data-Access](images/Three-tier-Architecture-for-Data-Access.gif)

侧架构特殊之处在于，引入中间层服务。

流程：命令和结构都会经过该层。

吸引：可以增加企业数据的访问控制，以及多种类型的更新；另外，也可简化应用的部署，并在多数情况下有性能优势。

历史趋势： 以往，因性能问题，中间层都用 C 或 C++ 编写，随着优化编译器（将 Java 字节码 转为 高效的 特定机器码）和技术的发展，如EJB，Java 开始用于中间层的开发这也让 Java 的优势突显出现出来，使用 Java 作为服务器代码语言，JDBC随之被重视。



## 四、JDBC编程步骤

加载驱动程序：

```java
Class.forName(driverClass)
//加载MySql驱动
Class.forName("com.mysql.jdbc.Driver")
//加载Oracle驱动
Class.forName("oracle.jdbc.driver.OracleDriver")
```

获得数据库连接：

```java
DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/imooc", "root", "root");
```

创建Statement\PreparedStatement对象：

```java
conn.createStatement();
conn.prepareStatement(sql);
```



## 参考资料

1、[java官方资料](https://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/)

2、[菜鸟教程-JDBC](https://www.runoob.com/w3cnote/jdbc-use-guide.html)

3、[MySQL Connector/J 8.0 Developer Guide](https://dev.mysql.com/doc/connector-j/8.0/en/)



