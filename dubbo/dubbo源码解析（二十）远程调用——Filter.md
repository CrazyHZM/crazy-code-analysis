# 远程调用——Filter

> 目标：介绍dubbo-rpc-api中的各种filter过滤器的实现逻辑。

## 前言

本文会介绍在dubbo中的过滤器，先来看看下面的图：

![dubbo-framework](https://github.com/CrazyHZM/crazy-code-analysis/blob/master/dubbo/image/%E4%BA%8C%E5%8D%81/dubbo-framework.jpg?raw=true)

可以看到红色圈圈不服，在服务发现和服务引用中都会进行一些过滤器过滤。具体有哪些过滤器，就看下面的介绍。

## 源码分析

### （一）AccessLogFilter

该过滤器是对记录日志的过滤器，它所做的工作就是把引用服务或者暴露服务的调用链信息写入到文件中。日志消息先被放入日志集合，然后加入到日志队列，然后被放入到写入文件到任务中，最后进入文件。

#### 1.属性

```java
private static final Logger logger = LoggerFactory.getLogger(AccessLogFilter.class);

/**
 * 日志访问名称，默认的日志访问名称
 */
private static final String ACCESS_LOG_KEY = "dubbo.accesslog";

/**
 * 日期格式
 */
private static final String FILE_DATE_FORMAT = "yyyyMMdd";

private static final String MESSAGE_DATE_FORMAT = "yyyy-MM-dd HH:mm:ss";

/**
 * 日志队列大小
 */
private static final int LOG_MAX_BUFFER = 5000;

/**
 * 日志输出的频率
 */
private static final long LOG_OUTPUT_INTERVAL = 5000;

/**
 * 日志队列 key为访问日志的名称，value为该日志名称对应的日志集合
 */
private final ConcurrentMap<String, Set<String>> logQueue = new ConcurrentHashMap<String, Set<String>>();

/**
 * 日志线程池
 */
private final ScheduledExecutorService logScheduled = Executors.newScheduledThreadPool(2, new NamedThreadFactory("Dubbo-Access-Log", true));

/**
 * 日志记录任务
 */
private volatile ScheduledFuture<?> logFuture = null;
```

按照我上面讲到日志流向，日志先进入到是日志队列中的日志集合，再进入logQueue，在进入logFuture，最后落地到文件。

#### 2.init

```java
private void init() {
    // synchronized是一个重操作消耗性能，所有加上判空
    if (logFuture == null) {
        synchronized (logScheduled) {
            // 为了不重复初始化
            if (logFuture == null) {
                // 创建日志记录任务
                logFuture = logScheduled.scheduleWithFixedDelay(new LogTask(), LOG_OUTPUT_INTERVAL, LOG_OUTPUT_INTERVAL, TimeUnit.MILLISECONDS);
            }
        }
    }
}
```

该方法是初始化方法，就创建了日志记录任务。

#### 3.log

```java
private void log(String accesslog, String logmessage) {
    init();
    Set<String> logSet = logQueue.get(accesslog);
    if (logSet == null) {
        logQueue.putIfAbsent(accesslog, new ConcurrentHashSet<String>());
        logSet = logQueue.get(accesslog);
    }
    if (logSet.size() < LOG_MAX_BUFFER) {
        logSet.add(logmessage);
    }
}
```

该方法是增加日志信息到日志集合中。

#### 4.invoke

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    try {
        // 获得日志名称
        String accesslog = invoker.getUrl().getParameter(Constants.ACCESS_LOG_KEY);
        if (ConfigUtils.isNotEmpty(accesslog)) {
            // 获得rpc上下文
            RpcContext context = RpcContext.getContext();
            // 获得调用的接口名称
            String serviceName = invoker.getInterface().getName();
            // 获得版本号
            String version = invoker.getUrl().getParameter(Constants.VERSION_KEY);
            // 获得组，是消费者侧还是生产者侧
            String group = invoker.getUrl().getParameter(Constants.GROUP_KEY);
            StringBuilder sn = new StringBuilder();
            sn.append("[").append(new SimpleDateFormat(MESSAGE_DATE_FORMAT).format(new Date())).append("] ").append(context.getRemoteHost()).append(":").append(context.getRemotePort())
                    .append(" -> ").append(context.getLocalHost()).append(":").append(context.getLocalPort())
                    .append(" - ");
            // 拼接组
            if (null != group && group.length() > 0) {
                sn.append(group).append("/");
            }
            // 拼接服务名称
            sn.append(serviceName);
            // 拼接版本号
            if (null != version && version.length() > 0) {
                sn.append(":").append(version);
            }
            sn.append(" ");
            // 拼接方法名
            sn.append(inv.getMethodName());
            sn.append("(");
            // 拼接参数类型
            Class<?>[] types = inv.getParameterTypes();
            // 拼接参数类型
            if (types != null && types.length > 0) {
                boolean first = true;
                for (Class<?> type : types) {
                    if (first) {
                        first = false;
                    } else {
                        sn.append(",");
                    }
                    sn.append(type.getName());
                }
            }
            sn.append(") ");
            // 拼接参数
            Object[] args = inv.getArguments();
            if (args != null && args.length > 0) {
                sn.append(JSON.toJSONString(args));
            }
            String msg = sn.toString();
            // 如果用默认的日志访问名称
            if (ConfigUtils.isDefault(accesslog)) {
                LoggerFactory.getLogger(ACCESS_LOG_KEY + "." + invoker.getInterface().getName()).info(msg);
            } else {
                // 把日志加入集合
                log(accesslog, msg);
            }
        }
    } catch (Throwable t) {
        logger.warn("Exception in AcessLogFilter of service(" + invoker + " -> " + inv + ")", t);
    }
    // 调用下一个调用链
    return invoker.invoke(inv);
}
```

该方法是最重要的方法，从拼接了日志信息，把日志加入到集合，并且调用下一个调用链。

#### 4.LogTask

```java
private class LogTask implements Runnable {
    @Override
    public void run() {
        try {
            if (logQueue != null && logQueue.size() > 0) {
                // 遍历日志队列
                for (Map.Entry<String, Set<String>> entry : logQueue.entrySet()) {
                    try {
                        // 获得日志名称
                        String accesslog = entry.getKey();
                        // 获得日志集合
                        Set<String> logSet = entry.getValue();
                        // 如果文件不存在则创建文件
                        File file = new File(accesslog);
                        File dir = file.getParentFile();
                        if (null != dir && !dir.exists()) {
                            dir.mkdirs();
                        }
                        if (logger.isDebugEnabled()) {
                            logger.debug("Append log to " + accesslog);
                        }
                        if (file.exists()) {
                            // 获得现在的时间
                            String now = new SimpleDateFormat(FILE_DATE_FORMAT).format(new Date());
                            // 获得文件最后一次修改的时间
                            String last = new SimpleDateFormat(FILE_DATE_FORMAT).format(new Date(file.lastModified()));
                            // 如果文件最后一次修改的时间不等于现在的时间
                            if (!now.equals(last)) {
                                // 获得重新生成文件名称
                                File archive = new File(file.getAbsolutePath() + "." + last);
                                // 因为都是file的绝对路径，所以没有进行移动文件，而是修改文件名
                                file.renameTo(archive);
                            }
                        }
                        // 把日志集合中的日志写入到文件
                        FileWriter writer = new FileWriter(file, true);
                        try {
                            for (Iterator<String> iterator = logSet.iterator();
                                 iterator.hasNext();
                                 iterator.remove()) {
                                writer.write(iterator.next());
                                writer.write("\r\n");
                            }
                            writer.flush();
                        } finally {
                            writer.close();
                        }
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
    }
}
```

该内部类实现了Runnable，是把日志消息落地到文件到线程。

### （二）ActiveLimitFilter

该类时对于每个服务的每个方法的最大可并行调用数量限制的过滤器，它是在服务消费者侧的过滤。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 获得url对象
    URL url = invoker.getUrl();
    // 获得方法名称
    String methodName = invocation.getMethodName();
    // 获得并发调用数（单个服务的单个方法），默认为0
    int max = invoker.getUrl().getMethodParameter(methodName, Constants.ACTIVES_KEY, 0);
    // 通过方法名来获得对应的状态
    RpcStatus count = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName());
    if (max > 0) {
        // 获得该方法调用的超时次数
        long timeout = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, 0);
        // 获得系统时间
        long start = System.currentTimeMillis();
        long remain = timeout;
        // 获得该方法的调用数量
        int active = count.getActive();
        // 如果活跃数量大于等于最大的并发调用数量
        if (active >= max) {
            synchronized (count) {
                // 当活跃数量大于等于最大的并发调用数量时一直循环
                while ((active = count.getActive()) >= max) {
                    try {
                        // 等待超时时间
                        count.wait(remain);
                    } catch (InterruptedException e) {
                    }
                    // 获得累计时间
                    long elapsed = System.currentTimeMillis() - start;
                    remain = timeout - elapsed;
                    // 如果累计时间大于超时时间，则抛出异常
                    if (remain <= 0) {
                        throw new RpcException("Waiting concurrent invoke timeout in client-side for service:  "
                                + invoker.getInterface().getName() + ", method: "
                                + invocation.getMethodName() + ", elapsed: " + elapsed
                                + ", timeout: " + timeout + ". concurrent invokes: " + active
                                + ". max concurrent invoke limit: " + max);
                    }
                }
            }
        }
    }
    try {
        // 获得系统时间作为开始时间
        long begin = System.currentTimeMillis();
        // 开始计数
        RpcStatus.beginCount(url, methodName);
        try {
            // 调用后面的调用链，如果没有抛出异常，则算成功
            Result result = invoker.invoke(invocation);
            // 结束计数，记录时间
            RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, true);
            return result;
        } catch (RuntimeException t) {
            RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, false);
            throw t;
        }
    } finally {
        if (max > 0) {
            synchronized (count) {
                // 唤醒count
                count.notify();
            }
        }
    }
}
```

该类只有这一个方法。该过滤器是用来限制调用数量，先进行调用数量的检测，如果没有到达最大的调用数量，则先调用后面的调用链，如果在后面的调用链失败，则记录相关时间，如果成功也记录相关时间和调用次数。

### （三）ClassLoaderFilter

该过滤器是做类加载器切换的。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 获得当前的类加载器
    ClassLoader ocl = Thread.currentThread().getContextClassLoader();
    // 设置invoker携带的服务的类加载器
    Thread.currentThread().setContextClassLoader(invoker.getInterface().getClassLoader());
    try {
        // 调用下面的调用链
        return invoker.invoke(invocation);
    } finally {
        // 最后切换回原来的类加载器
        Thread.currentThread().setContextClassLoader(ocl);
    }
}
```

可以看到先切换成当前的线程锁携带的类加载器，然后调用结束后，再切换回原先的类加载器。

### （四）CompatibleFilter

该过滤器是做兼容性的过滤器。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 调用下一个调用链
    Result result = invoker.invoke(invocation);
    // 如果方法前面没有$或者结果没有异常
    if (!invocation.getMethodName().startsWith("$") && !result.hasException()) {
        Object value = result.getValue();
        if (value != null) {
            try {
                // 获得方法
                Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                // 获得返回的数据类型
                Class<?> type = method.getReturnType();
                Object newValue;
                // 序列化方法
                String serialization = invoker.getUrl().getParameter(Constants.SERIALIZATION_KEY);
                // 如果是json或者fastjson形式
                if ("json".equals(serialization)
                        || "fastjson".equals(serialization)) {
                    // 获得方法的泛型返回值类型
                    Type gtype = method.getGenericReturnType();
                    // 把数据结果进行类型转化
                    newValue = PojoUtils.realize(value, type, gtype);
                    // 如果value不是type类型
                } else if (!type.isInstance(value)) {
                    // 如果是pojo，则，转化为type类型，如果不是，则进行兼容类型转化。
                    newValue = PojoUtils.isPojo(type)
                            ? PojoUtils.realize(value, type)
                            : CompatibleTypeUtils.compatibleTypeConvert(value, type);

                } else {
                    newValue = value;
                }
                // 重新设置RpcResult的result
                if (newValue != value) {
                    result = new RpcResult(newValue);
                }
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
    return result;
}
```

可以看到对于调用链的返回结果，如果返回值类型和返回值不一样的时候，就需要做兼容类型的转化。重新把结果放入RpcResult，返回。

### （五）ConsumerContextFilter

该过滤器做的是在当前的RpcContext中记录本地调用的一次状态信息。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 设置rpc上下文
    RpcContext.getContext()
            .setInvoker(invoker)
            .setInvocation(invocation)
            .setLocalAddress(NetUtils.getLocalHost(), 0)
            .setRemoteAddress(invoker.getUrl().getHost(),
                    invoker.getUrl().getPort());
    // 如果该会话域是rpc会话域
    if (invocation instanceof RpcInvocation) {
        // 设置实体域
        ((RpcInvocation) invocation).setInvoker(invoker);
    }
    try {
        // 调用下个调用链
        RpcResult result = (RpcResult) invoker.invoke(invocation);
        // 设置附加值
        RpcContext.getServerContext().setAttachments(result.getAttachments());
        return result;
    } finally {
        // 情况附加值
        RpcContext.getContext().clearAttachments();
    }
}
```

可以看到RpcContext记录了一次调用状态信息，然后先调用后面的调用链，再回来把附加值设置到RpcContext中。然后返回RpcContext，再清空，这样是因为后面的调用链中的附加值对前面的调用链是不可见的。

### （六）ContextFilter

该过滤器做的是初始化rpc上下文。

```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 获得会话域的附加值
        Map<String, String> attachments = invocation.getAttachments();
        // 删除异步属性以避免传递给以下调用链
        if (attachments != null) {
            attachments = new HashMap<String, String>(attachments);
            attachments.remove(Constants.PATH_KEY);
            attachments.remove(Constants.GROUP_KEY);
            attachments.remove(Constants.VERSION_KEY);
            attachments.remove(Constants.DUBBO_VERSION_KEY);
            attachments.remove(Constants.TOKEN_KEY);
            attachments.remove(Constants.TIMEOUT_KEY);
            attachments.remove(Constants.ASYNC_KEY);// Remove async property to avoid being passed to the following invoke chain.
        }
        // 在rpc上下文添加上一个调用链的信息
        RpcContext.getContext()
                .setInvoker(invoker)
                .setInvocation(invocation)
//                .setAttachments(attachments)  // merged from dubbox
                .setLocalAddress(invoker.getUrl().getHost(),
                        invoker.getUrl().getPort());

        // mreged from dubbox
        // we may already added some attachments into RpcContext before this filter (e.g. in rest protocol)
        if (attachments != null) {
            // 把会话域中的附加值全部加入RpcContext中
            if (RpcContext.getContext().getAttachments() != null) {
                RpcContext.getContext().getAttachments().putAll(attachments);
            } else {
                RpcContext.getContext().setAttachments(attachments);
            }
        }

        // 如果会话域属于rpc的会话域，则设置实体域
        if (invocation instanceof RpcInvocation) {
            ((RpcInvocation) invocation).setInvoker(invoker);
        }
        try {
            // 调用下一个调用链
            RpcResult result = (RpcResult) invoker.invoke(invocation);
            // pass attachments to result 把附加值加入到RpcResult
            result.addAttachments(RpcContext.getServerContext().getAttachments());
            return result;
        } finally {
            // 移除本地的上下文
            RpcContext.removeContext();
            // 清空附加值
            RpcContext.getServerContext().clearAttachments();
        }
    }
```

在[《 dubbo源码解析（十九）远程调用——开篇》](https://segmentfault.com/a/1190000017787521)中我已经介绍了RpcContext的作用，角色。该过滤器就是做了初始化RpcContext的作用。

### （七）DeprecatedFilter

该过滤器的作用是调用了废弃的方法时打印错误日志。

```java
private static final Logger LOGGER = LoggerFactory.getLogger(DeprecatedFilter.class);

/**
 * 日志集合
 */
private static final Set<String> logged = new ConcurrentHashSet<String>();

@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 获得key 服务+方法
    String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
    // 如果集合中没有该key
    if (!logged.contains(key)) {
        // 则加入集合
        logged.add(key);
        // 如果该服务方法是废弃的，则打印错误日志
        if (invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.DEPRECATED_KEY, false)) {
            LOGGER.error("The service method " + invoker.getInterface().getName() + "." + getMethodSignature(invocation) + " is DEPRECATED! Declare from " + invoker.getUrl());
        }
    }
    // 调用下一个调用链
    return invoker.invoke(invocation);
}

/**
 * 获得方法定义
 * @param invocation
 * @return
 */
private String getMethodSignature(Invocation invocation) {
    // 方法名
    StringBuilder buf = new StringBuilder(invocation.getMethodName());
    buf.append("(");
    // 参数类型
    Class<?>[] types = invocation.getParameterTypes();
    // 拼接参数
    if (types != null && types.length > 0) {
        boolean first = true;
        for (Class<?> type : types) {
            if (first) {
                first = false;
            } else {
                buf.append(", ");
            }
            buf.append(type.getSimpleName());
        }
    }
    buf.append(")");
    return buf.toString();
}
```

该过滤器比较简单。

### （八）EchoFilter

该过滤器是处理回声测试的方法。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    // 如果调用的方法是回声测试的方法 则直接返回结果，否则 调用下一个调用链
    if (inv.getMethodName().equals(Constants.$ECHO) && inv.getArguments() != null && inv.getArguments().length == 1)
        return new RpcResult(inv.getArguments()[0]);
    return invoker.invoke(inv);
}
```

如果调用的方法是回声测试的方法 则直接返回结果，否则 调用下一个调用链。

### （九）ExceptionFilter

该过滤器是作用是对异常的处理。

```java
private final Logger logger;

public ExceptionFilter() {
    this(LoggerFactory.getLogger(ExceptionFilter.class));
}

public ExceptionFilter(Logger logger) {
    this.logger = logger;
}

@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    try {
        // 调用下一个调用链，返回结果
        Result result = invoker.invoke(invocation);
        // 如果结果有异常，并且该服务不是一个泛化调用
        if (result.hasException() && GenericService.class != invoker.getInterface()) {
            try {
                // 获得异常
                Throwable exception = result.getException();

                // directly throw if it's checked exception
                // 如果这是一个checked的异常，则直接返回异常，也就是接口上声明的Unchecked的异常
                if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                    return result;
                }
                // directly throw if the exception appears in the signature
                // 如果已经在接口方法上声明了该异常，则直接返回
                try {
                    // 获得方法
                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    // 获得异常类型
                    Class<?>[] exceptionClassses = method.getExceptionTypes();
                    for (Class<?> exceptionClass : exceptionClassses) {
                        if (exception.getClass().equals(exceptionClass)) {
                            return result;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    return result;
                }

                // for the exception not found in method's signature, print ERROR message in server's log.
                // 打印错误 该异常没有在方法上申明
                logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                // directly throw if exception class and interface class are in the same jar file.
                // 如果异常类和接口类在同一个jar包里面，则抛出异常
                String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
                String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
                if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                    return result;
                }
                // directly throw if it's JDK exception
                // 如果是jdk中定义的异常，则直接抛出
                String className = exception.getClass().getName();
                if (className.startsWith("java.") || className.startsWith("javax.")) {
                    return result;
                }
                // directly throw if it's dubbo exception
                // 如果 是dubbo的异常，则直接抛出
                if (exception instanceof RpcException) {
                    return result;
                }

                // otherwise, wrap with RuntimeException and throw back to the client
                // 如果不是以上的异常，则包装成为RuntimeException并且抛出
                return new RpcResult(new RuntimeException(StringUtils.toString(exception)));
            } catch (Throwable e) {
                logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                return result;
            }
        }
        return result;
    } catch (RuntimeException e) {
        logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
        throw e;
    }
}
```

可以看到除了接口上声明的Unchecked的异常和有定义的异常外，都会包装成RuntimeException来返回，为了防止客户端反序列化失败。

### （十）ExecuteLimitFilter

该过滤器是限制最大可并行执行请求数，该过滤器是服务提供者侧，而上述讲到的ActiveLimitFilter是在消费者侧的限制。

```java
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 获得url对象
        URL url = invoker.getUrl();
        // 方法名称
        String methodName = invocation.getMethodName();
        Semaphore executesLimit = null;
        boolean acquireResult = false;
        int max = url.getMethodParameter(methodName, Constants.EXECUTES_KEY, 0);
        // 如果该方法设置了executes并且值大于0
        if (max > 0) {
            // 获得该方法对应的RpcStatus
            RpcStatus count = RpcStatus.getStatus(url, invocation.getMethodName());
//            if (count.getActive() >= max) {
            /**
             * http://manzhizhen.iteye.com/blog/2386408
             * use semaphore for concurrency control (to limit thread number)
             */
            // 获得信号量
            executesLimit = count.getSemaphore(max);
            // 如果不能获得许可，则抛出异常
            if(executesLimit != null && !(acquireResult = executesLimit.tryAcquire())) {
                throw new RpcException("Failed to invoke method " + invocation.getMethodName() + " in provider " + url + ", cause: The service using threads greater than <dubbo:service executes=\"" + max + "\" /> limited.");
            }
        }
        long begin = System.currentTimeMillis();
        boolean isSuccess = true;
        // 计数加1
        RpcStatus.beginCount(url, methodName);
        try {
            // 调用下一个调用链
            Result result = invoker.invoke(invocation);
            return result;
        } catch (Throwable t) {
            isSuccess = false;
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new RpcException("unexpected exception when ExecuteLimitFilter", t);
            }
        } finally {
            // 计数减1
            RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, isSuccess);
            if(acquireResult) {
                executesLimit.release();
            }
        }
    }
```

为什么这里需要用到信号量来控制，可以看一下以下链接的介绍：http://manzhizhen.iteye.com/blog/2386408

### （十一）GenericFilter

该过滤器就是对于泛化调用的请求和结果进行反序列化和序列化的操作，它是服务提供者侧的。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    // 如果是泛化调用
    if (inv.getMethodName().equals(Constants.$INVOKE)
            && inv.getArguments() != null
            && inv.getArguments().length == 3
            && !invoker.getInterface().equals(GenericService.class)) {
        // 获得请求名字
        String name = ((String) inv.getArguments()[0]).trim();
        // 获得请求参数类型
        String[] types = (String[]) inv.getArguments()[1];
        // 获得请求参数
        Object[] args = (Object[]) inv.getArguments()[2];
        try {
            // 获得方法
            Method method = ReflectUtils.findMethodByMethodSignature(invoker.getInterface(), name, types);
            // 获得该方法的参数类型
            Class<?>[] params = method.getParameterTypes();
            if (args == null) {
                args = new Object[params.length];
            }
            // 获得附加值
            String generic = inv.getAttachment(Constants.GENERIC_KEY);

            // 如果附加值为空，在用上下文携带的附加值
            if (StringUtils.isBlank(generic)) {
                generic = RpcContext.getContext().getAttachment(Constants.GENERIC_KEY);
            }

            // 如果附加值还是为空或者是默认的泛化序列化类型
            if (StringUtils.isEmpty(generic)
                    || ProtocolUtils.isDefaultGenericSerialization(generic)) {
                // 直接进行类型转化
                args = PojoUtils.realize(args, params, method.getGenericParameterTypes());
            } else if (ProtocolUtils.isJavaGenericSerialization(generic)) {
                for (int i = 0; i < args.length; i++) {
                    if (byte[].class == args[i].getClass()) {
                        try {
                            UnsafeByteArrayInputStream is = new UnsafeByteArrayInputStream((byte[]) args[i]);
                            // 使用nativejava方式反序列化
                            args[i] = ExtensionLoader.getExtensionLoader(Serialization.class)
                                    .getExtension(Constants.GENERIC_SERIALIZATION_NATIVE_JAVA)
                                    .deserialize(null, is).readObject();
                        } catch (Exception e) {
                            throw new RpcException("Deserialize argument [" + (i + 1) + "] failed.", e);
                        }
                    } else {
                        throw new RpcException(
                                "Generic serialization [" +
                                        Constants.GENERIC_SERIALIZATION_NATIVE_JAVA +
                                        "] only support message type " +
                                        byte[].class +
                                        " and your message type is " +
                                        args[i].getClass());
                    }
                }
            } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                for (int i = 0; i < args.length; i++) {
                    if (args[i] instanceof JavaBeanDescriptor) {
                        // 用JavaBean方式反序列化
                        args[i] = JavaBeanSerializeUtil.deserialize((JavaBeanDescriptor) args[i]);
                    } else {
                        throw new RpcException(
                                "Generic serialization [" +
                                        Constants.GENERIC_SERIALIZATION_BEAN +
                                        "] only support message type " +
                                        JavaBeanDescriptor.class.getName() +
                                        " and your message type is " +
                                        args[i].getClass().getName());
                    }
                }
            }
            // 调用下一个调用链
            Result result = invoker.invoke(new RpcInvocation(method, args, inv.getAttachments()));
            if (result.hasException()
                    && !(result.getException() instanceof GenericException)) {
                return new RpcResult(new GenericException(result.getException()));
            }
            if (ProtocolUtils.isJavaGenericSerialization(generic)) {
                try {
                    UnsafeByteArrayOutputStream os = new UnsafeByteArrayOutputStream(512);
                    // 用nativejava方式序列化
                    ExtensionLoader.getExtensionLoader(Serialization.class)
                            .getExtension(Constants.GENERIC_SERIALIZATION_NATIVE_JAVA)
                            .serialize(null, os).writeObject(result.getValue());
                    return new RpcResult(os.toByteArray());
                } catch (IOException e) {
                    throw new RpcException("Serialize result failed.", e);
                }
            } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                // 使用JavaBean方式序列化返回结果
                return new RpcResult(JavaBeanSerializeUtil.serialize(result.getValue(), JavaBeanAccessor.METHOD));
            } else {
                // 直接转化为pojo类型然后返回
                return new RpcResult(PojoUtils.generalize(result.getValue()));
            }
        } catch (NoSuchMethodException e) {
            throw new RpcException(e.getMessage(), e);
        } catch (ClassNotFoundException e) {
            throw new RpcException(e.getMessage(), e);
        }
    }
    // 调用下一个调用链
    return invoker.invoke(inv);
}
```

### （十二）GenericImplFilter

该过滤器也是对于泛化调用的序列化检查和处理，它是消费者侧的过滤器。

```java
private static final Logger logger = LoggerFactory.getLogger(GenericImplFilter.class);

/**
 * 参数集合
 */
private static final Class<?>[] GENERIC_PARAMETER_TYPES = new Class<?>[]{String.class, String[].class, Object[].class};

@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 获得泛化的值
    String generic = invoker.getUrl().getParameter(Constants.GENERIC_KEY);
    // 如果该值是nativejava或者bean或者true，并且不是一个返回调用
    if (ProtocolUtils.isGeneric(generic)
            && !Constants.$INVOKE.equals(invocation.getMethodName())
            && invocation instanceof RpcInvocation) {
        RpcInvocation invocation2 = (RpcInvocation) invocation;
        // 获得方法名称
        String methodName = invocation2.getMethodName();
        // 获得参数类型集合
        Class<?>[] parameterTypes = invocation2.getParameterTypes();
        // 获得参数集合
        Object[] arguments = invocation2.getArguments();

        // 把参数类型的名称放入集合
        String[] types = new String[parameterTypes.length];
        for (int i = 0; i < parameterTypes.length; i++) {
            types[i] = ReflectUtils.getName(parameterTypes[i]);
        }

        Object[] args;
        // 对参数集合进行序列化
        if (ProtocolUtils.isBeanGenericSerialization(generic)) {
            args = new Object[arguments.length];
            for (int i = 0; i < arguments.length; i++) {
                args[i] = JavaBeanSerializeUtil.serialize(arguments[i], JavaBeanAccessor.METHOD);
            }
        } else {
            args = PojoUtils.generalize(arguments);
        }

        // 重新把序列化的参数放入
        invocation2.setMethodName(Constants.$INVOKE);
        invocation2.setParameterTypes(GENERIC_PARAMETER_TYPES);
        invocation2.setArguments(new Object[]{methodName, types, args});
        // 调用下一个调用链
        Result result = invoker.invoke(invocation2);

        if (!result.hasException()) {
            Object value = result.getValue();
            try {
                Method method = invoker.getInterface().getMethod(methodName, parameterTypes);
                if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                    if (value == null) {
                        return new RpcResult(value);
                    } else if (value instanceof JavaBeanDescriptor) {
                        // 用javabean方式反序列化
                        return new RpcResult(JavaBeanSerializeUtil.deserialize((JavaBeanDescriptor) value));
                    } else {
                        throw new RpcException(
                                "The type of result value is " +
                                        value.getClass().getName() +
                                        " other than " +
                                        JavaBeanDescriptor.class.getName() +
                                        ", and the result is " +
                                        value);
                    }
                } else {
                    // 直接转化为pojo类型
                    return new RpcResult(PojoUtils.realize(value, method.getReturnType(), method.getGenericReturnType()));
                }
            } catch (NoSuchMethodException e) {
                throw new RpcException(e.getMessage(), e);
            }
            // 如果调用链中有异常抛出，并且是GenericException类型的异常
        } else if (result.getException() instanceof GenericException) {
            GenericException exception = (GenericException) result.getException();
            try {
                // 获得异常类名
                String className = exception.getExceptionClass();
                Class<?> clazz = ReflectUtils.forName(className);
                Throwable targetException = null;
                Throwable lastException = null;
                try {
                    targetException = (Throwable) clazz.newInstance();
                } catch (Throwable e) {
                    lastException = e;
                    for (Constructor<?> constructor : clazz.getConstructors()) {
                        try {
                            targetException = (Throwable) constructor.newInstance(new Object[constructor.getParameterTypes().length]);
                            break;
                        } catch (Throwable e1) {
                            lastException = e1;
                        }
                    }
                }
                if (targetException != null) {
                    try {
                        Field field = Throwable.class.getDeclaredField("detailMessage");
                        if (!field.isAccessible()) {
                            field.setAccessible(true);
                        }
                        field.set(targetException, exception.getExceptionMessage());
                    } catch (Throwable e) {
                        logger.warn(e.getMessage(), e);
                    }
                    result = new RpcResult(targetException);
                } else if (lastException != null) {
                    throw lastException;
                }
            } catch (Throwable e) {
                throw new RpcException("Can not deserialize exception " + exception.getExceptionClass() + ", message: " + exception.getExceptionMessage(), e);
            }
        }
        return result;
    }

    // 如果是泛化调用
    if (invocation.getMethodName().equals(Constants.$INVOKE)
            && invocation.getArguments() != null
            && invocation.getArguments().length == 3
            && ProtocolUtils.isGeneric(generic)) {

        Object[] args = (Object[]) invocation.getArguments()[2];
        if (ProtocolUtils.isJavaGenericSerialization(generic)) {

            for (Object arg : args) {
                // 如果调用消息不是字节数组类型，则抛出异常
                if (!(byte[].class == arg.getClass())) {
                    error(generic, byte[].class.getName(), arg.getClass().getName());
                }
            }
        } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
            for (Object arg : args) {
                if (!(arg instanceof JavaBeanDescriptor)) {
                    error(generic, JavaBeanDescriptor.class.getName(), arg.getClass().getName());
                }
            }
        }

        // 设置附加值
        ((RpcInvocation) invocation).setAttachment(
                Constants.GENERIC_KEY, invoker.getUrl().getParameter(Constants.GENERIC_KEY));
    }
    return invoker.invoke(invocation);
}

/**
 * 抛出错误异常
 * @param generic
 * @param expected
 * @param actual
 * @throws RpcException
 */
private void error(String generic, String expected, String actual) throws RpcException {
    throw new RpcException(
            "Generic serialization [" +
                    generic +
                    "] only support message type " +
                    expected +
                    " and your message type is " +
                    actual);
}
```

### （十三）TimeoutFilter

该过滤器是当服务调用超时的时候，记录告警日志。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 获得开始时间
    long start = System.currentTimeMillis();
    // 调用下一个调用链
    Result result = invoker.invoke(invocation);
    // 获得调用使用的时间
    long elapsed = System.currentTimeMillis() - start;
    // 如果服务调用超时，则打印告警日志
    if (invoker.getUrl() != null
            && elapsed > invoker.getUrl().getMethodParameter(invocation.getMethodName(),
            "timeout", Integer.MAX_VALUE)) {
        if (logger.isWarnEnabled()) {
            logger.warn("invoke time out. method: " + invocation.getMethodName()
                    + " arguments: " + Arrays.toString(invocation.getArguments()) + " , url is "
                    + invoker.getUrl() + ", invoke elapsed " + elapsed + " ms.");
        }
    }
    return result;
}
```

### （十四）TokenFilter

该过滤器提供了token的验证功能，关于token的介绍可以查看官方文档。

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation inv)
        throws RpcException {
    // 获得token值
    String token = invoker.getUrl().getParameter(Constants.TOKEN_KEY);
    if (ConfigUtils.isNotEmpty(token)) {
        // 获得服务类型
        Class<?> serviceType = invoker.getInterface();
        // 获得附加值
        Map<String, String> attachments = inv.getAttachments();
        String remoteToken = attachments == null ? null : attachments.get(Constants.TOKEN_KEY);
        // 如果令牌不一样，则抛出异常
        if (!token.equals(remoteToken)) {
            throw new RpcException("Invalid token! Forbid invoke remote service " + serviceType + " method " + inv.getMethodName() + "() from consumer " + RpcContext.getContext().getRemoteHost() + " to provider " + RpcContext.getContext().getLocalHost());
        }
    }
    // 调用下一个调用链
    return invoker.invoke(inv);
}
```

### （十五）TpsLimitFilter

该过滤器的作用是对TPS限流。

```java
/**
 * TPS 限制器对象
 */
private final TPSLimiter tpsLimiter = new DefaultTPSLimiter();

@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {

    // 如果限流器不允许，则抛出异常
    if (!tpsLimiter.isAllowable(invoker.getUrl(), invocation)) {
        throw new RpcException(
                "Failed to invoke service " +
                        invoker.getInterface().getName() +
                        "." +
                        invocation.getMethodName() +
                        " because exceed max service tps.");
    }

    // 调用下一个调用链
    return invoker.invoke(invocation);
}
```

其中关键是TPS 限制器对象，请看下面的分析。

### （十六）TPSLimiter

```java
public interface TPSLimiter {

    /**
     * judge if the current invocation is allowed by TPS rule
     * 是否允许通过
     * @param url        url
     * @param invocation invocation
     * @return true allow the current invocation, otherwise, return false
     */
    boolean isAllowable(URL url, Invocation invocation);

}
```

该接口是tps限流器的接口，只定义了一个是否允许通过的方法。

### （十七）StatItem

该类是统计的数据结构。

```java
class StatItem {

    /**
     * 服务名
     */
    private String name;

    /**
     * 最后一次重置的时间
     */
    private long lastResetTime;

    /**
     * 周期
     */
    private long interval;

    /**
     * 剩余多少流量
     */
    private AtomicInteger token;

    /**
     * 限制大小
     */
    private int rate;

    StatItem(String name, int rate, long interval) {
        this.name = name;
        this.rate = rate;
        this.interval = interval;
        this.lastResetTime = System.currentTimeMillis();
        this.token = new AtomicInteger(rate);
    }

    public boolean isAllowable() {
        long now = System.currentTimeMillis();
        // 如果限制的时间大于最后一次时间加上周期，则重置
        if (now > lastResetTime + interval) {
            token.set(rate);
            lastResetTime = now;
        }

        int value = token.get();
        boolean flag = false;
        // 直到有流量
        while (value > 0 && !flag) {
            flag = token.compareAndSet(value, value - 1);
            value = token.get();
        }

        // 返回flag
        return flag;
    }

    long getLastResetTime() {
        return lastResetTime;
    }

    int getToken() {
        return token.get();
    }

    @Override
    public String toString() {
        return new StringBuilder(32).append("StatItem ")
                .append("[name=").append(name).append(", ")
                .append("rate = ").append(rate).append(", ")
                .append("interval = ").append(interval).append("]")
                .toString();
    }

}
```

可以看到该类中记录了一些访问的流量，并且设置了周期重置机制。

### （十八）DefaultTPSLimiter

该类实现了TPSLimiter，是默认的tps限流器实现。

```java
public class DefaultTPSLimiter implements TPSLimiter {

    /**
     * 统计项集合
     */
    private final ConcurrentMap<String, StatItem> stats
            = new ConcurrentHashMap<String, StatItem>();

    @Override
    public boolean isAllowable(URL url, Invocation invocation) {
        // 获得tps限制大小，默认-1，不限制
        int rate = url.getParameter(Constants.TPS_LIMIT_RATE_KEY, -1);
        // 获得限流周期
        long interval = url.getParameter(Constants.TPS_LIMIT_INTERVAL_KEY,
                Constants.DEFAULT_TPS_LIMIT_INTERVAL);
        String serviceKey = url.getServiceKey();
        // 如果限制
        if (rate > 0) {
            // 从集合中获得统计项
            StatItem statItem = stats.get(serviceKey);
            // 如果为空，则新建
            if (statItem == null) {
                stats.putIfAbsent(serviceKey,
                        new StatItem(serviceKey, rate, interval));
                statItem = stats.get(serviceKey);
            }
            // 返回是否允许
            return statItem.isAllowable();
        } else {
            StatItem statItem = stats.get(serviceKey);
            if (statItem != null) {
                // 移除该服务的统计项
                stats.remove(serviceKey);
            }
        }

        return true;
    }

}
```

是否允许的逻辑还是调用了统计项中的isAllowable方法。

本文介绍了很多的过滤器，哪些过滤器是在服务引用的，哪些服务器是服务暴露的，可以查看相应源码过滤器的实现上的注解，

例如ActiveLimitFilter上：

```java
@Activate(group = Constants.CONSUMER, value = Constants.ACTIVES_KEY)
```

可以看到group为consumer组的，也就是服务消费者侧的，则是服务引用过程中的的过滤器。 

例如ExecuteLimitFilter上：

```java
@Activate(group = Constants.PROVIDER, value = Constants.EXECUTES_KEY)
```

可以看到group为provider组的，也就是服务消费者侧的，则是服务暴露过程中的的过滤器。

##  后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/filter

该文章讲解了在服务引用和服务暴露中的各种filter过滤器。接下来我将开始对rpc模块的监听器进行讲解。