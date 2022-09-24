---
title: MyBatis使用示例
categories:
    - 编程
        - 框架
tags:
    - MyBatis
---


在这篇文档中将给出MyBatis的使用例子，让自己在例子中进行学习。

## 一、准备MySQL

直接使用Docker来创建一个MySQL容器使用了。创建容器的命令如下所示：

```bash
$ docker run --name my-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=<设置root密码> -e MYSQL_USER=<设置用户名> -e MYSQL_PASSWORD=<设置用户密码> -e MYSQL_DATABASE=test  -d mysql:oracle
```

然后登录到MySQL容器中，使用下面的命令进入mysql:

```bash
$ mysql -u <用户名> -p          # 回车之后输入用户密码，就可以登录进入mysql了
```

接下来需要创建表格并插入数据：

```mysql
-- 创建表格
CREATE TABLE IF NOT EXISTS `Blog`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `title` VARCHAR(100) NOT NULL,
   `author` VARCHAR(40) NOT NULL,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```mysql
-- 插入数据
INSERT INTO Blog (title, author) VALUES ( '人生困惑20讲', '哈哈');
```

<!-- more -->

## 二、编写MyBatis示例代码

先声明，使用maven构建的工程。我们首先看一下整个工程的文件目录结构：

```bash
studyMyBatis/
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── org
    │   │       └── mybatis
    │   │           └── example
    │   │               ├── Blog.java
    │   │               ├── BlogMapper.java
    │   │               └── Example.java
    │   └── resources
    │       └── org
    │           └── mybatis
    │               └── example
    │                   ├── BlogMapper.xml
    │                   └── mybatis-config.xml
    └── test
        └── java
```



### 1、添加MyBatis和其他需要使用的依赖。

```xml
<!-- MyBatis依赖 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.6</version>
</dependency>
<!-- mysql驱动依赖 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
<!-- lombok方便生成getter、setter方法 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.24</version>
    <scope>provided</scope>
</dependency>
```



### 2、MyBatis配置文件

文件名：mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!-- 使用MyBatis时的配置文件 -->
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/> <!-- 配置事务类型 -->
            <dataSource type="POOLED"> <!-- 数据源 -->
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/test"/>
                <property name="username" value="<用户名>"/>
                <property name="password" value="<用户密码>"/>
            </dataSource>
        </environment>
    </environments>
    <mappers> <!-- 指定Mapper的配置 -->
        <mapper resource="org/mybatis/example/BlogMapper.xml"/>
    </mappers>
</configuration>
```



### 3、映射器配置文件

文件名：BlogMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- BlogMapper映射器的配置文件 -->
<mapper namespace="org.mybatis.example.BlogMapper">
    <select id="selectBlog" resultType="org.mybatis.example.Blog">
        select *
        from Blog
        where id = #{id}
    </select>
</mapper>
```



### 4、映射器接口和实体类



映射器接口

```java
package org.mybatis.example;

/**
 * 使用MyBatis时对于Blog类型的映射器
 */
public interface BlogMapper {
    Blog selectBlog(int i);
}
```



实体类

```java
package org.mybatis.example;

import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

/**
 * 描述博客文章的类型
 */
@Getter
@Setter
@ToString
public class Blog {

    private int id;

    private String title;

    private String author;
}
```



### 5、示例程序

所有的准备代码都有了，最后就该写我们的示例程序了。

```java
package org.mybatis.example;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

/**
 * 演示MyBatis的基本使用
 */
public class Example {

    public static void main(String[] args) throws IOException {
        String resource = "org/mybatis/example/mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        try (SqlSession session = sqlSessionFactory.openSession()) {
            BlogMapper mapper = session.getMapper(BlogMapper.class);
            Blog blog = mapper.selectBlog(1);
            System.out.println("查询结果: " + blog);
        }
    }
}
```

执行结果如下所示：

```bash
查询结果: Blog(id=1, title=人生困惑20讲, author=哈哈)
```



## 三、总结

这个例子虽然简单，但是使用MyBatis时基本的知识都涉及到了。配置文件、映射器、SqlSessionFactoryBuilder、sqlSessionFactory、SqlSession都有体现。如果要深入了解某一个方面的话，阅读参考文档的相关章节就可以了。

