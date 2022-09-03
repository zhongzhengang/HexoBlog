---
title: Junit5设计分析（一）
categories:
    - Java
        - 测试
tags:
    - JUnit5
---

代码仓库：https://github.com/junit-team/junit5/



提到Java的单元测试，其实自己很熟悉了，从开始工作的时候就被要求写单元测试，所用到的框架就是`Junit`了，它最新的版本已经来到了5。

在平时写Java单元测试的时候，大致流程就是创建个测试类，在里面写个不要参数并且返回`void`的方法，然后在方法上面添加`@Test`注解。有了这三步，测试方法就能跑起来了，只不过啥用也没有。但是在测试方法里面写上调用业务的方法，然后再使用`assert`断言，就多增加了这两步，一个完整、有价值的单元测试就完成了。

在Junit5的帮助下，只需要上面这流畅的五步就可以写好单元测试了，反正我在平时根本不需要去注意Junit5的细节和其他知识就能完成单元测试的编写了。

现在仔细想象，这难道不是Junit5软件设计成功的地方吗？用户只需要知道必要的极少知识就能完成单元测试——Junit5软件的目标。

之前也听人说过Junit的代码写的非常好，再加之上面这些原因，我就来阅读一下Junit5的源码，希望有所收获。

<!-- more -->


## 一、IDEA中Junit5测试执行过程

首先上一张执行过程总览图：

![执行过程总览图](images/Junit5源码分析-测试执行流程.png)

先是由IDEA的测试插件启动测试过程，然后JUnit解析执行参数并发现测试方法，最后以Java反射的方式执行测试方法。步骤细节已经放在图片中了，下面对一些重点过程在详细解析一下。

## 二、任务对象NodeTestTask

在JUnit中将需要执行的任务抽成了`NodeTestTask`，从这个名字上可以看出来，其实这个类兼具了**两个方面的抽象**：

- 需要执行的任务。我认为就是`NodeTestTask`中的`TestTask`。
- 任务层次结构。JUnit将任务按照**测试引擎**、**测试类**、**测试方法**这三个层次进行了组织，形成了树形结构。`NodeTestTask`中的`Node`就是这个方面的抽象。

下面是这个类的字段信息：

```java
# 所在包：org.junit.platform.engine.support.hierarchical
class NodeTestTask<C extends EngineExecutionContext> implements TestTask {
	// 实际执行任务时需要用到的域对象
	private final NodeTestTaskContext taskContext;
    // 该接口描述了一个需要执行的任务，不限于测试方法。
	private final TestDescriptor testDescriptor;
    // 任务层次树中的节点
	private final Node<C> node;
	// 父节点的域对象
	private C parentContext;
    // 当前节点的域对象
	private C context;
	// 如果需要跳过测试执行方法，保存跳过的结果
	private SkipResult skipResult;
    // 标识是否开始
	private boolean started;
    // 异常收集器
	private ThrowableCollector throwableCollector;

	NodeTestTask(NodeTestTaskContext taskContext, TestDescriptor testDescriptor) {
		this.taskContext = taskContext;
		this.testDescriptor = testDescriptor;
		this.node = NodeUtils.asNode(testDescriptor);
	}
	...... 省略
}
```

在`NodeTestTask`字段中有两个重要的字段是`testDescriptor`和`node`，他们方便代表测试任务的描述和任务层次节点，这个在上面已经解释过了。然而JUnit5在这个两个变量上的处理有点儿巧妙，这两个字段都指向了同一个对象，这个对象实现了`TestDescriptor`和`Node`接口。显然**在`NodeTestTask`中是想把一个对象的这两个方面分开来对待**。

我觉得JUnit这么做的好处是，（1）可以将概念层次压平，（2）在使用时只暴露合适的功能，降低心智负担。下面就以`TestMethodTestDescriptor`类型示意一下。

先看一下该类的继承与实现关系：

<img src="/images/TestMethodTestDescriptor类继承关系.png" alt="image-20220818234140159" style="zoom: 67%;" />

下面我们来抽象一下`TestMethodTestDescriptor`类示例在`NodeTestTask`中被使用的情形：

![NodeTestTask中两个变量引用同一个对象的分析](images/NodeTestTask中两个变量引用同一个对象的分析.png)

在`NodeTestTask`当要使用测试描述方面的功能时，就会使用`TestDescriptor`类型`testDescriptor`成员变量，这样可以在实际使用时约束使用者确实想用的时测试描述方面的功能。同样的，在使用`Node`类型的`node`变量时也可以约束使用者确实想用的时任务层次节点的功能。

小结一下，我认为这个地方时JUnit5在降低复杂功能对象`TestMethodTestDescriptor`使用时的心智负担，并且使用Java类型系统来约束使用者的思考逻辑。

接下来我们在看看`NodeTestTask`中的方法细节：

```java
# 所在包：org.junit.platform.engine.support.hierarchical
class NodeTestTask<C extends EngineExecutionContext> implements TestTask {
	...... 省略
	@Override
	public void execute() { // 执行测试任务的主干流程
		throwableCollector = taskContext.getThrowableCollectorFactory().create();
        // 1、开始准备工作
		prepare();
        // 2、判断是否是跳过测试任务
		if (throwableCollector.isEmpty()) {
			checkWhetherSkipped();
		}
        // 3、递归执行测试任务和子任务
		if (throwableCollector.isEmpty() && !skipResult.isSkipped()) {
			executeRecursively();
		}
        // 4、执行清理动作
		if (context != null) {
			cleanUp();
		}
        // 5、通知监听器测试任务执行完成
		reportCompletion();
	}

	private void executeRecursively() {
        // 1、通知监听器测试任务开始执行
		taskContext.getListener().executionStarted(testDescriptor);
        // 2、设置开始标识为true
		started = true;
		// 3、将任务执行流程封装成Lambda表达式，提交到异常收集器中执行。
		throwableCollector.execute(() -> {
            // 3.1 获取子任务节点
			// @formatter:off
			List<NodeTestTask<C>> children = testDescriptor.getChildren().stream()
					.map(descriptor -> new NodeTestTask<C>(taskContext, descriptor))
					.collect(toCollection(ArrayList::new));
			// @formatter:on
			
            // 3.2 执行任务开始前的动作
			context = node.before(context); // ref-3
			// 3.3 开始执行测试任务
			DynamicTestExecutor dynamicTestExecutor = new DefaultDynamicTestExecutor();
			context = node.execute(context, dynamicTestExecutor); // ref-1
			// 3.4 调用子节点任务
			if (!children.isEmpty()) {
				children.forEach(child -> child.setParentContext(context));
				taskContext.getExecutorService().invokeAll(children); // ref-2
			}
			// 3.5 等待当前节点任务执行完成
			dynamicTestExecutor.awaitFinished();
		});
		// 4、将任务执行完后的动作封装为lambda表达式，提交到异常收集器中执行
		throwableCollector.execute(() -> node.after(context)); // ref-4
	}

	private void reportCompletion() {
        // 1、跳过测试任务的情况
		if (throwableCollector.isEmpty() && skipResult.isSkipped()) {
			taskContext.getListener().executionSkipped(testDescriptor, skipResult.getReason().orElse("<unknown>"));
			return;
		}
        // 2、还没有开始执行任务的情况
		if (!started) {
			// Call executionStarted first to comply with the contract of EngineExecutionListener.
			taskContext.getListener().executionStarted(testDescriptor);
		}
        // 3、向监听器告知测试任务执行完成，并把结果发送给监听器
		taskContext.getListener().executionFinished(testDescriptor, throwableCollector.toTestExecutionResult());
		throwableCollector = null;
	}
    ...... 省略其他方法
}
```

执行测试任务的主干流程在`execute(...)`方法中，执行当前节点任务和递归执行子节点任务的逻辑在`executeRecursively(...)`方法中，详细的执行步骤已经用注释的方式写在上面的代码中了。

有两个重要的点需要关注，`ref-1`处的代码是当前节点任务的执行，`ref-2`处是调用子节点任务的逻辑，我们接下来一个一个的来看。



## 三、当前节点任务的执行

其实在执行流程中，首先在`ref-1`处调用的节点任务是`JupiterEngineDescriptor`实例的`execute`方法，这个类的`execute`方法是一个空实现。由于递归执行的原因，然后在`ref-1`处调用的节点任务是`ClassTestDescriptor`实例的`execute`方法，这个类的`execute`方法也是一个空实现。最后在`ref-1`处调用的节点任务是`TestMethodTestDescriptor`实例，它的`execute`方法会执行我们写的单元测试方法。

下面我们先看下这三个`Node`实现类的层次关系：

<img src="/images/三个Node实现类继承层次关系.png" alt="image-20220819105042375" style="zoom: 50%;" />

可以看到这三个类既可以当作`Node`来看待，也可以当作`TestTask`来看待，所以他们才能被封装到`NodeTestTask`中。从概念上讲也是说的通的，就拿`TestMethodTestDescriptor`来说，它封装了执行测试方法的逻辑，所以可以看作是一个任务，同时它又在概念上属于测试类里面的范畴，所以也可以看作是任务层次树中的一个节点。

接下来我们具体看看单元测试方法的具体执行过程，也就是`TestMethodTestDescriptor`的`execute`方法：	

```java
package org.junit.jupiter.engine.descriptor;

@API(status = INTERNAL, since = "5.0")
public class TestMethodTestDescriptor extends MethodBasedTestDescriptor {
	// 执行任务的调用器
	private static final ExecutableInvoker executableInvoker = new ExecutableInvoker();
	...... 省略其他
    
    // 该方法会在NodeTestTask的execute(...)方法中被调用
	@Override
	public JupiterEngineExecutionContext prepare(JupiterEngineExecutionContext context) throws Exception {
		// 1、获取“扩展器”，相当于JUnit4的Rule
		ExtensionRegistry registry = populateNewExtensionRegistry(context);
        // 2、获取测试类的实例对象
		Object testInstance = context.getTestInstanceProvider().getTestInstance(Optional.of(registry));
		// 3、创建异常收集器
		ThrowableCollector throwableCollector = new OpenTest4JAwareThrowableCollector();
		// 4、创建扩展域对象
        ExtensionContext extensionContext = new MethodExtensionContext(context.getExtensionContext(),
			context.getExecutionListener(), this, context.getConfigurationParameters(), testInstance,
			throwableCollector);
		
        // 5、将“扩展器”、扩展域对象、异常收集器封装到域对象中
		// @formatter:off
		return context.extend()
				.withExtensionRegistry(registry)
				.withExtensionContext(extensionContext)
				.withThrowableCollector(throwableCollector)
				.build();
		// @formatter:on
	}

	@Override
	public JupiterEngineExecutionContext execute(JupiterEngineExecutionContext context,
			DynamicTestExecutor dynamicTestExecutor) throws Exception {
        // 1、获取异常收集器
		ThrowableCollector throwableCollector = context.getThrowableCollector();
		
        // 2、下面这段代码描述的就是我们测试方法的生命周期了
		// @formatter:off
		invokeBeforeEachCallbacks(context); //2.1 @RegisterExtention定义的每个测试方法**调用**前的动作
			if (throwableCollector.isEmpty()) {
				invokeBeforeEachMethods(context); // 2.2 执行@BeforeEach标注的方法
				if (throwableCollector.isEmpty()) {
                    // 2.3 @RegisterExtention定义的每个测试方法**执行**前的动作
					invokeBeforeTestExecutionCallbacks(context);
					if (throwableCollector.isEmpty()) {
						invokeTestMethod(context, dynamicTestExecutor); // 2.4 执行测试方法
					}
                    // 2.5 @RegisterExtention定义的每个测试方法**执行**后的动作
					invokeAfterTestExecutionCallbacks(context);
				}
				invokeAfterEachMethods(context); // 2.2 执行@AfterEach标注的方法
			}
		invokeAfterEachCallbacks(context); //2.1 @RegisterExtention定义的每个测试方法**调用**后的动作
		// @formatter:on
		
        // 3、判断是否有异常产生
		throwableCollector.assertEmpty();

		return context;
	}

	protected void invokeTestMethod(JupiterEngineExecutionContext context, DynamicTestExecutor dynamicTestExecutor) {
        // 1、获取扩展域对象
		ExtensionContext extensionContext = context.getExtensionContext();
        // 2、获取异常收集器
		ThrowableCollector throwableCollector = context.getThrowableCollector();
		
        // 3、在异常收集器中执行反射调用
		throwableCollector.execute(() -> {
			try {
                // 3.1、获取单元测试方法
				Method testMethod = getTestMethod();
                // 3.2 获取单元测试类实例
				Object instance = extensionContext.getRequiredTestInstance();
                // 3.3 使用调用器进行调用
				executableInvoker.invoke(testMethod, instance, extensionContext, context.getExtensionRegistry()); // ref-5
			}
			catch (Throwable throwable) {
				invokeTestExecutionExceptionHandlers(context.getExtensionRegistry(), extensionContext, throwable);
			}
		});
	}
	...... 省略其他方法
}
```

`prepare(...)`方法就是在准备执行测试方法需要用到的测试类实例、异常收集器和域对象。`execute(...)`方法封装的是执行单元测试方法的主干流程，从代码中可以看出JUnit5是`@BeforeEach`和`@AfterEach`标注的方法看作是单元测试方法的一部分，然后在其上叠加扩展器的方法。下面用一个图展示一下它们的关系。

![Junit5源码分析-测试方法生命周期](images/Junit5源码分析-测试方法生命周期.png)

`TestMethodTestDescriptor`描述的是一个单元测试方法，所以和测试方法强相关的生命周期动作就在该类的`execute(...)`方法中执行了。`ClassTestDescriptor`描述的是一个单元测试类，所以像`@BeforeAll`和`@AfterAll`之类的生命周期动作就在该类的`before(...)`和`after(...)`方法中执行了，这两个方法分别在`NodeTestTask`的`ref-3`和`ref-4`处执行。

在`invokeTestMethod(...)`方法的`ref-5`处会使用调用器来调用单元测试方法，具体方式就是使用Java的反射机制。下面我们看看具体代码：

```java
package org.junit.jupiter.engine.execution;

@API(status = INTERNAL, since = "5.0")
public class ExecutableInvoker {
    ...... 省略其他
	public Object invoke(Method method, Object target, ExtensionContext extensionContext,
			ExtensionRegistry extensionRegistry) {

		@SuppressWarnings("unchecked")
		Optional<Object> optionalTarget = (target instanceof Optional ? (Optional<Object>) target
				: Optional.ofNullable(target));
        // ref-6
		return ReflectionUtils.invokeMethod(method, target,
			resolveParameters(method, optionalTarget, extensionContext, extensionRegistry));
	}
    
	private Object[] resolveParameters(Executable executable, Optional<Object> target,
			ExtensionContext extensionContext, ExtensionRegistry extensionRegistry) {
		// ref-7
		return resolveParameters(executable, target, null, extensionContext, extensionRegistry);
	}
    ...... 省略其他
}
```

可以看到在`ExecutableInvoker`中做的**主要的工作就是解析并获取测试方法需要的参数**，然后就是使用反射工具调用单元测试方法。



## 四、子节点任务的递归

当前节点的任务执行完成了，就该递归执行子节点任务了，这个操作在`ref-2`处。我们先来回顾一下这个段代码：

```java
package org.junit.platform.engine.support.hierarchical
    
class NodeTestTask<C extends EngineExecutionContext> implements TestTask {
	private void executeRecursively() {
        ...... 省略
		throwableCollector.execute(() -> {
			...... 省略
			if (!children.isEmpty()) {
				children.forEach(child -> child.setParentContext(context));
				taskContext.getExecutorService().invokeAll(children); // ref-2
			}
            ...... 省略
		});
        ...... 省略
	}
    ...... 省略其他方法
}
```

在`ref-2`处其实是两个动作，首先`taskContext.getExecutorService()`获取执行器服务，具体获取到的是`SameThreadHierarchicalTestExecutorService`实例。第二个动作就是调用执行器服务的`invokeAll(...)`函数了。

`SameThreadHierarchicalTestExecutorService`的`invokeAll(...)`函数比较简单，下面我们看下具体的代码：

```java
package org.junit.platform.engine.support.hierarchical;

@API(status = EXPERIMENTAL, since = "1.3")
public class SameThreadHierarchicalTestExecutorService implements HierarchicalTestExecutorService {
	@Override
	public Future<Void> submit(TestTask testTask) {
		testTask.execute();
		return completedFuture(null);
	}

	@Override
	public void invokeAll(List<? extends TestTask> tasks) {
		tasks.forEach(TestTask::execute); 
	}

	@Override
	public void close() {
		// nothing to do
	}
}
```

在`invokeAll(...)`方法中直接使用`forEach`语法遍历子任务并调用`execute(...)`方法。

其实还有和ForkJoinPoolHierarchicalTestExecutorService类型执行器服务，可以并行执行单元测试。下面我们看下这两种类型执行器的继承层次结构：

<img src="/images/JUnit5源码分析-执行器服务类型继承层次.png" alt="image-20220819134925395" style="zoom:50%;" />

在`HierarchicalTestExecutorService`接口中定义了`submit(...)`、`invokeAll(...)`、`close`方法。



## 参考资料

1、idea插件源码：https://github.com/JetBrains/intellij-community



