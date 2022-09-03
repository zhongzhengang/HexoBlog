---
title: Junit5设计分析（三）
categories:
    - Java
        - 测试
tags:
    - JUnit5
---

代码仓库：https://github.com/junit-team/junit5/

在"Junit5设计分析（一）"和"Junit5设计分析（二）"中，我们已经梳理清楚了Junit5执行测试的主流程、测试任务发现和参数解析。下面就是进入Junit5设计分析的终极目标——**Junit5顶层设计**。


## 一、执行任务的核心类

在执行测试任务的主干流程上，**Launcher**负责接收测试请求（request），然后会找到测试引擎**TestEngine**进行处理，测试引擎会将找到的测试任务**TestTask**提交给执行器**HierarchicalTestExecutorService**。

下面这幅图展示了它们之间的合作关系：

![Junit5源码分析-顶层设计](images/Junit5源码分析-顶层设计.png)

<!-- more -->

## 二、测试任务层次结构

测试引擎处理测试请求的结果就是生成具有层次结构的测试任务类。其实在“Junit5设计分析（一）”的主干流程中有程序执行中的测试任务层次结构，下面单独进行展示：

![Junit5源码分析-测试任务层次结构](images/Junit5源码分析-测试任务层次结构.png)

测试任务一共分为三个层次，分别是：引擎层次、测试类层次、测试方法层次，它们都对应着具体的类型。下面展示这些类型的层次关系：

![image-20220829110813045](images/三个层次测试任务层次关系.png)

从集成层次图中可以看到`JupiterEngineDescriptor`、`ClassTestDescriptor`、`TestMethodTestDescriptor`三个类都实现了`TestDescripter`接口，因此它们可以描述测试任务，只是对应的描述层次不同。并且，它们三个类都实现了`Node`接口，因此它们可以像树结构一样被组织起来。



## 三、Junit5中的设计模式

### 1、过滤器模式

在"Junit5设计分析（二）"中，我们跟读`DefaultLauncher#discoverRoot`方法的时候，看到了过滤器的应用。下面是对应的代码：

```java
// org.junit.platform.launcher.core.DefaultLauncher#discoverRoot方法
private Root discoverRoot(LauncherDiscoveryRequest discoveryRequest, String phase) {
    // 1、创建Root对象
    Root root = new Root();
	// 2、从每一个TestEngine中发现TestDescriptor
    for (TestEngine testEngine : this.testEngines) {
        // @formatter:off
        boolean engineIsExcluded = discoveryRequest.getEngineFilters().stream()
            .map(engineFilter -> engineFilter.apply(testEngine)) // ref-1
            .anyMatch(FilterResult::excluded);
        // @formatter:on

        if (engineIsExcluded) {
            ...... 打印日志
            continue;
        }
		...... 打印日志
		// 2.1 发现TestDescriptor
        Optional<TestDescriptor> engineRoot = discoverEngineRoot(testEngine, discoveryRequest);
        // 2、2 将发现的发现TestDescriptor添加到root对象中
        engineRoot.ifPresent(rootDescriptor -> root.add(testEngine, rootDescriptor));
    }
    root.applyPostDiscoveryFilters(discoveryRequest);
    // 3、删除掉没有测试任务的TestDescriptor
    root.prune();
    return root;
}
```

在`ref-1`处应用了过滤器。我们先来看看网上给出的过滤器模式的定义，然后再来看看Junit5中过滤器的实现，我觉得这样来学习过滤器模式更有实践意义。



#### 过滤器模式的定义

过滤器模式（Filter Pattern）或标准模式（Criteria Pattern）是一种设计模式，这种模式允许开发人员使用不同的标准来过滤一组对象，通过逻辑运算以解耦的方式把它们连接起来。这种类型的设计模式属于结构型模式，它结合多个标准来获得单一标准。



#### Junit5中过滤器的实现

其实过滤器模式比较简单，在Junit5中首先定义了一个`Filter`，如下所示：

```java
package org.junit.platform.engine;

@FunctionalInterface
@API(status = STABLE, since = "1.0")
public interface Filter<T> {
    ...... 省略其他静态方法
	/**
	 * Apply this filter to the supplied object.
	 */
	FilterResult apply(T object); // ref-2
}
```

网上查到的过滤器模式，是传入一组对象，然后调用过滤方法，返回一组结果对象。Junit5的`Filter`是在过滤单个对象，然后生成过滤结果。

我们再来看一个实现，加深一下理解：

```java
package org.junit.platform.launcher;

@API(status = STABLE, since = "1.0")
public class EngineFilter implements Filter<TestEngine> {

	public static EngineFilter includeEngines(String... engineIds) {
		return includeEngines(Arrays.asList(engineIds));
	}

	public static EngineFilter includeEngines(List<String> engineIds) {
		return new EngineFilter(engineIds, Type.INCLUDE);
	}

	public static EngineFilter excludeEngines(String... engineIds) {
		return excludeEngines(Arrays.asList(engineIds));
	}

	public static EngineFilter excludeEngines(List<String> engineIds) {
		return new EngineFilter(engineIds, Type.EXCLUDE);
	}

	private final List<String> engineIds;
	private final Type type;

	private EngineFilter(List<String> engineIds, Type type) {
		this.engineIds = validateAndTrim(engineIds);
		this.type = type;
	}

	@Override
	public FilterResult apply(TestEngine testEngine) {
		Preconditions.notNull(testEngine, "TestEngine must not be null");
		String engineId = testEngine.getId();
		Preconditions.notBlank(engineId, "TestEngine ID must not be null or blank");

		if (this.type == Type.INCLUDE) { // ref-3
			return includedIf(this.engineIds.stream().anyMatch(engineId::equals), //
				() -> String.format("Engine ID [%s] is in included list [%s]", engineId, this.engineIds), //
				() -> String.format("Engine ID [%s] is not in included list [%s]", engineId, this.engineIds));
		}
		else {
			return includedIf(this.engineIds.stream().noneMatch(engineId::equals), //
				() -> String.format("Engine ID [%s] is not in excluded list [%s]", engineId, this.engineIds), //
				() -> String.format("Engine ID [%s] is in excluded list [%s]", engineId, this.engineIds));
		}
	}

}
```

在`EngineFilter#apply`方法内的`ref-3`处，会依据当前过滤类型是包含还是排除来使用保存的一组“标准”（`engineIds`）对目标（`testEngine`）进行过滤。

我们可以看到`EngineFilter`代码逻辑比较简单，但是它提供的静态工厂方法和`includedIf`封装都值得学习，这些封装使得`EngineFilter`更加易用且阅读清晰。

小总结：**过滤器模式就是从一组对象中挑选出满足要求的对象**。



### 2、观察者模式

当单元测试执行的时候，会产生很多的变化，这些变化就形成了事件，如果有关心这些变化的对象，那么它们就可以向`Launcher`中注册监听器，当事件发生的时候会被通知到。

我们先来看看Junit5中使用观察者模式的地方。

```java
// org.junit.platform.engine.support.hierarchical.NodeTestTask#executeRecursively 方法
private void executeRecursively() {
    taskContext.getListener().executionStarted(testDescriptor); // ref-4
    started = true;

    throwableCollector.execute(() -> {
        // @formatter:off
        List<NodeTestTask<C>> children = testDescriptor.getChildren().stream()
            .map(descriptor -> new NodeTestTask<C>(taskContext, descriptor))
            .collect(toCollection(ArrayList::new));
        // @formatter:on

        context = node.before(context);

        DynamicTestExecutor dynamicTestExecutor = new DefaultDynamicTestExecutor();
        context = node.execute(context, dynamicTestExecutor);

        if (!children.isEmpty()) {
            children.forEach(child -> child.setParentContext(context));
            taskContext.getExecutorService().invokeAll(children);
        }

        dynamicTestExecutor.awaitFinished();
    });

    throwableCollector.execute(() -> node.after(context));
}
```

`NodeTestTask#executeRecursively`在"Junit5设计分析（一）"中就分析过了，不过当时是分析的Junit5单元测试主干流程。今天我们看看`ref-4`处的监听器调用部分。

`taskContext.getListener()`返回的是`EngineExecutionListener`，然后调用它的`executionStarted`方法，通知测试开始执行了。

还是和前面一样，我们先看看观察者模式的定义，然后再看看Junit5中的实际实现。



#### 观察者模式的定义



**模式动机**

建立一种对象与对象之间的依赖关系，一个对象发生改变时将自动通知其他对象，其他对象将相应做出反应。在此，发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间没有相互联系，可以根据需要增加和删除观察者，使得系统更易于扩展，这就是观察者模式的模式动机。



**模式定义**

观察者模式(Observer Pattern)：定义对象间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。观察者模式又叫做发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。

观察者模式是一种对象行为型模式。



**模式结构**

观察者模式包含如下角色：

- Subject: 目标
- ConcreteSubject: 具体目标
- Observer: 观察者
- ConcreteObserver: 具体观察者

![观察者模式_模式结构](images/Obeserver.jpg)

(图片来自于网络)



**时序图**

![观察者模式_时序图](images/seq_Obeserver.jpg)

当目标发生状态改变的时候，会通知观察者。



#### Junit5中观察者的实现

我们先来看看Junit5中观察者接口的定义：

```java
package org.junit.platform.engine;

@API(status = STABLE, since = "1.0")
public interface EngineExecutionListener {

	void dynamicTestRegistered(TestDescriptor testDescriptor);

	void executionSkipped(TestDescriptor testDescriptor, String reason);

	void executionStarted(TestDescriptor testDescriptor);

	void executionFinished(TestDescriptor testDescriptor, TestExecutionResult testExecutionResult);

	void reportingEntryPublished(TestDescriptor testDescriptor, ReportEntry entry);
}
```

可以看到`EngineExecutionListener`中并没有定义独立的事件类，而是不同的事件对应着不同的方法，`executionSkipped`、`executionStarted`和`executionFinished`三个方法最明显。

当有事件发生的时候，就会触发对应的方法，然后把相关的对象传递给具体的实现类。我们接下来看看都有哪些实现类：

<img src="images/image-20220902111014837.png" alt="image-20220902111014837" style="zoom:50%;" />

在IDEA中看，只有一个实现类，我们来详细看看这个实现类的细节：

```java
package org.junit.platform.launcher.core;

class ExecutionListenerAdapter implements EngineExecutionListener {

	private final TestPlan testPlan;
	private final TestExecutionListener testExecutionListener;

	ExecutionListenerAdapter(TestPlan testPlan, TestExecutionListener testExecutionListener) {
		this.testPlan = testPlan;
		this.testExecutionListener = testExecutionListener;
	}
	...... // 所有的方法实现都委派给了testExecutionListener
}
```

可以看到`ExecutionListenerAdapter`是一个适配器类，真正执行事件处理的是`TestExecutionListener`，这个类其实是接口。

```java
package org.junit.platform.launcher;

@API(status = STABLE, since = "1.0")
public interface TestExecutionListener {

	default void testPlanExecutionStarted(TestPlan testPlan) {
	}

	default void testPlanExecutionFinished(TestPlan testPlan) {
	}

	default void dynamicTestRegistered(TestIdentifier testIdentifier) {
	}

	default void executionSkipped(TestIdentifier testIdentifier, String reason) {
	}

	default void executionStarted(TestIdentifier testIdentifier) {
	}

	default void executionFinished(TestIdentifier testIdentifier, TestExecutionResult testExecutionResult) {
	}

	default void reportingEntryPublished(TestIdentifier testIdentifier, ReportEntry entry) {
	}
}
```

可以看到这个接口里面的方法是和`EngineExecutionListener`接口基本一致，就多出来了`testPlanExecutionStarted`和`testPlanExecutionFinished`方法。

在使用IDEA执行单元测试的时候，传入进来的实现类是`JUnit5TestExecutionListener`，它会处理相应的事件，并把结果在IDEA上呈现。

这儿需要注意的是Junit5应该是遵循了分层设计，`TestExecutionListener`和`EngineExecutionListener`接口功能几乎一致，但是它们却是在不同层次的测试执行事件观察者抽象，`TestExecutionListener`是在`Launcher`层面进行的抽象，层次更高，`EngineExecutionListener`是在测试执行引擎层面的抽象。

在Junit5中被观察的目标是`NodeTestTask`，它在执行测试动作的时候会通知观察者，比如`ref-4`处代码就是在通知观察者测试开始执行了。



## 四、@API注解

在贴出来的类层次图和一些类的源码中都有`@API`注解，我们接下来就探索一下这个注解。

该项目的地址为：https://github.com/apiguardian-team/apiguardian

这是一个非常小的项目，它的作用也是非常的明确，我们直接来看看它的类注释上的内容：

>@API is used to annotate public types, methods, constructors, and fields within a framework or application in order to publish their status and level of stability and to indicate how they are intended to be used by consumers of the API.
>
>If @API is present on a type, it is considered to hold for all public members of the type as well. However, a member of such an annotated type is allowed to declare a API.Status of lower stability. For example, a class annotated with @API(status = STABLE) may declare a constructor for internal usage that is annotated with @API(status = INTERNAL).

简单的说，这个注解就是用来标注接口和类的稳定性的，方便使用者明白类和接口的意图。

然后我们来看看Junit5中的一个实际例子：

```java
package org.junit.platform.launcher;

// 标识出这个接口是稳定的，可以被外部使用，status最后被修改时接口的版本为1.0
@API(status = STABLE, since = "1.0")  
public interface TestExecutionListener {
    ...... 省略
}
```



## 五、总结

写了三篇文章来分析Junit5，先是分析了Junit5单元测试执行的主干流程，然后分析了测试任务发现过程、参数解析过程，最后分析了执行任务的核心类、测试任务层次结构和涉及到的设计模式。基本上算是把Junit5分析透彻了，从中学习到了很多的知识，特别时接口的分层、异常处理技巧，还有过滤器模式和观察者模式的应用。



## 参考资料

1、[观察者模式](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html#id16)

