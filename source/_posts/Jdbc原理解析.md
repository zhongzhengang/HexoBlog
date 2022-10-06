---
title: Jdbc原理解析
categories:
    - Java
        - 数据库
tags:
    - Jdbc
---


由于Jdbc的存在，Java程序员不用关注各个数据库的连接细节，使用统一的接口与数据库交互。Jdbc为我们提供了这么大的便利，它是怎么达成的呢？这篇文章就是探索这个问题的答案。



## 一、完整的Jdbc示例程序

文章后续的分析都是基于下面的示例程序：

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

<!-- more -->

使用的MySQL驱动版本和maven依赖坐标为：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
</dependency>
```

MySQL版本为： 8.0.30 MySQL Community Server - GPL


## 二、执行流程分析



### 1、主流程分析

- step1 调用DriverManager获取连接Connection。

  ```java
  Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
  ```

  

  - step1.1 通过属性jdbc.drivers或者ServiceLoader加载数据库驱动类

    ```java
    // java.sql.DriverManager.java文件
    private static void ensureDriversInitialized() {
        if (driversInitialized) {
            return;
        }
    
        synchronized (lockForInitDrivers) {
            if (driversInitialized) {
                return;
            }
    		...... 省略
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
    
                    ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                    Iterator<Driver> driversIterator = loadedDrivers.iterator();
                    try {
                        while (driversIterator.hasNext()) {
                            driversIterator.next();
                        }
                    } catch (Throwable t) {
                        // Do nothing
                    }
                    return null;
                }
            });
    		...... 省略
        }
    }
    
    ```

  - step1.2 <span id="driverInitialize">数据库驱动类完成初始化</span>

    ```java
    // com.mysql.cj.jdbc.Driver.java文件
    // 这个类的父类会做很多的初始化操作。
    public class Driver extends NonRegisteringDriver implements java.sql.Driver {
        //
        // Register ourselves with the DriverManager
        //
        static {
            try {
                java.sql.DriverManager.registerDriver(new Driver()); // ref-1
            } catch (SQLException E) {
                throw new RuntimeException("Can't register driver!");
            }
        }
        
        public Driver() throws SQLException {
            // Required for Class.forName().newInstance()
        }
    }
    ```

    

  - step1.3 通过驱动类实例<span id="getConnection">获取连接Connection</span>

    ```java
    // java.sql.DriverManager.java文件
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
        ...... 省略
        ensureDriversInitialized();
    
        // Walk through the loaded registeredDrivers attempting to make a connection.
        // Remember the first exception that gets raised so we can reraise it.
        SQLException reason = null;
    
        for (DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            if (isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info); // ref-2
                    if (con != null) {
                        // Success!
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }
    
            } else {
                println("    skipping: " + aDriver.driver.getClass().getName());
            }
    
        }
    
        // if we got here nobody could connect.
        if (reason != null)    {
            println("getConnection failed: " + reason);
            throw reason;
        }
    
        println("getConnection: no suitable driver found for "+ url);
        throw new SQLException("No suitable driver found for "+ url, "08001");
    }
    ```

    

- step2 调用Connection创建Statement。

  - step1.1 获取在驱动程序内部创建的连接实例JdbcConnection。

    ```java
    // com.mysql.cj.jdbc.ConnectionImpl.java文件
    @Override
    public JdbcConnection getMultiHostSafeProxy() {
        return this.getProxy();
    }
    private JdbcConnection getProxy() {
        return (this.topProxy != null) ? this.topProxy : (JdbcConnection) this;
    }
    ```

    

  - step1.2 创建Statement接口实现类StatementImpl的实例。

    ```java
    // com.mysql.cj.jdbc.ConnectionImpl.java文件
    @Override
    public java.sql.Statement createStatement(int resultSetType, int resultSetConcurrency) throws SQLException {
        StatementImpl stmt = new StatementImpl(getMultiHostSafeProxy(), this.database);
        stmt.setResultSetType(resultSetType);
        stmt.setResultSetConcurrency(resultSetConcurrency);
        return stmt;
    }
    ```

    

- step3 使用Statement执行SQL语句。

  - step3.1 通过驱动程序内部连接JdbcConnection获取会话（NativeSession）。

    ```java
    // com.mysql.cj.jdbc.StatementImpl.java文件
    @Override
    public java.sql.ResultSet executeQuery(String sql) throws SQLException {
        synchronized (checkClosed().getConnectionMutex()) {
            JdbcConnection locallyScopedConn = this.connection;
    		......省略
            try {
    			......省略
                statementBegins();
                this.results = ((NativeSession) locallyScopedConn.getSession()).execSQL(this, sql, this.maxRows, null, createStreamingResultSet(),getResultSetFactory(),cachedMetaData, false);                                   
                if (timeoutTask != null) {
                    stopQueryTimer(timeoutTask, true, true);
                    timeoutTask = null;
                }
            } catch (CJTimeoutException | OperationCancelledException e) {
                throw SQLExceptionsMapping.translateException(e, this.exceptionInterceptor);
            } finally {
    			......省略
            }
            this.lastInsertId = this.results.getUpdateID();
    		......省略
            return this.results;
        }
    }
    ```

    

  - step3.2 私有协议NativeProtocol由SQL语句字符串构建报文，然后发送给数据库服务器端。

    ```java
    // com.mysql.cj.NativeSession.java文件
    public <T extends Resultset> T execSQL(Query callingQuery, String query, int maxRows, NativePacketPayload packet, boolean streamResults,
                                           ProtocolEntityFactory<T, NativePacketPayload> resultSetFactory, ColumnDefinition cachedMetadata, boolean isBatch) {
    	......省略
        try {
            return packet == null
                ? ((NativeProtocol) this.protocol).sendQueryString(callingQuery, query, this.characterEncoding.getValue(), maxRows, streamResults,cachedMetadata, resultSetFactory)
                : ((NativeProtocol) this.protocol).sendQueryPacket(callingQuery, packet, maxRows, streamResults, cachedMetadata, resultSetFactory);
            
        } catch (CJException sqlE) {
      	......省略
        } catch (Throwable ex) {
        ......省略
        } finally {
        ......省略
        }
    }
    ```

    

- step4 处理SQL语句执行结果。

  ```java
  // 示例程序中的代码
  ResultSet resultSet = statement.executeQuery("SELECT * FROM students");
  //如果有数据，resultSet.next()返回true
  while (resultSet.next()) {
      System.out.println(" 名称: " + resultSet.getString("name") +
                         " 年龄: " + resultSet.getInt("age") +
                         " 年级: " + resultSet.getString("grade") );
  }
  ```

  

### 2、MySQL驱动程序初始化

在Jdbc的驱动管理类中，通过ServiceLoader加载到MySQL驱动类后，只是调用到了迭代器的`next()`方法，其他的啥也没有干。其实`next()`方法会导致驱动类`com.mysql.cj.jdbc.Driver`被加载，然后就会执行到其中的静态初始代码块。

首先执行的是`Driver`类的父类`NonRegisteringDriver`中的静态初始化代码，如下所示：

```java
// com.mysql.cj.jdbc.NonRegisteringDriver.java文件
public class NonRegisteringDriver implements java.sql.Driver {
    ...... 省略其他方法
    static {
        try {
            Class.forName(AbandonedConnectionCleanupThread.class.getName()); // ref-3
        } catch (ClassNotFoundException e) {
            // ignore
        }
    }
}
```

ref-3处代码就干了一件事，加载类`AbandonedConnectionCleanupThread`，<span id="cleanThreadInit">然后会触发其静态代码块被执行：</span>

```java
// com.mysql.cj.jdbc.AbandonedConnectionCleanupThread.java文件
/**
 * This class implements a thread that is responsible for closing abandoned MySQL connections, 
 * i.e., connections that are not explicitly closed.
 * There is only one instance of this class and there is a single thread to do this task. This 
 * thread's executor is statically referenced in this same class.
 */
public class AbandonedConnectionCleanupThread implements Runnable {
    private static final Set<ConnectionFinalizerPhantomReference> connectionFinalizerPhantomRefs = ConcurrentHashMap.newKeySet();
    // ref-4
    private static final ReferenceQueue<MysqlConnection> referenceQueue = new ReferenceQueue<>();
    private static final ExecutorService cleanupThreadExecutorService;
    private static Thread threadRef = null;
    private static Lock threadRefLock = new ReentrantLock();
    private static boolean abandonedConnectionCleanupDisabled = Boolean.getBoolean(PropertyDefinitions.SYSP_disableAbandonedConnectionCleanup);

    static {
        if (abandonedConnectionCleanupDisabled) {
            cleanupThreadExecutorService = null;
        } else {
            cleanupThreadExecutorService = Executors.newSingleThreadExecutor(r -> {
                Thread t = new Thread(r, "mysql-cj-abandoned-connection-cleanup");
                t.setDaemon(true);
                // Tie the thread's context ClassLoader to the ClassLoader that loaded the class                 // instead of inheriting the context ClassLoader from the current
                // thread, which would happen by default.
                // Application servers may use this information if they attempt to shutdown this                 // thread. By leaving the default context ClassLoader this thread
                // could end up being shut down even when it is shared by other applications and,                 // being it statically initialized, thus, never restarted again.

                ClassLoader classLoader = AbandonedConnectionCleanupThread.class.getClassLoader();
                if (classLoader == null) {
                    // This class was loaded by the Bootstrap ClassLoader, so let's tie the thread's context ClassLoader to the System ClassLoader instead.
                    classLoader = ClassLoader.getSystemClassLoader();
                }

                t.setContextClassLoader(classLoader);
                return threadRef = t;
            });
            cleanupThreadExecutorService.execute(new AbandonedConnectionCleanupThread());
        }
    }

    private AbandonedConnectionCleanupThread() {
    }

    public void run() {
        for (;;) {
            try {
                checkThreadContextClassLoader();
                Reference<? extends MysqlConnection> reference = referenceQueue.remove(5000);
                if (reference != null) {
                    finalizeResource((ConnectionFinalizerPhantomReference) reference);
                }
            } catch (InterruptedException e) {
                threadRefLock.lock();
                try {
                    threadRef = null;
                    
                    // Finalize remaining references.
                    Reference<? extends MysqlConnection> reference;
                    while ((reference = referenceQueue.poll()) != null) {
                        finalizeResource((ConnectionFinalizerPhantomReference) reference);
                    }
                    connectionFinalizerPhantomRefs.clear();
                } finally {
                    threadRefLock.unlock();
                }
                return;
            } catch (Exception ex) {
                // Nowhere to really log this.
            }
        }
    }
}
```

在这个类中为我们展示了一种关闭被忘记释放的IO资源的方式。首先`ReferenceQueue<MysqlConnection> referenceQueue`会接收垃圾回收器将要收集的`MysqlConnection`实例，然后单线程执行器`cleanupThreadExecutorService`会运行`AbandonedConnectionCleanupThread`的`run()`方法来释放资源`finalizeResource((ConnectionFinalizerPhantomReference) reference)`。

当这些初始化完成后，`Driver`会将自己注册到Jdbc中，如下主流程分析小节中的[ref-1](#driverInitialize)处代码所示，下面是`ref-1`处代码调用的`registerDriver(...)`函数细节：

```java
// java.sql.DriverManager.java文件
public class DriverManager {
    public static void registerDriver(java.sql.Driver driver)
        throws SQLException {

        registerDriver(driver, null);
    }
    public static void registerDriver(java.sql.Driver driver,
                                      DriverAction da)
        throws SQLException {

        /* Register the driver if it has not already been added to our list */
        if (driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da)); // 添加驱动器到列表
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }
}
```

`registerDriver(...)`方法其实就是将驱动器类实例添加到一个列表里面，列表定义在`ref-4`处代码。

到这里，我们就把MySQL驱动程序的初始化流程分析结束了。



### 3、获取连接的流程

Jdbc的驱动管理类`DriverManager`是通过MySQL驱动器类`Driver`获取连接的，如主流程分析小节中代码[ref-2](#getConnection)所示，下面是ref-2处代码调用的`connect(...)`函数细节。

```java
// com.mysql.cj.jdbc.NonRegisteringDriver.java文件
@Override
public java.sql.Connection connect(String url, Properties info) throws SQLException {

    try {
        if (!ConnectionUrl.acceptsUrl(url)) {
            /*
                 * According to JDBC spec:
                 * The driver should return "null" if it realizes it is the wrong kind of driver 
                 * to connect to the given URL. This will be common, as when the
                 * JDBC driver manager is asked to connect to a given URL it passes the URL to 
                 * each loaded driver in turn.
                 */
            return null;
        }

        ConnectionUrl conStr = ConnectionUrl.getConnectionUrlInstance(url, info);
        switch (conStr.getType()) { // 支持多种类型的连接
            case SINGLE_CONNECTION:
                // ref-5
                return com.mysql.cj.jdbc.ConnectionImpl.getInstance(conStr.getMainHost());
                
            case FAILOVER_CONNECTION:
            case FAILOVER_DNS_SRV_CONNECTION:
                return FailoverConnectionProxy.createProxyInstance(conStr);

            case LOADBALANCE_CONNECTION:
            case LOADBALANCE_DNS_SRV_CONNECTION:
                return LoadBalancedConnectionProxy.createProxyInstance(conStr);

            case REPLICATION_CONNECTION:
            case REPLICATION_DNS_SRV_CONNECTION:
                return ReplicationConnectionProxy.createProxyInstance(conStr);

            default:
                return null;
        }

    } catch (UnsupportedConnectionStringException e) {
        // when Connector/J can't handle this connection string the Driver must return null
        return null;

    } catch (CJException ex) {
        throw ExceptionFactory.createException(UnableToConnectException.class,
                                               Messages.getString("NonRegisteringDriver.17", new Object[] { ex.toString() }), ex);
    }
}
```

从代码中可以看到MySQL驱动程序支持多种连接url，在我们的示例程序中是`SINGLE_CONNECTION`类型，也就是`ref-5`处的代码。下面我们详细看看创建连接实例的过程：

```java
// com.mysql.cj.jdbc.ConnectionImpl.java文件  

/**
 * Creates a connection instance.
 * 
 */
public static JdbcConnection getInstance(HostInfo hostInfo) throws SQLException {
    return new ConnectionImpl(hostInfo);
}

/**
 * Creates a connection to a MySQL Server.
 * 
 */
public ConnectionImpl(HostInfo hostInfo) throws SQLException {
    try {
        // Stash away for later, used to clone this connection for Statement.cancel and Statement.setQueryTimeout().
        this.origHostInfo = hostInfo;
        this.origHostToConnectTo = hostInfo.getHost();
        this.origPortToConnectTo = hostInfo.getPort();

        this.database = hostInfo.getDatabase();
        this.user = hostInfo.getUser();
        this.password = hostInfo.getPassword();

        this.props = hostInfo.exposeAsProperties();

        this.propertySet = new JdbcPropertySetImpl();

        this.propertySet.initializeProperties(this.props);

        // We need Session ASAP to get access to central driver functionality
        this.nullStatementResultSetFactory = new ResultSetFactory(this, null);
        this.session = new NativeSession(hostInfo, this.propertySet); // ref-7
        this.session.addListener(this); // listen for session status changes

		...... 设置各种属性

        // 创建异常拦截器
        String exceptionInterceptorClasses = this.propertySet.getStringProperty(PropertyKey.exceptionInterceptors).getStringValue();
        if (exceptionInterceptorClasses != null && !"".equals(exceptionInterceptorClasses)) {
            this.exceptionInterceptor = new ExceptionInterceptorChain(exceptionInterceptorClasses, this.props, this.session.getLog());
        }
		// 创建PreparedStatement的缓存
        if (this.cachePrepStmts.getValue()) {
            createPreparedStatementCaches();
        }

      	......获取缓存、连接和查询相关的配置属性

        this.dbmd = getMetaData(false, false);

        initializeSafeQueryInterceptors();

    } catch (CJException e1) {
        throw SQLExceptionsMapping.translateException(e1, getExceptionInterceptor());
    }

    try {
        createNewIO(false); // 底层使用socket连接MySQL数据库

        unSafeQueryInterceptors(); // ref-7

        AbandonedConnectionCleanupThread.trackConnection(this, this.getSession().getNetworkResources()); // ref-8
    } catch (SQLException ex) {
		......省略
    } catch (Exception ex) {
        ...... 省略
    }
}
```

ref-7处代码是在为会话session设置查询拦截器，代码如下所示：

```java
// com.mysql.cj.jdbc.ConnectionImpl.java文件  
@Override
public void unSafeQueryInterceptors() throws SQLException {
    this.queryInterceptors = this.queryInterceptors.stream().map(u -> ((NoSubInterceptorWrapper) u).getUnderlyingInterceptor())
        .collect(Collectors.toList());
	// 这个session就是在构造器方法中ref-6处创建的NativeSession类的实例。
    if (this.session != null) {
        this.session.setQueryInterceptors(this.queryInterceptors);
    }
}
```

ref-8处代码将当前新创建的连接添加到遗弃连接清理线程`AbandonedConnectionCleanupThread`中，当该连接被丢掉的时候就会被捕捉并清理，`AbandonedConnectionCleanupThread`在[MySQL驱动程序初始化](#cleanThreadInit)的时候就创建了一个线程池专门运行连接清理任务。



### 4、执行SQL语句的流程

在实例程序中，拿到数据库连接后会创建`Statement`，然后通过它执行SQL语句获取结果集。

```java
Statement statement = conn.createStatement();
ResultSet resultSet = statement.executeQuery("SELECT * FROM students"); // ref-9
```

ref-9处代码背后就是MySQL驱动程序执行SQL语句的逻辑，详细代码如下所示：

```java
@Override
public java.sql.ResultSet executeQuery(String sql) throws SQLException {
    synchronized (checkClosed().getConnectionMutex()) {
        JdbcConnection locallyScopedConn = this.connection;
        this.retrieveGeneratedKeys = false;
        checkNullOrEmptyQuery(sql);
        resetCancelledState();
        implicitlyCloseAllOpenResults();
        if (sql.charAt(0) == '/') { // 执行ping命令
            if (sql.startsWith(PING_MARKER)) {
                doPingInstead();
                return this.results;
            }
        }
        setupStreamingTimeout(locallyScopedConn);
        if (this.doEscapeProcessing) {
            ...... 处理SQL语句中的转义
        }

        if (!isResultSetProducingQuery(sql)) { // 判断SQL语句是否会产生查询结果。
            throw SQLError.createSQLException(Messages.getString("Statement.57"), MysqlErrorNumbers.SQL_STATE_ILLEGAL_ARGUMENT, getExceptionInterceptor());
        }

        CachedResultSetMetaData cachedMetaData = null;

        if (useServerFetch()) {  // 判断是否使用MySQl服务器的fetch功能
            this.results = createResultSetUsingServerFetch(sql);
            return this.results;
        }

        CancelQueryTask timeoutTask = null;
        String oldDb = null;

        try {
            timeoutTask = startQueryTimer(this, getTimeoutInMillis()); // ref-10
            if (!locallyScopedConn.getDatabase().equals(getCurrentDatabase())) {
                oldDb = locallyScopedConn.getDatabase();
                locallyScopedConn.setDatabase(getCurrentDatabase());
            }

            //
            // Check if we have cached metadata for this query...
            //
            if (locallyScopedConn.getPropertySet().getBooleanProperty(PropertyKey.cacheResultSetMetadata).getValue()) {
                cachedMetaData = locallyScopedConn.getCachedMetaData(sql);
            }

            locallyScopedConn.setSessionMaxRows(this.maxRows);
            statementBegins();
            // 获取session并执行sql // ref-11
            this.results = ((NativeSession) locallyScopedConn.getSession()).execSQL(this, sql, this.maxRows, null, createStreamingResultSet(),
                                                                                    getResultSetFactory(), cachedMetaData, false);
            if (timeoutTask != null) {
                // 关闭超时控制任务
                stopQueryTimer(timeoutTask, true, true); // ref-12
                timeoutTask = null;
            }
        } catch (CJTimeoutException | OperationCancelledException e) {
            throw SQLExceptionsMapping.translateException(e, this.exceptionInterceptor);
        } finally {
            ......省略
        }
        ...... 判断是否需要更新查询结果的缓存，如果需要的话就执行更新操作。
            return this.results;
    }
}
```

ref-10处代码是在进行查询任务的超时控制，<span id= "execSQL">ref-11处代码会获取session执行sql语句</span>。

在整个方法中，只看见了超时控制任务`timeoutTask`的创建，并没有看见它的执行，我们来一探究竟。先看ref-10处中调用的`startQueryTimer`方法。

```java
// com.mysql.cj.jdbc.StatementImpl.java文件
@Override
public CancelQueryTask startQueryTimer(Query stmtToCancel, int timeout) {
    return this.query.startQueryTimer(stmtToCancel, timeout);
}
```

`this.query`变量的类型为`com.mysql.cj.SimpleQuery`，下面是类型定义：

```java
// com.mysql.cj.SimpleQuery.java文件
package com.mysql.cj;

public class SimpleQuery extends AbstractQuery {

    public SimpleQuery(NativeSession sess) {
        super(sess);
    }

}
```

看来`this.query`的`startQueryTimer(...)`方法在父类中，如下所示：

```java
// com.mysql.cj.AbstractQuery.java文件
public CancelQueryTask startQueryTimer(Query stmtToCancel, int timeout) {
    if (this.session.getPropertySet().getBooleanProperty(PropertyKey.enableQueryTimeouts).getValue() && timeout != 0) {
        // 创建取消查询的任务
        CancelQueryTaskImpl timeoutTask = new CancelQueryTaskImpl(stmtToCancel);
        // 将控制查询超时的任务放在Timer中进行调度，并指定执行的延迟时间
        this.session.getCancelTimer().schedule(timeoutTask, timeout);
        return timeoutTask;
    }
    return null;
}
```

超时控制是使用了Java的`Timer`来完成的，如果SQL语句顺利执行，超时控制任务会被关闭，如代码ref-12所示，最终会调用`CancelQueryTask#cancel`方法和`Timer#purge`方法来关闭超时控制任务。那么超时控制任务`CancelQueryTaskImpl`干了什么事情呢？

```java
// com.mysql.cj.CancelQueryTaskImpl.java文件
package com.mysql.cj;

/**
 * Thread used to implement query timeouts...Eventually we could be more
 * efficient and have one thread with timers, but this is a straightforward
 * and simple way to implement a feature that isn't used all that often.
 */
public class CancelQueryTaskImpl extends TimerTask implements CancelQueryTask {

    Query queryToCancel;
    Throwable caughtWhileCancelling = null;
    boolean queryTimeoutKillsConnection = false;

    public CancelQueryTaskImpl(Query cancellee) {
        this.queryToCancel = cancellee;
        NativeSession session = (NativeSession) cancellee.getSession();
        this.queryTimeoutKillsConnection = session.getPropertySet().getBooleanProperty(PropertyKey.queryTimeoutKillsConnection).getValue();
    }

    @Override
    public boolean cancel() {
        boolean res = super.cancel();
        this.queryToCancel = null;
        return res;
    }

    @Override
    public void run() {

        Thread cancelThread = new Thread() {

            @Override
            public void run() {
                Query localQueryToCancel = CancelQueryTaskImpl.this.queryToCancel;
                if (localQueryToCancel == null) {
                    return;
                }
                NativeSession session = (NativeSession) localQueryToCancel.getSession();
                if (session == null) {
                    return;
                }

                try {
                    if (CancelQueryTaskImpl.this.queryTimeoutKillsConnection) {
                        // 设置查询的取消状态
                        localQueryToCancel.setCancelStatus(CancelStatus.CANCELED_BY_TIMEOUT);
						// 通知做清理工作的监听器
                        session.invokeCleanupListeners(new OperationCancelledException(Messages.getString("Statement.ConnectionKilledDueToTimeout")));
                    } else {
                        synchronized (localQueryToCancel.getCancelTimeoutMutex()) {
                            long origConnId = session.getThreadId();
                            HostInfo hostInfo = session.getHostInfo();
                            String database = hostInfo.getDatabase();
                            String user = hostInfo.getUser();
                            String password = hostInfo.getPassword();

                            NativeSession newSession = null;
                            try {
                                // step1 创建session
                                newSession = new NativeSession(hostInfo, session.getPropertySet());		 // step2 连接到数据库
                                newSession.connect(hostInfo, user, password, database, 30000, new TransactionEventHandler() {
                                    @Override
                                    public void transactionCompleted() {
                                    }

                                    public void transactionBegun() {
                                    }
                                });
                                // step3 向数据库发送杀掉指定查询的命令
                                newSession.getProtocol().sendCommand(new NativeMessageBuilder(newSession.getServerSession().supportsQueryAttributes())
                                                                     .buildComQuery(newSession.getSharedSendPacket(), "KILL QUERY " + origConnId), false, 0);
                            } finally {
                                try {
                                    newSession.forceClose();
                                } catch (Throwable t) {
                                    // no-op.
                                }
                            }
                            localQueryToCancel.setCancelStatus(CancelStatus.CANCELED_BY_TIMEOUT);
                        }
                    }
                    // } catch (NullPointerException npe) {
                    // Case when connection closed while starting to cancel.
                    // We can't easily synchronize this, because then one thread can't cancel() a running query.
                    // Ignore, we shouldn't re-throw this, because the connection's already closed, so the statement has been timed out.
                } catch (Throwable t) {
                    CancelQueryTaskImpl.this.caughtWhileCancelling = t;
                } finally {
                    setQueryToCancel(null);
                }
            }
        };
		// step4 开启执行杀掉查询的线程
        cancelThread.start();
    }

    public Throwable getCaughtWhileCancelling() {
        return this.caughtWhileCancelling;
    }

    public void setCaughtWhileCancelling(Throwable caughtWhileCancelling) {
        this.caughtWhileCancelling = caughtWhileCancelling;
    }

    public Query getQueryToCancel() {
        return this.queryToCancel;
    }

    public void setQueryToCancel(Query queryToCancel) {
        this.queryToCancel = queryToCancel;
    }
}
```

可以看到查询任务是由`Timer`在指定延迟时间后被触发的，但是并没有在Timer中执行停止查询的操作，而是新创建一个线程单独来执行，这里应该是防止停止查询的操作执行时间过长会影响`Timer`。通过连接到数据库服务器，然后发送终止查询任务的命令`KILL QUERY`让数据库服务器处理已经超时的SQL语句。

至此，查询的超时控制就分析完了，接下来继续分析SQL语句的实际执行。[ref-11](#execSQL)处代码调用的`execSQL(...)`方法如下所示：

```java
// com.mysql.cj.NativeSession.java文件   
/**
 * Send a query to the server. Returns one of the ResultSet objects.
 * To ensure that Statement's queries are serialized, calls to this method
 * should be enclosed in a connection mutex synchronized block.
 * 
 * @param <T> extends {@link Resultset}
 * @param callingQuery {@link Query} object
 * @param query the SQL statement to be executed
 * @param maxRows rows limit
 * @param packet {@link NativePacketPayload}
 * @param streamResults whether a stream result should be created
 * @param resultSetFactory {@link ProtocolEntityFactory}
 * @param cachedMetadata use this metadata instead of the one provided on wire
 * @param isBatch is it a batch query
 * @return a ResultSet holding the results
 */
public <T extends Resultset> T execSQL(Query callingQuery, String query, int maxRows, NativePacketPayload packet, boolean streamResults,
ProtocolEntityFactory<T, NativePacketPayload> resultSetFactory, ColumnDefinition cachedMetadata, boolean isBatch) {

    long queryStartTime = this.gatherPerfMetrics.getValue() ? System.currentTimeMillis() : 0;
    int endOfQueryPacketPosition = packet != null ? packet.getPosition() : 0;

    this.lastQueryFinishedTime = 0; // we're busy!

    if (this.autoReconnect.getValue() && (getServerSession().isAutoCommit() || this.autoReconnectForPools.getValue()) && this.needsPing && !isBatch) {
        try {
            ping(false, 0);
            this.needsPing = false;

        } catch (Exception Ex) {
            invokeReconnectListeners();
        }
    }

    try {
        // 通过私有协议向MySQL服务器发送SQL语句数据
        return packet == null
            ? ((NativeProtocol) this.protocol).sendQueryString(callingQuery, query, this.characterEncoding.getValue(), maxRows, streamResults, cachedMetadata, resultSetFactory)
            : ((NativeProtocol) this.protocol).sendQueryPacket(callingQuery, packet, maxRows, streamResults, cachedMetadata, resultSetFactory);
    } catch (CJException sqlE) {
		......省略
        throw sqlE;
    } catch (Throwable ex) {
        ......省略
        throw ExceptionFactory.createException(ex.getMessage(), ex, this.exceptionInterceptor);
    } finally {
        ......省略
    }
```

由于调用栈上层传递的是SQL语句字符串，所以驱动程序会通过私有协议`NativeProtocol`向MySQL服务器发送SQL语句`query`，下面是发送语句的细节：

```java
// com.mysql.cj.protocol.a.NativeProtocol.java文件
public final <T extends Resultset> T sendQueryString(Query callingQuery, String query, String characterEncoding, int maxRows, boolean streamResults,
                                                     ColumnDefinition cachedMetadata, ProtocolEntityFactory<T, NativePacketPayload> resultSetFactory) throws IOException {
    String statementComment = this.queryComment;
	// 将线程的名称设置为语句Statement的备注。
    if (this.propertySet.getBooleanProperty(PropertyKey.includeThreadNamesAsStatementComment).getValue()) {
        statementComment = (statementComment != null ? statementComment + ", " : "") + "java thread: " + Thread.currentThread().getName();
    }

    // We don't know exactly how many bytes we're going to get from the query. Since we're dealing with UTF-8, the max is 4, so pad it
    // (4 * query) + space for headers
    int packLength = 1 /* com_query */ + (query.length() * 4) + 2;
	// 计算packet数据的长度
    // TODO decide how to safely use the shared this.sendPacket
    NativePacketPayload sendPacket = new NativePacketPayload(packLength);

    sendPacket.setPosition(0);
    sendPacket.writeInteger(IntegerDataType.INT1, NativeConstants.COM_QUERY);

    if (supportsQueryAttributes) {
        ......将属性数据写入到sendPacket
    }
    sendPacket.setTag("QUERY");
    if (commentAsBytes != null) {
        ......将备注数据写入到sendPacket
    }
	// 将SQL语句数据写入到sendPacket
    if (!this.session.getServerSession().getCharsetSettings().doesPlatformDbCharsetMatches() && StringUtils.startsWithIgnoreCaseAndWs(query, "LOAD DATA")) {
        sendPacket.writeBytes(StringLengthDataType.STRING_FIXED, StringUtils.getBytes(query));
    } else {
        sendPacket.writeBytes(StringLengthDataType.STRING_FIXED, StringUtils.getBytes(query, characterEncoding));
    }
	// 将sendPacket发送到MySQL服务器
    return sendQueryPacket(callingQuery, sendPacket, maxRows, streamResults, cachedMetadata, resultSetFactory);
}
```

SQL语句报文最终会通过socket发送到MySQL服务器，详细语句如下：

```java
// com.mysql.cj.protocol.a.SimplePacketSender.java文件
package com.mysql.cj.protocol.a;
/**
 * Simple implementation of {@link MessageSender} which handles the transmission of logical MySQL 
 * packets to the provided output stream. Large packets will be split into multiple chunks.
 */
public class SimplePacketSender implements MessageSender<NativePacketPayload> {

    public void send(byte[] packet, int packetLen, byte packetSequence) throws IOException {
        PacketSplitter packetSplitter = new PacketSplitter(packetLen);
        while (packetSplitter.nextPacket()) {
            this.outputStream.write(NativeUtils.encodeMysqlThreeByteInteger(packetSplitter.getPacketLen()));
            this.outputStream.write(packetSequence++);
            this.outputStream.write(packet, packetSplitter.getOffset(), packetSplitter.getPacketLen());
        }
        this.outputStream.flush();
    }
    ......省略其他方法
}
```

当运行到`send(...)`方法时，相应变量实际信息如下图所示：

![image-20220930141540221](images/SimplePacketSender的send方法调用栈信息.png)

从Debug信息中可以看到`outputStream`连接到的就是数据库服务器。

SQL语句数据发送到MySQL服务器后会被执行，然后会将执行结果返回驱动程序，如下所示：

```java
// com.mysql.cj.protocol.a.NativeProtocol.java文件
public final <T extends Resultset> T sendQueryPacket(Query callingQuery, NativePacketPayload queryPacket, int maxRows, boolean streamResults,ColumnDefinition cachedMetadata, ProtocolEntityFactory<T, NativePacketPayload> resultSetFactory) throws IOException {

    final long queryStartTime = getCurrentTimeNanosOrMillis();

    this.statementExecutionDepth++;

    byte[] queryBuf = queryPacket.getByteBuffer();
    int oldPacketPosition = queryPacket.getPosition(); // save the packet position
    int queryPosition = queryPacket.getTag("QUERY");
    LazyString query = new LazyString(queryBuf, queryPosition, (oldPacketPosition - queryPosition));

    try {
        ......省略 准备工作
        // Send query command and sql query string
        NativePacketPayload resultPacket = sendCommand(queryPacket, false, 0);
		......省略 搜集本地查询执行过程的指标数据
        // 读取SQL语句执行的结果数据
        T rs = readAllResults(maxRows, streamResults, resultPacket, false, cachedMetadata, resultSetFactory);
		......省略 对SQL执行结果进行一些处理
        if (this.hadWarnings) {
            scanForAndThrowDataTruncation();
        }

        if (this.queryInterceptors != null) {
            rs = invokeQueryInterceptorsPost(query, callingQuery, rs, false);
        }

        return rs;

    } catch (CJException sqlEx) {
       ......省略
    } finally {
        this.statementExecutionDepth--;
    }
}
```

好了，到这儿，Jdbc通过MySQL驱动程序执行SQL语句的全部流程就分析完了。



### 5、Jdbc中迷惑的驱动程序加载顺序

在分析Jdbc驱动程序加载过程的时候，对于加载顺序有迷惑，下面向贴出详细代码片段，然后再说迷惑的地方。

```java
// java.sql.DriverManager.java文件
/*
 * Load the initial JDBC drivers by checking the System property
 * jdbc.drivers and then use the {@code ServiceLoader} mechanism
 */
@SuppressWarnings("removal")
private static void ensureDriversInitialized() {
    if (driversInitialized) {
        return;
    }

    synchronized (lockForInitDrivers) {
        if (driversInitialized) {
            return;
        }
        String drivers;
        try {
            // step1 获取属性中设置的驱动器类
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty(JDBC_DRIVERS_PROPERTY);
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }

        // If the driver is packaged as a Service Provider, load it.
        // Get all the drivers through the classloader
        // exposed as a java.sql.Driver.class service.
        // ServiceLoader.load() replaces the sun.misc.Providers()
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
				// step2 SPI方式：ServiceLoader获取Driver实现类
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                /* Load these drivers, so that they can be instantiated.
                     * It may be the case that the driver class may not be there
                     * i.e. there may be a packaged driver with the service class
                     * as implementation of java.sql.Driver but the actual class
                     * may be missing. In that case a java.util.ServiceConfigurationError
                     * will be thrown at runtime by the VM trying to locate
                     * and load the service.
                     *
                     * Adding a try catch block to catch those runtime errors
                     * if driver not available in classpath but it's
                     * packaged as service and that service is there in classpath.
                     */
                try {
                    while (driversIterator.hasNext()) {
                        // step3 加载ServiceLoader找到的驱动器类
                        driversIterator.next();
                    }
                } catch (Throwable t) {
                    // Do nothing
                }
                return null;
            }
        });

        println("DriverManager.initialize: jdbc.drivers = " + drivers);

        if (drivers != null && !drivers.isEmpty()) {
            String[] driversList = drivers.split(":");
            println("number of Drivers:" + driversList.length);
            for (String aDriver : driversList) {
                try {
                    println("DriverManager.Initialize: loading " + aDriver);
                    // step4 加载属性中设置的驱动器类
                    Class.forName(aDriver, true,
                                  ClassLoader.getSystemClassLoader());
                } catch (Exception ex) {
                    println("DriverManager.Initialize: load failed: " + ex);
                }
            }
        }

        driversInitialized = true;
        println("JDBC DriverManager initialized");
    }
}
```

在step1的时候就已经获取到属性中设置的驱动器类了，为啥要等到ServiceLoader获取完驱动器类并加载完之后才在step4加载属性中设置的驱动器类？

