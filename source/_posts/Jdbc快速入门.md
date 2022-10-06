---
title: Jdbc快速入门
categories:
    - Java
        - 数据库
tags:
    - Jdbc
---


在这篇文章中，我们将使用Jdbc来操作MySQL数据库。


## 一、导入MySQL驱动包依赖

由于是使用maven管理的依赖，就直接给出依赖坐标了，如果是gradle可以改下写法就好了。

```xml
<dependencies>
    <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.30</version>
    </dependency>
</dependencies>
```

<!-- more -->

## 二、运行MySQL容器

为了图省事方便，直接使用本地docker运行mysql的容器，执行下面的命令就好了。

```bash
$ docker run --name mucao-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_USER=mucao -e MYSQL_PASSWORD=1 23456 -e MYSQL_DATABASE=test  -d mysql:oracle
```



## 三、完整的Jdbc示例程序

编写jdbc操作mysql的程序就三个步骤：加载驱动、获取连接、执行语句，下面是完整的示例程序，完成的功能非常简单，就是从数据库中查询学生信息，然后打印到控制台。

```java
package com.mucao;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class Example {

    private static final String URL = "jdbc:mysql://localhost:3306/test";

    private static final String USER = "mucao";

    private static final String PASSWORD = "123456";

    public static void main(String[] args) throws Exception {
        //1.加载驱动程序
        // Class.forName("com.mysql.jdbc.Driver"); 现在已经是通过SPI注册驱动程序了，不再需要手动注册。
        //2. 获得数据库连接
        Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
        //3.操作数据库，实现增删改查
        Statement statement = conn.createStatement();
        ResultSet resultSet = statement.executeQuery("SELECT * FROM students");
        //如果有数据，resultSet.next()返回true
        while (resultSet.next()) {
            System.out.println("名称: " + resultSet.getString("name") +
                    "  年龄: " + resultSet.getInt("age") +
                    " 年级: " + resultSet.getString("grade") );
        }
    }
}
```



## 四、总结

Jdbc简化了Java程序连接数据库的逻辑，并且操作数据库的步骤被统一，程序员不用关注每一个不同数据库的连接细节。

