---
title: 使用JDBC连接H2database数据库
categories:
    - Java
        - 数据库
tags:
    - H2database
---


H2database可以当作应用中嵌入的数据库使用，这篇文章就来实践一下。

## 一、引入H2依赖

我们使用maven来管理工程，下面是H2数据库依赖的坐标：

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.1.214</version>
</dependency>
```

<!-- more -->


## 二、连接H2数据库

```java
import java.sql.*;
public class H2databaseExample {
    public static void main(String[] a) throws Exception {
        Connection conn = DriverManager.getConnection("jdbc:h2:~/test", "", "");
        // add application code here
        conn.close();
    }
}
```

首先得保证用户家目录下存在`test`目录，然后在使用`DriverManager`获取数据库连接的时候用户名和密码都设置为空。



### 三、执行SQL语句

现在已经创建了连接，那么我们在此基础上来执行下SQL语句吧，代码如下所示：

```java
package com.mucao;

import java.sql.*;

public class H2databaseExample {
    public static void main( String[] args ) throws SQLException {
        Connection conn = DriverManager.getConnection("jdbc:h2:~/h2database/test", "", "");
        Statement statement = conn.createStatement();
        statement.execute("CREATE TABLE IF NOT EXISTS test (id INT PRIMARY KEY , name VARCHAR(255))");
        statement.execute("INSERT INTO test VALUES (1, 'hello')");
        statement.execute("INSERT INTO test VALUES (2, 'world')");
        ResultSet resultSet = statement.executeQuery("SELECT * FROM test ORDER BY id");
        while (resultSet.next()) {
            System.out.println("id: " + resultSet.getInt("id") + " name: " + resultSet.getString("name"));
        }
        statement.execute("UPDATE test SET name = 'hi' WHERE id = 1");
        statement.execute("DELETE FROM test WHERE id = 2");
        conn.close();
    }
}
```

执行结果如下所示：

```bash
id: 1 name: hello
id: 2 name: world
```



## 四、总结

简单的小节一下，发现H2作为应用嵌入式数据库使用起来非常的方便。在应用开发初期或者对数据库性能要求不高的时候，使用H2作为嵌入式数据库可以降低开发成本和维护成本。



## 参考资料

1、http://www.h2database.com/html/tutorial.html#connecting_using_jdbc

