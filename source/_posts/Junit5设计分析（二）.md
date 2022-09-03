---
title: Junit5设计分析（二）
categories:
    - Java
        - 测试
tags:
    - JUnit5
---

代码仓库：https://github.com/junit-team/junit5/



在**Junit5设计分析（一）**中分析了IDEA中执行单元测试的主流程和主要类的设计，但是还有两个小点没有分析到：

(1) TestEngine是怎么发现测试任务的？

(2) 单元测试方法的参数是怎么解析并获取的？

在这篇文章里面，就来探索这两个问题。


## 一、TestEngine是怎么发现测试任务的？

我们还是用"Junit5设计分析（一）"中的例子来进行探索。先来回顾一下这个示例程序吧。

```java
// 简单计算器类
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
}
```

```java
// 单元测试方法
class CalculatorTest {

    private final Calculator calculator = new Calculator();

    @Test
    void add() {
        assertEquals(2, calculator.add(1, 1));
    }
}
```

<!-- more -->

在IDEA中执行测试方法的主流程上，我们先看看测试任务在哪儿开始使用的。

```java
package org.junit.platform.engine.support.hierarchical;

class HierarchicalTestExecutor<C extends EngineExecutionContext> {
	private final ExecutionRequest request;
	private final C rootContext;
	private final HierarchicalTestExecutorService executorService;
	private final ThrowableCollector.Factory throwableCollectorFactory;

	HierarchicalTestExecutor(ExecutionRequest request, C rootContext, HierarchicalTestExecutorService executorService,
			ThrowableCollector.Factory throwableCollectorFactory) {
		this.request = request;
		this.rootContext = rootContext;
		this.executorService = executorService;
		this.throwableCollectorFactory = throwableCollectorFactory;
	}

	Future<Void> execute() {
        // 1、从request中获取根TestDescriptor
		TestDescriptor rootTestDescriptor = this.request.getRootTestDescriptor(); // ref-1
        // 2、从request中获取监听器
		EngineExecutionListener executionListener = this.request.getEngineExecutionListener();
        // 3、创建节点访问器，使用这个访问器执行任务的时候可以保护互斥资源
		NodeExecutionAdvisor executionAdvisor = new NodeTreeWalker().walk(rootTestDescriptor);
        // 4、创建NodeTestTask的域对象
		NodeTestTaskContext taskContext = new NodeTestTaskContext(executionListener, this.executorService, this.throwableCollectorFactory, executionAdvisor);
        // 5、创建NodeTestTask对象
		NodeTestTask<C> rootTestTask =new NodeTestTask<>(taskContext, rootTestDescriptor);//ref-2
		// 6、把当前域对象设置为rootTestTask的父域对象
        rootTestTask.setParentContext(this.rootContext);
        // 7、向执行服务中提交任务。
		return this.executorService.submit(rootTestTask); // ref-3
	}
}
```

`ref-1`处代码获取到的`TestDescriptor`是`JupiterEngineDescriptor`实例，它里面包含着测试类的描述和测试方法的描述。`ref-2`处代码会会将`rootTestDescriptor`封装到`NodeTestTask`中，然后在`ref-3`代码处提交到执行服务中执行，具体执行过程已经"Junit5设计分析（一）"中详细描述了。

从上面这段代码可以得到，测试任务一直被`request`携带着，那么我们就来看看这个`request`又是从哪儿来的？

```java
// org.junit.platform.launcher.core.DefaultLauncher#execute 方法
private void execute(Root root, ConfigurationParameters configurationParameters,
                     TestExecutionListener... listeners) {
    TestExecutionListenerRegistry listenerRegistry = buildListenerRegistryForExecution(listeners);
    withInterceptedStreams(configurationParameters, listenerRegistry, testExecutionListener -> {
        TestPlan testPlan = TestPlan.from(root.getEngineDescriptors());
        testExecutionListener.testPlanExecutionStarted(testPlan);
        ExecutionListenerAdapter engineExecutionListener = new ExecutionListenerAdapter(testPlan,
                                                                                        testExecutionListener);
        for (TestEngine testEngine : root.getTestEngines()) {
            TestDescriptor testDescriptor = root.getTestDescriptorFor(testEngine); // ref-5
            execute(testEngine, new ExecutionRequest(testDescriptor, engineExecutionListener, configurationParameters)); // ref-4
        }
        testExecutionListener.testPlanExecutionFinished(testPlan);
    });
}
```

我们在上文看到的request是在`ref-4`代码处创建出来的，`testDescriptor`是在`ref-5`处从`root`中获取到的。

那么接下来就是要看看`root`对象是从哪儿来的了？

```java
// org.junit.platform.launcher.core.DefaultLauncher#execute方法
@Override
public void execute(LauncherDiscoveryRequest discoveryRequest, TestExecutionListener... listeners) {
    Preconditions.notNull(discoveryRequest, "LauncherDiscoveryRequest must not be null");
    Preconditions.notNull(listeners, "TestExecutionListener array must not be null");
    Preconditions.containsNoNullElements(listeners, "individual listeners must not be null");
    execute(discoverRoot(discoveryRequest, "execution"), discoveryRequest.getConfigurationParameters(), listeners); // ref-6
}
```

可以看到`root`对象是在`ref-6`处调用`discoverRoot`得到的。我们来详细看看`discoverRoot`方法的细节。

```java
// org.junit.platform.launcher.core.DefaultLauncher#discoverRoot方法
private Root discoverRoot(LauncherDiscoveryRequest discoveryRequest, String phase) {
    // 1、创建Root对象
    Root root = new Root();
	// 2、从每一个TestEngine中发现TestDescriptor
    for (TestEngine testEngine : this.testEngines) {
        // @formatter:off
        boolean engineIsExcluded = discoveryRequest.getEngineFilters().stream()
            .map(engineFilter -> engineFilter.apply(testEngine))
            .anyMatch(FilterResult::excluded);
        // @formatter:on

        if (engineIsExcluded) {
            logger.debug(() -> String.format(
                "Test discovery for engine '%s' was skipped due to an EngineFilter in phase '%s'.",
                testEngine.getId(), phase));
            continue;
        }

        logger.debug(() -> String.format("Discovering tests during Launcher %s phase in engine '%s'.", phase,
                                         testEngine.getId()));
		// 2.1 发现TestDescriptor // ref-7
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

在`ref-7`处调用`discoverEngineRoot`发现`TestDescriptor`时，可以看到每次调用的时候都会将`discoveryRequest`传递下去，一会儿我们会分析到该对象的。我们接下来先看看`discoverEngineRoot`函数的细节：

```java
// org.junit.platform.launcher.core.DefaultLauncher#discoverEngineRoot方法
private Optional<TestDescriptor> discoverEngineRoot(TestEngine testEngine,
                                                    LauncherDiscoveryRequest discoveryRequest) {
	// 获取测试引擎第一无二的标识id
    UniqueId uniqueEngineId = UniqueId.forEngine(testEngine.getId());
    try {
        // ref-8
        TestDescriptor engineRoot = testEngine.discover(discoveryRequest, uniqueEngineId);
        discoveryResultValidator.validate(testEngine, engineRoot);
        return Optional.of(engineRoot);
    }
    catch (Throwable throwable) {
        handleThrowable(testEngine, "discover", throwable);
        return Optional.empty();
    }
}
```

在`ref-8`处会将`discoveryRequest`传递给`testEngine`进行测试任务的发现，那么我们测试任务的发现细节应该就是在`TestEngine#discover`方法里面了。我们接着看代码：

```java
// org.junit.jupiter.engine.JupiterTestEngine#discover 方法
@Override
public TestDescriptor discover(EngineDiscoveryRequest discoveryRequest, UniqueId uniqueId) {
   	// ref-9
    JupiterEngineDescriptor engineDescriptor = new JupiterEngineDescriptor(uniqueId);
    // ref-10
    new DiscoverySelectorResolver().resolveSelectors(discoveryRequest, engineDescriptor);
    return engineDescriptor;
}
```

在`ref-9`处会创建一个`TestDescriptor`的实现类`JupiterEngineDescriptor`的实例。在`ref-10`处会创建一个`DiscoverySelectorResolver`来从`discoveryRequest`发现测试任务，并把测试任务封装到`engineDescriptor`中。

接着继续跟`resolveSelectors`方法细节，如下所示：

```java
// org.junit.jupiter.engine.discovery.DiscoverySelectorResolver#resolveSelectors 方法
public void resolveSelectors(EngineDiscoveryRequest request, TestDescriptor engineDescriptor) {
    // 1、创建类型过滤器
    ClassFilter classFilter = buildClassFilter(request, isTestClassWithTests);
    // 2、开始解析测试任务
    resolve(request, engineDescriptor, classFilter); // ref-11
    // 3、用classFilter来过滤engineDescriptor中的测试任务
    filter(engineDescriptor, classFilter);
    // 4、将不包含测试方法的任务去除掉
    pruneTree(engineDescriptor);
}
```

现在的关键就是`ref-11`处代码底层时怎么解析测试任务的了。我们接着继续跟！

```java
// org.junit.jupiter.engine.discovery.DiscoverySelectorResolver#resolve 方法
private void resolve(EngineDiscoveryRequest request, TestDescriptor engineDescriptor, ClassFilter classFilter) {
// 1、创建Java元素解析器
    JavaElementsResolver javaElementsResolver = createJavaElementsResolver(request.getConfigurationParameters(), engineDescriptor, classFilter);
    request.getSelectorsByType(ClasspathRootSelector.class).forEach(javaElementsResolver::resolveClasspathRoot);
    request.getSelectorsByType(ModuleSelector.class).forEach(javaElementsResolver::resolveModule);
    request.getSelectorsByType(PackageSelector.class).forEach(javaElementsResolver::resolvePackage);
 
 request.getSelectorsByType(ClassSelector.class).forEach(javaElementsResolver::resolveClass);
 // ref-12   request.getSelectorsByType(MethodSelector.class).forEach(javaElementsResolver::resolveMethod);
    request.getSelectorsByType(UniqueIdSelector.class).forEach(javaElementsResolver::resolveUniqueId);
}
```

在`resolve`方法中首先创建了一个Java元素解析器，然后从`request`中获取所有可能的选择器，然后把获取到的选择器传递给元素解析器进行实际的解析。所有选择器的关系如下图所示：

![image-20220820230044639](images/选择器的层次关系.png)

首先得说一点，`DiscoverySelector`接口中没有任何得方法，是一个标识接口，在它的实现类中都有不同的方法，但是这些方法都是为了选择出测试任务，这个`DiscoverySelector`就是标识出所有不同实现类拥有着相同的目标。学到了啊，这就是标识接口的标准用法。

在我们的使用的单元测试例子中，只会进入到`ref-12`处的方法选择器中。我们详细来看看`resolveMethod`方法：

```java
# org.junit.jupiter.engine.discovery.JavaElementsResolver#resolveMethod 方法
void resolveMethod(MethodSelector selector) {
    try {
        // 1、从选择器中获取测试类信息
        Class<?> testClass = selector.getJavaClass();
        // 2、从选择器中获取测试方法信息
        Method testMethod = selector.getJavaMethod();
		// 3、解析testClass相关的测试任务（TestDescriptor），并且挂载到engineDescriptor的Children上。
        Set<TestDescriptor> potentialParents = resolveContainerWithParents(testClass); // ref-13
        // 3、解析testMethod相关的测试任务，并且挂载到相应的testClass的TestDescriptor的Children上。
        Set<TestDescriptor> resolvedDescriptors = resolveForAllParents(testMethod, potentialParents); // ref-14

        if (resolvedDescriptors.isEmpty()) {
            logger.debug(() -> format("Method '%s' could not be resolved.", testMethod.toGenericString()));
        }

        logMultipleTestDescriptorsForSingleElement(testMethod, resolvedDescriptors);
    }
    catch (Throwable t) {
        rethrowIfBlacklisted(t);
        logger.debug(t, () -> format("Method '%s' in class '%s' could not be resolved.", selector.getMethodName(), selector.getClassName()));
    }
}
```

我们已经跟到了测试类与测试方法的解析主流程了，`ref-13`处代码会从`testClass`上解析出来相关的`TestDescriptor`，`ref-14`处会从`testMethod`上解析出来相关的`TestDescriptor`。这两处代码应该就隐藏着解析测试任务的细节了，我们接着跟！

先看`ref-13`处的代码，如下所示：

```java
// org.junit.jupiter.engine.discovery.JavaElementsResolver#resolveContainerWithParents 方法
private Set<TestDescriptor> resolveContainerWithParents(Class<?> testClass) {
    if (isInnerClass.test(testClass)) { // 判断是否为内部类
        Set<TestDescriptor> potentialParents =                      					resolveContainerWithParents(testClass.getDeclaringClass()); // 获取外部类，进行递归调用
        return resolveForAllParents(testClass, potentialParents); 
    } else {
        // 为所有的父节点解析TestDescriptor          // ref-15
        return resolveForAllParents(testClass, Collections.singleton(this.engineDescriptor)); 
    }
}
```

不管是否为内部类，最终都会调用到`resolveForAllParents`方法，特别注意`ref-15`处，会将`this.engineDescriptor`作为父节点传入到`resolveForAllParents`方法。下面我们具体看一下该方法：

```java
private Set<TestDescriptor> resolveForAllParents(AnnotatedElement element, Set<TestDescriptor> potentialParents) {
   Set<TestDescriptor> resolvedDescriptors = new HashSet<>();
    // ref-16
   potentialParents.forEach(parent -> resolvedDescriptors.addAll(resolve(element, parent)));
   // @formatter:off
   resolvedDescriptors.stream()
         .filter(Filterable.class::isInstance)
         .map(Filterable.class::cast)  // 完成类型转换
         .forEach(testDescriptor -> testDescriptor.getDynamicDescendantFilter().allowAll());//过滤
   // @formatter:on
   return resolvedDescriptors;
}
```

`ref-16`处代码会调用`resolve`解析出来所有的`TestDescriptor`，并且将他们添加到`resolvedDescriptors`中，以便后续做类型转换和过滤。那么重点来了，就是这个`resolve`方法的细节了，我们接着跟！

```java
// org.junit.jupiter.engine.discovery.JavaElementsResolver#resolve 方法
private Set<TestDescriptor> resolve(AnnotatedElement element, TestDescriptor parent) {
    // 使用解析器尝试解析element对象相关的测试任务（TestDescriptor）
    Set<TestDescriptor> descriptors = this.resolvers.stream()
        .map(resolver -> tryToResolveWithResolver(element, parent, resolver)) // ref-17
        .filter(testDescriptors -> !testDescriptors.isEmpty())
        .flatMap(Collection::stream)
        .collect(toSet());

    logMultipleTestDescriptorsForSingleElement(element, descriptors);

    return descriptors;
}
```

`resolve`方法使用所有解析器来尝试解析`element`元素相关的测试任务`TestDescriptor`，我们先来看看解析器都有哪些：

![image-20220823201401694](images/所有类型的解析器.png)

这些解析器都会在`ref-17`处的`tryToResolveWithResolver`方法中被调用，让我们来看看详细代码：

```java
// org.junit.jupiter.engine.discovery.JavaElementsResolver#tryToResolveWithResolver 方法
private Set<TestDescriptor> tryToResolveWithResolver(AnnotatedElement element, TestDescriptor parent, ElementResolver resolver) {
	// 1、调用解析器解析TestDescriptor
    Set<TestDescriptor> resolvedDescriptors = resolver.resolveElement(element, parent); // ref-18
    Set<TestDescriptor> result = new LinkedHashSet<>();
	// 2、将解析出来的TestDescriptor挂载到parent上，并且添加到结果集合result中。
    resolvedDescriptors.forEach(testDescriptor -> {
        // 3、在当前的this.engineDescriptor中查找是否已经存在该testDescriptor
        Optional<TestDescriptor> existingTestDescriptor = findTestDescriptorByUniqueId(
            testDescriptor.getUniqueId());
        if (existingTestDescriptor.isPresent()) { // 如果存在，就只添加到结果集合result中
            result.add(existingTestDescriptor.get());
        }
        else { // 如果不存在，那么先将testDescriptor挂载到parent上，再添加到结果集合result中
            parent.addChild(testDescriptor);
            result.add(testDescriptor);
        }
    });

    return result;
}
```

到这儿的重点就是`ref-18`处代码的`resolveElement`方法了。通过debug发现`ref-13`处代码调用到`ref-18`处时，解析器为`TestContainerResolver`，`ref-14`处代码调用到`ref-18`处时，解析器为`TestMethodResolver`。

我们先来看看解析测试类的`TestContainerResolver`:

```java
package org.junit.jupiter.engine.discovery;

class TestContainerResolver implements ElementResolver {

	private static final IsPotentialTestContainer isPotentialTestContainer = new IsPotentialTestContainer();

	static final String SEGMENT_TYPE = "class";

	protected final ConfigurationParameters configurationParameters;

	public TestContainerResolver(ConfigurationParameters configurationParameters) {
		this.configurationParameters = configurationParameters;
	}

	@Override
	public Set<TestDescriptor> resolveElement(AnnotatedElement element, TestDescriptor parent) {
		// 1、判断是否为class类型的对象
        if (!(element instanceof Class)) {
			return Collections.emptySet();
		}
        // 2、判断是否为潜在的候选者，也就是判断是否为有效的测试类
		Class<?> clazz = (Class<?>) element;
		if (!isPotentialCandidate(clazz)) {
			return Collections.emptySet();
		}
		// 3、创建唯一标识
		UniqueId uniqueId = createUniqueId(clazz, parent);
        // 4、创建ClassTestDescriptor对象，并封装到集合中，然后返回
		return Collections.singleton(resolveClass(clazz, uniqueId));
	}

	protected boolean isPotentialCandidate(Class<?> element) {
		return isPotentialTestContainer.test(element);
	}

	protected UniqueId createUniqueId(Class<?> testClass, TestDescriptor parent) {
		return parent.getUniqueId().append(getSegmentType(), getSegmentValue(testClass));
	}

	protected TestDescriptor resolveClass(Class<?> testClass, UniqueId uniqueId) {
		return new ClassTestDescriptor(uniqueId, testClass, this.configurationParameters);
	}
	...... 省略其他方法
}
```

然后，我们再看看解析测试方法的`TestMethodResolver`:

```java
package org.junit.jupiter.engine.discovery;

class TestMethodResolver extends AbstractMethodResolver {

	private static final Predicate<Method> isTestMethod = new IsTestMethod();

	static final String SEGMENT_TYPE = "method";

	TestMethodResolver() {
		super(SEGMENT_TYPE, isTestMethod);
	}

	@Override // ref-19
	protected TestDescriptor createTestDescriptor(UniqueId uniqueId, Class<?> testClass, Method method) {
		return new TestMethodTestDescriptor(uniqueId, testClass, method);
	}

}
```

在`TestMethodResolver`中没有看见`resolveElement`方法，那么应该就在它的父类里面了，我们来详细看看：

```java
package org.junit.jupiter.engine.discovery;

abstract class AbstractMethodResolver implements ElementResolver {

	private static final MethodFinder methodFinder = new MethodFinder();

	private final String segmentType;
	private final Predicate<Method> methodPredicate;

	AbstractMethodResolver(String segmentType, Predicate<Method> methodPredicate) {
		this.segmentType = segmentType;
		this.methodPredicate = methodPredicate;
	}

	@Override
	public Set<TestDescriptor> resolveElement(AnnotatedElement element, TestDescriptor parent) {
		// 1、判断element是否为方法
        if (!(element instanceof Method)) {
			return Collections.emptySet();
		}
		// 2、判断方法所在的类是否为测试类
		if (!(parent instanceof ClassTestDescriptor)) {
			return Collections.emptySet();
		}
		// 3、判断方法是否为有效的测试方法
		Method method = (Method) element;
		if (!isRelevantMethod(method)) {
			return Collections.emptySet();
		}
		// 4、创建TestMethodTestDescriptor实例对象
		return Collections.singleton(createTestDescriptor(parent, method));
	}

	private boolean isRelevantMethod(Method candidate) {
		return methodPredicate.test(candidate);
	}

	private UniqueId createUniqueId(Method method, TestDescriptor parent) {
		String methodId = String.format("%s(%s)", method.getName(),
			ClassUtils.nullSafeToString(method.getParameterTypes()));
		return parent.getUniqueId().append(this.segmentType, methodId);
	}

	private TestDescriptor createTestDescriptor(TestDescriptor parent, Method method) {
        // 1、创建唯一标识
		UniqueId uniqueId = createUniqueId(method, parent);
        // 2、获取测试方法对应的测试类
		Class<?> testClass = ((ClassTestDescriptor) parent).getTestClass();
        // 3、调用子类的createTestDescriptor方法，创建实际的TestDescriptor实例。
		return createTestDescriptor(uniqueId, testClass, method);
	}

	protected abstract TestDescriptor createTestDescriptor(UniqueId uniqueId, Class<?> testClass, Method method);
	...... 省略其他方法
}
```

至此，我们已经了解JUint5发现测试任务的详细过程，那么现在还有一个疑问需要探索，那就是支撑整个测试任务发现过程的`MethodSelector`是哪儿来的呢？

初步来看，这个`request`是来自于`ref-12`处的`request`，`request`又是在`DefaultLauncher#execute`方法传入的，那么可以推断出`LauncherDiscoveryRequest`类型的实例`request`是从IDEA的测试插件传入的。让我们详细来看看：

```java
// github仓库地址：https://github.com/JetBrains/intellij-community
// plugins/junit5_rt/src/com/intellij/junit5/JUnit5IdeaTestRunner.java 文件
@Override
public int startRunnerWithArgs(String[] args, String name, int count, boolean sendTree) {
    try {
        JUnit5TestExecutionListener listener = myExecutionListeners.get(0);
        listener.initializeIdSuffix(!sendTree);
        final String[] packageNameRef = new String[1]; // ref-20
        final LauncherDiscoveryRequest discoveryRequest = JUnit5TestRunnerUtil.buildRequest(args, packageNameRef); // ref-21
		...... 省略
        myLauncher.execute(discoveryRequest, listeners.toArray(new TestExecutionListener[0]));
        return listener.wasSuccessful() ? 0 : -1;
    } catch (Exception e) {
        System.err.println("Internal Error occurred.");
        e.printStackTrace(System.err);
        return -2;
    } finally {
        if (count > 0) myExecutionListeners.remove(0);
    }
}
```

上文提到的`request`就是在`ref-21`处创建出来的，它的入参`packageNameRef`是长度为1的空白字符串数组，`args`是的值为`["com.mucao.CalculatorTest,add"]`。

接下来我们看一下创建`MethodSelector`的方法：

```java
// github仓库地址：https://github.com/JetBrains/intellij-community
// plugins/junit5_rt/src/com/intellij/junit5/JUnit5TestRunnerUtil.java 文件
public static LauncherDiscoveryRequest buildRequest(String[] suiteClassNames, String[] packageNameRef) {
    if (suiteClassNames.length == 0) {
        return null;
    }
    // 1、创建builder对象
    LauncherDiscoveryRequestBuilder builder = LauncherDiscoveryRequestBuilder.request();

    if (suiteClassNames.length == 1 && suiteClassNames[0].charAt(0) == '@') {
        ...... 省略
    }
    else {
        // 2、创建selector对象
        DiscoverySelector selector = createSelector(suiteClassNames[0], packageNameRef); //ref-22
        if (selector instanceof MethodSelector && !loadMethodByReflection((MethodSelector)selector)) {
            DiscoverySelector classSelector = createClassSelector(((MethodSelector)selector).getClassName());
            DiscoverySelector methodSelector = classSelector instanceof NestedClassSelector
                ? DiscoverySelectors.selectMethod(((NestedClassSelector)classSelector).getNestedClassName(),
                                                  ((MethodSelector)selector).getMethodName())
                : selector;
            builder.filters(createMethodFilter(Collections.singletonList(methodSelector)));
            selector = classSelector;
        }
        assert selector != null : "selector by class name is never null";
        // 3、封装selector对象到builder中，然后构建LauncherDiscoveryRequest
        return builder.selectors(selector).build(); // ref-23
    }

    return null;
}
```

在`ref-22`处创建了`DiscoverySelector`了，然后经过一系列的判断和处理，然后在`ref-23`处设置到`builder`中，最后构建出来一个`LauncherDiscoveryRequest`对象返回。由于IDEA的debug没有办法挂载到这段属于plugin的代码，所以没有办法跟踪细节，但是大体流程是对着的。

我们在看看`createSelector`方法的细节：

```java
// github仓库地址：https://github.com/JetBrains/intellij-community
// plugins/junit5_rt/src/com/intellij/junit5/JUnit5TestRunnerUtil.java文件
  protected static DiscoverySelector createSelector(String line, String[] packageNameRef) {
    if (line.startsWith("\u001B")) {
      String uniqueId = line.substring("\u001B".length());
      return DiscoverySelectors.selectUniqueId(uniqueId);
    }
    else if (line.startsWith("\u002B")) {
      String directory = line.substring("\u002B".length());
      List<ClasspathRootSelector> selectors = DiscoverySelectors.selectClasspathRoots(Collections.singleton(Paths.get(directory)));
      if (selectors.isEmpty()) {
        return null;
      } else {
        return selectors.iterator().next();
      }
    }
    else if (line.contains(",")) { // ref-23
      MethodSelector selector = DiscoverySelectors.selectMethod(line.replaceFirst(",", "#"));
      if (packageNameRef != null) {
        packageNameRef[0] = selector.getClassName();
      }
      return selector;
    }
    else {
      if (packageNameRef != null) {
        packageNameRef[0] = line;
      }

      return createClassSelector(line);
    }
  }
```

由于`args`参数的值，可以判断具体会从`ref-23`处创建`MethodSelector`，该处调用的代码已经是JUnit5的代码了，我们来详细看看：

```java
// org.junit.platform.engine.discovery.DiscoverySelectors.java 文件
@API(status = STABLE, since = "1.0")
public final class DiscoverySelectors {
    public static MethodSelector selectMethod(String fullyQualifiedMethodName) throws PreconditionViolationException {
        // 1、将className, methodName, methodParameters解析出来
        String[] methodParts = ReflectionUtils.parseFullyQualifiedMethodName(fullyQualifiedMethodName);
        // 2、创建MethodSelector
        return selectMethod(methodParts[0], methodParts[1], methodParts[2]);
    }

    public static MethodSelector selectMethod(String className, String methodName, String methodParameterTypes) {
        Preconditions.notBlank(className, "Class name must not be null or blank");
        Preconditions.notBlank(methodName, "Method name must not be null or blank");
        Preconditions.notNull(methodParameterTypes, "Parameter types must not be null");
        return new MethodSelector(className, methodName, methodParameterTypes.trim());
    }
    ...... 省略其他方法
}
```

跟到这儿，我们已经清楚了`MethodSelector` 的创建和测试任务`TestDescriptor`的发现，下面由一张图来小总结一下：

![Junit5源码分析-测试任务发现流程](images/Junit5源码分析-测试任务发现流程.png)

在这个图里面重点注意三个：`LauncherDiscoveryRequest`、`EngineDiscoveryRequest`、`ExecutionRequest`，这三个请求分别对应JUnit5执行测试任务的阶段：启动阶段、引擎发现阶段、任务执行阶段。申明一下，没有在官网找到相关文档，这只是个人的理解。而且，从这儿看出来**JUnit5是把自己当作一个独立运行的服务**，而是被第三方嵌入的SDK，别人向它提交的都是一个个的请求。



## 二、单元测试方法的参数是怎么解析并获取的？

我们先来看看JUnit5中实际调用测试方法的地方：

```java
// org.junit.jupiter.engine.execution.ExecutableInvoker#invoke 方法
/**
* Invoke the supplied method on the supplied target object with dynamic parameter
* resolution.
*
* @param method the method to invoke and resolve parameters for
* @param target the object on which the method will be invoked; should be
* {@code null} for static methods
* @param extensionContext the current {@code ExtensionContext}
* @param extensionRegistry the {@code ExtensionRegistry} to retrieve
* {@code ParameterResolvers} from
*/
public Object invoke(Method method, Object target, ExtensionContext extensionContext,
                     ExtensionRegistry extensionRegistry) {

    @SuppressWarnings("unchecked")
    Optional<Object> optionalTarget = (target instanceof Optional ? (Optional<Object>) target
                                       : Optional.ofNullable(target));
    return ReflectionUtils.invokeMethod(method, target,
    	resolveParameters(method, optionalTarget, extensionContext, extensionRegistry)); //ref-24
}
```

`ref-24`处的代码就是解析测试方法参数的地方，我们先看看JUnit5是怎么解析出来参数的？

```java
// org.junit.jupiter.engine.execution.ExecutableInvoker#resolveParameters 方法
private Object[] resolveParameters(Executable executable, Optional<Object> target,
                ExtensionContext extensionContext, ExtensionRegistry extensionRegistry) {
    return resolveParameters(executable, target, null, extensionContext, extensionRegistry);
}
```
```java
// org.junit.jupiter.engine.execution.ExecutableInvoker#resolveParameters 方法
private Object[] resolveParameters(Executable executable, Optional<Object> target, Object outerInstance, ExtensionContext extensionContext, ExtensionRegistry extensionRegistry) {

    Preconditions.notNull(target, "target must not be null");

    Parameter[] parameters = executable.getParameters();
    Object[] values = new Object[parameters.length];
    int start = 0;

    // Ensure that the outer instance is resolved as the first parameter if
    // the executable is a constructor for an inner class.
    if (outerInstance != null) {
        values[0] = outerInstance;
        start = 1;
    }

    // Resolve remaining parameters dynamically
    for (int i = start; i < parameters.length; i++) {
        ParameterContext parameterContext = new DefaultParameterContext(parameters[i], i, target);
        values[i] = resolveParameter(parameterContext, executable, extensionContext, extensionRegistry); // ref-25
    }
    return values;
}
```

上面贴出来的这两个方法，都是在做解析参数前的准备工作，会把方法的形式参数信息拿到，然后去一个一个解析具体参数值，`ref-25`处代码就是在解析单个具体参数的值。我们接着看：

```java
// org.junit.jupiter.engine.execution.ExecutableInvoker#resolveParameter 方法
private Object resolveParameter(ParameterContext parameterContext, Executable executable,
                ExtensionContext extensionContext, ExtensionRegistry extensionRegistry) {
    try {
        // 1、获取extensionRegistry中所有的参数解析器
        // @formatter:off
        List<ParameterResolver> matchingResolvers = extensionRegistry.stream(ParameterResolver.class)
            .filter(resolver -> resolver.supportsParameter(parameterContext, extensionContext))
            .collect(toList());
        // @formatter:on

        if (matchingResolvers.isEmpty()) {
            ...... 抛出异常
        }

        if (matchingResolvers.size() > 1) {
            ...... 抛出异常
        }
        // 2、获取唯一的参数解析器
        ParameterResolver resolver = matchingResolvers.get(0);
        // 3、参数解析器开始解析参数
        Object value = resolver.resolveParameter(parameterContext, extensionContext); // ref-26
        // 4、对解析出来的参数验证类型
        validateResolvedType(parameterContext.getParameter(), value, executable, resolver);
        ...... 记录日志
        return value;
    }
    catch (ParameterResolutionException ex) {
        ...... 处理异常
    }
}
```

在`ref-26`处调用唯一的那个参数解析器开始解析具体参数值，`resolver#resolveParameter`就封装了不同种类参数的具体解析逻辑。

我们看看都有哪些参数解析器实现？

![image-20220826121941564](images/参数解析器实现类.png)

- `RepetitionInfo`是解析重复测试的参数，也就是标注了`@RepeatedTest`的测试方法会用到这个参数解析器。
- `TestReporterParameterResolver`在解析参数的时候，会向执行监听器发送事件，并不会解析参数。
- `TestInfoParameterResolver`用于将测试信息转入到测试方法中，和`@TestInfo`注解有关。
- `ParameterizedTestParameterResolver`在参数化测试的时候会注入参数到测试方法，和`@ParameterizedTest`有关。

现在还有一个疑问，这些参数解析器从哪儿来的呢？首先我们看看`extensionRegistry`是从哪儿来的？

```java
// org.junit.jupiter.engine.descriptor.JupiterEngineDescriptor#prepare 方法
@Override
public JupiterEngineExecutionContext prepare(JupiterEngineExecutionContext context) {
    ExtensionRegistry extensionRegistry = createRegistryWithDefaultExtensions(context.getConfigurationParameters()); // ref-27
    EngineExecutionListener executionListener = context.getExecutionListener();
    ExtensionContext extensionContext = new JupiterEngineExtensionContext(executionListener, this,
                                                                          context.getConfigurationParameters());

    // @formatter:off
    return context.extend()
        .withExtensionRegistry(extensionRegistry)
        .withExtensionContext(extensionContext)
        .build();
    // @formatter:on
}
```

`ExtensionRegistry`是在`ref-27`处创建出来的，我们接着看看创建的具体逻辑？

```java
@API(status = INTERNAL, since = "5.0")
public class ExtensionRegistry {
    private static final List<Extension> DEFAULT_EXTENSIONS = Collections.unmodifiableList(Arrays.asList(//
        new DisabledCondition(), //
        new ScriptExecutionCondition(), //
        new RepeatedTestExtension(), //
        new TestInfoParameterResolver(), // ref-28
        new TestReporterParameterResolver())); // ref-29

    public static ExtensionRegistry createRegistryWithDefaultExtensions(ConfigurationParameters configParams) {
        ExtensionRegistry extensionRegistry = new ExtensionRegistry(null);
		...... 打印日志
        // 1、把默认的扩展器注入到extensionRegistry
        DEFAULT_EXTENSIONS.forEach(extensionRegistry::registerDefaultExtension);
        
        // 2、把自动探测到的扩展器注入到extensionRegistry
        if (configParams.getBoolean(EXTENSIONS_AUTODETECTION_ENABLED_PROPERTY_NAME).orElse(Boolean.FALSE)) {
            registerAutoDetectedExtensions(extensionRegistry);
        }
        return extensionRegistry;
    }

    private static void registerAutoDetectedExtensions(ExtensionRegistry extensionRegistry) {
        // 1、使用SPI加载外部的的扩展器
        Iterable<Extension> extensions = ServiceLoader.load(Extension.class, ClassLoaderUtils.getDefaultClassLoader());
		...... 打印日志
        // 2、将加载到的扩展器注入到extensionRegistry
        extensions.forEach(extensionRegistry::registerDefaultExtension);
    }
	...... 省略其他方法
}
```

原来我们的参数解析器是以扩展器`Extension`的形式注入到`ExtensionRegistry`，`ref-28`和`ref-29`处就是`TestInfoParameterResolver`和`TestReporterParameterResolver`参数解析器。

我们最后在看下`extensionRegistry::registerDefaultExtension`方法：

```java
// org.junit.jupiter.engine.extension.ExtensionRegistry#registerDefaultExtension 方法
private void registerDefaultExtension(Extension extension) {
    // 1、添加扩展器
    this.registeredExtensions.add(extension);
    // 2、增加扩展器的类型
    this.registeredExtensionTypes.add(extension.getClass());
}
```

小总结，参数解析器就是以`Extention`的形式添加到`RegistryExtention`中，然后在执行测试方法之前，就可以拿出来进行调用，解析处需要的参数，传递给测试方法，就可以执行了。


