---
title: 异步操作测试工具Awaitility
categories:
    - 编程
        - 测试
tags:
    - Awaitility
---


平时在写单元测试的时候，经常会遇到一些异步场景，不确定异步线程中当前的执行状态是否已经达到自己预期的观测点，这个时候我们都会使用`Thread.sleep(time)`方式武断的等待一段时间，或者自己写个while死循环来不停的检测某个条件是否满足。前一种方式会浪费测试用例执行时间，后一种方式不够优雅。awaitility工具就是用来解决这个场景的。

<!-- more -->

## 一、概述

异步系统难以测试。不仅因为需要处理多线程、超时和并发问题，还因为这些细节会遮蔽测试代码的意图。Awaitility是一个DSL，允许你准确且易读地表达对于异步系统的期望。示例如下：

```java
@Test
public void updatesCustomerStatus() {
    // Publish an asynchronous message to a broker (e.g. RabbitMQ):
    messageBroker.publishMessage(updateCustomerStatusMessage);
    // Awaitility lets you wait until the asynchronous operation completes:
    await().atMost(5, SECONDS).until(customerStatusIsUpdated());
    ...
}
```



## 二、安装

Maven:

```xml
<dependency>
      <groupId>org.awaitility</groupId>
      <artifactId>awaitility</artifactId>
      <version>4.2.0</version>
      <scope>test</scope>
</dependency>
```

Gradle（Groovy DSL）:

```groovy
testImplementation 'org.awaitility:awaitility:4.2.0'
```



## 三、静态导入

为了有效地使用Awaitility，建议从Awaitility框架静态导入如下方法：

- `org.awaitility.Awaitility.*`

导入下面这些方法可能也是有用的：

- `java.time.Duration.*`
- `java.util.concurrent.TimeUnit.*`
- `org.hamcrest.Matchers.*`
- `org.junit.Assert.*`



## 四、使用示例

### 简单用法

假设我们发送"add user"消息给我们的异步系统，就像下面这样：

```java
publish(new AddUserMessage("Awaitility Rocks"));
```

在你的测试用例中，Awaitility可以帮助你轻松的检测数据库已经被更新了。最简单的形式就像下面这样：

```java
await().until(newUserIsAdded());
```

在测试用例中你自己得实现`newUserIsAdded`方法。只有当它指定的条件被满足了，Awaitility才会停止等待。

```java
private Callable<Boolean> newUserIsAdded() {
	return () -> userRepository.size() == 1; // The condition that must be fulfilled
}
```

你当然也可以内联`newUserIsAdded`方法，就像下面这样写：

```java
await().until(() -> userRepository.size() == 1);
```

默认情况下Awaitility将会等待10秒钟，并且当到这个时间的时候如果用户仓库的数据量不等于1，那么它就会抛出一个[ConditionTimeoutException](http://static.javadoc.io/org.awaitility/awaitility/4.2.0/org/awaitility/core/ConditionTimeoutException.html)异常来使测试失败。如果你想要不同的超时时间，你可以像下面这样定义：

```java
await().atMost(5, SECONDS).until(newUserWasAdded());
```

### 复用等待条件

Awaitility也支持分割等待条件为数据提供（supplying）和条件匹配（matching）部分，这样可以更好的使用。下面是例子：

```java
await().until( userRepositorySize(), equalTo(1) );
```

`userRepositorySize`方法现在是一个`Integer`类型的`Callable`:

```java
private Callable<Integer> userRepositorySize() {
      return () -> userRepository.size(); // The suppling part of the condition
}
```

`equalTo` 是一个标准的 [Hamcrest](http://code.google.com/p/hamcrest/) 匹配器（matcher），为Awaitility指定条件匹配部分。

现在我们可以在不同的测试用例中重用`userRepositorySize`。假设我们有一个测试用例在同一时间添加三个用户（user）对象。

```java
publish(new AddUserMessage("User 1"), new AddUserMessage("User 2"), new AddUserMessage("User 3"));
```

我们现在重用`userRepositorySize` "condition supplier"，并且简单的更新Hamcrest匹配器：

```java
await().until( userRepositorySize(), equalTo(3) );
```

在下面这种简单的例子中，你当然也可以利用Java的方法引用：

```java
await().until(userRepository::size, equalTo(3) );
```

### 字段类型条件

你也可以通过字段引用来构建数据提供部分，例如：

```java
await().until( fieldIn(object).ofType(int.class), equalTo(2) );
```

或者:

```java
await().until( fieldIn(object).ofType(int.class).andWithName("fieldName"), equalTo(2) );
```

或者:

```java
await().until( fieldIn(object).ofType(int.class).andAnnotatedWith(MyAnnotation.class), equalTo(2) );
```

### Atomic, Adders and Accumulators

如果你正在使用原子类型（Atomic），Awaitility提供简单的方法设置等待条件，直到原子类变量满足特定的值。

```java
AtomicInteger atomic = new AtomicInteger(0);
// Do some async stuff that eventually updates the atomic integer
await().untilAtomic(atomic, equalTo(1));
```

等待一个`AtomicBoolean`更加简单：

```java
AtomicBoolean atomic = new AtomicBoolean(false);
// Do some async stuff that eventually updates the atomic boolean
await().untilTrue(atomic);
```

如果你在使用Adders，例如 [LongAdder](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAdder.html)，Awaitility允许你简单等待它到达一个特定值：

```java
await().untilAdder(myLongAdder, equalTo(5L))
```

Likewise, if using Accumulators such as [LongAccumulator](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAccumulator.html), you can do:

同样地，如果使用Accumulators例如 [LongAccumulator](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/LongAccumulator.html)，你可以像下面这样做：

```java
await().untilAccumulator(myLongAccumulator, equalTo(5L))
```



### Lambdas

你可以条件中使用lambda表达式：

```java
await().atMost(5, SECONDS).until(() -> userRepository.size() == 1);
```

或者方法引用:

```java
await().atMost(5, SECONDS).until(userRepository::isNotEmpty);
```

或者组合方法引用和Hamcrest匹配器（matchers）:

```java
await().atMost(5, SECONDS).until(userRepository::size, is(1));
```

你也可以使用断言：

```java
await().atMost(5, SECONDS).until(userRepository::size, size -> size == 1);
```

使用示例参考： [Jayway team blog](http://www.jayway.com/2014/04/23/java-8-and-assertj-support-in-awaitility-1-6-0/).

### 使用AssertJ或者Fest Assert

你可以使用 [AssertJ](http://joel-costigliola.github.io/assertj/) 或 [Fest Assert](https://code.google.com/p/fest/) 作为Hamcrest的替代者。实际上可以使用任意的第三方库，只要它会在发生错误的时候抛异常。

```java
await().atMost(5, SECONDS).untilAsserted(() -> assertThat(fakeRepository.getValue()).isEqualTo(1));
```

### At Least

你可以使用Awaitility等待至少某段时间。例如：

```java
await().atLeast(1, SECONDS).and().atMost(2, SECONDS).until(value(), equalTo(1));
```

例子中，在由`atLeast`指定的时长之前条件会被满足，否则会抛出一个异常来表明在指定的时长之前条件可能不会被满足。


### 断言值被保持

从Awaitility 4.0.2开始，可以断言某个值被保持了指定的时长。例如，如果你需要确定在数据库中的某个值在1500毫秒中被保持了800毫秒，就可以像下面这样做：

```java
await().during(800, MILLISECONDS).atMost(1500, MILLISECONDS).until(() -> myRepository.findById("id"), equalTo("something"));
```

Awaitility最多等待1500毫秒，并且会检测`myRepository.findById("id")`必须等于"something"至少800毫秒。



## 五、重要事项

Awaitility不会确保线程安全和线程同步！那是你自己的责任！确保你的代码正确处理了同步，或者你正在使用线程安全的数据结构，例如volatile字段或类，比如AtomicInteger和ConcurrentHashMap。



**参考资料:**

1. GitHub地址：https://github.com/awaitility/awaitility
2. Getting started：https://github.com/awaitility/awaitility/wiki/Getting_started
3. 用户手册：https://github.com/awaitility/awaitility/wiki/Usage
4. [Awaitility test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/AwaitilityTest.java)
5. [Awaitility test case Java 8](https://github.com/awaitility/awaitility/blob/master/awaitility-java8-test/src/test/java/org/awaitility/AwaitilityJava8Test.java)
6. [Field supplier test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/UsingFieldSupplierTest.java)
7. [Atomic test case](https://github.com/awaitility/awaitility/blob/master/awaitility/src/test/java/org/awaitility/UsingAtomicTest.java)
8. [Presentation](http://awaitility.googlecode.com/files/awaitility-khelg-2011.pdf) from [Jayway](http://www.jayway.com/)'s KHelg 2011
