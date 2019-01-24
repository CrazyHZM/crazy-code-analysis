# 远程调用——dubbo协议

> 目标：介绍远程调用中跟dubbo协议相关的设计和实现，介绍dubbo-rpc-dubbo的源码。

## 前言

Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。反之，Dubbo 缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。这是官方文档的原话，并且官方文档还介绍了为什么使用单一长连接和 NIO 异步通讯以及为什么不适合传输大数据的服务。我就不赘述了。

我们先来看看dubbo-rpc-dubbo下的包结构：

![dubbo-rpc-dubbo](https://github.com/CrazyHZM/crazy-code-analysis/blob/master/dubbo/image/%E4%BA%8C%E5%8D%81%E5%9B%9B/dubbo-rpc-dubbo.png?raw=true)

1. filter：该包下面是对于dubbo协议独有的两个过滤器
2. status：该包下是做了对于服务和线程池状态的检测
3. telnet：该包下是对于telnet命令的支持
4. 最外层：最外层是dubbo协议的核心

## 源码分析

### （一）DubboInvoker

该类是dubbo协议独自实现的的invoker，其中实现了调用方法的三种模式，分别是异步发送、单向发送和同步发送，具体在下面介绍。

#### 1.属性

```java
/**
 * 信息交换客户端数组
 */
private final ExchangeClient[] clients;

/**
 * 客户端数组位置
 */
private final AtomicPositiveInteger index = new AtomicPositiveInteger();

/**
 * 版本号
 */
private final String version;

/**
 * 销毁锁
 */
private final ReentrantLock destroyLock = new ReentrantLock();

/**
 * Invoker对象集合
 */
private final Set<Invoker<?>> invokers;
```

#### 2.doInvoke

```java
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    // rpc会话域
    RpcInvocation inv = (RpcInvocation) invocation;
    // 获得方法名
    final String methodName = RpcUtils.getMethodName(invocation);
    // 把path放入到附加值中
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    // 把版本号放入到附加值
    inv.setAttachment(Constants.VERSION_KEY, version);

    // 当前的客户端
    ExchangeClient currentClient;
    // 如果数组内就一个客户端，则直接取出
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        // 取模轮询 从数组中取，当取到最后一个时，从头开始
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        // 是否启用异步
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        // 是否是单向发送
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        // 获得超时时间
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        // 如果是单项发送
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            // 单向发送只负责发送消息，不等待服务端应答，所以没有返回值
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {
            // 异步调用
            ResponseFuture future = currentClient.request(inv, timeout);
            // 保存future，方便后期处理
            RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
            return new RpcResult();
        } else {
            // 同步调用，等待返回结果
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

在调用invoker的时候，通过远程通信将Invocation信息传递给服务端，服务端在接收到该invocation信息后，要找到对应的本地方法，然后通过反射执行该方法，将方法的执行结果返回给客户端，在这里，客户端发送有三种模式：

1. 异步发送，也就是当我发送调用后，我不阻塞等待结果，直接返回，将返回的future保存到上下文，方便后期使用。
2. 单向发送，执行方法不需要返回结果。
3. 同步发送，执行方法后，等待结果返回，否则一直阻塞。

#### 3.isAvailable

```java
@Override
public boolean isAvailable() {
    if (!super.isAvailable())
        return false;
    for (ExchangeClient client : clients) {
        // 只要有一个客户端连接并且不是只读，则表示存活
        if (client.isConnected() && !client.hasAttribute(Constants.CHANNEL_ATTRIBUTE_READONLY_KEY)) {
            //cannot write == not Available ?
            return true;
        }
    }
    return false;
}
```

该方法是检查服务端是否存活。

#### 4.destroy

```java
@Override
public void destroy() {
    // in order to avoid closing a client multiple times, a counter is used in case of connection per jvm, every
    // time when client.close() is called, counter counts down once, and when counter reaches zero, client will be
    // closed.
    if (super.isDestroyed()) {
        return;
    } else {
        // double check to avoid dup close
        // 获得销毁锁
        destroyLock.lock();
        try {
            if (super.isDestroyed()) {
                return;
            }
            // 销毁
            super.destroy();
            // 从集合中移除
            if (invokers != null) {
                invokers.remove(this);
            }
            for (ExchangeClient client : clients) {
                try {
                    // 关闭每一个客户端
                    client.close(ConfigUtils.getServerShutdownTimeout());
                } catch (Throwable t) {
                    logger.warn(t.getMessage(), t);
                }
            }

        } finally {
            // 释放锁
            destroyLock.unlock();
        }
    }
}
```

该方法是销毁服务端，关闭所有连接到远程通信客户端。

### （二）DubboExporter

该类继承了AbstractExporter，是dubbo协议中独有的服务暴露者。

```java
/**
 * 服务key
 */
private final String key;

/**
 * 服务暴露者集合
 */
private final Map<String, Exporter<?>> exporterMap;

public DubboExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
    super(invoker);
    this.key = key;
    this.exporterMap = exporterMap;
}

@Override
public void unexport() {
    super.unexport();
    // 从集合中移除该key
    exporterMap.remove(key);
}
```

其中对于服务暴露者用集合做了缓存，并且只重写了了unexport。

### （三）DubboProtocol

该类是dubbo协议的核心实现，其中增加了比如延迟加载等处理。 并且其中还包括了对服务暴露和服务引用的逻辑处理。

#### 1.属性

```java
public static final String NAME = "dubbo";

/**
 * 默认端口号
 */
public static final int DEFAULT_PORT = 20880;
/**
 * 回调名称
 */
private static final String IS_CALLBACK_SERVICE_INVOKE = "_isCallBackServiceInvoke";
/**
 * dubbo协议的单例
 */
private static DubboProtocol INSTANCE;
/**
 * 信息交换服务器集合 key：host:port  value：ExchangeServer
 */
private final Map<String, ExchangeServer> serverMap = new ConcurrentHashMap<String, ExchangeServer>(); // <host:port,Exchanger>
/**
 * 信息交换客户端集合
 */
private final Map<String, ReferenceCountExchangeClient> referenceClientMap = new ConcurrentHashMap<String, ReferenceCountExchangeClient>(); // <host:port,Exchanger>
/**
 * 懒加载的客户端集合
 */
private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap = new ConcurrentHashMap<String, LazyConnectExchangeClient>();
/**
 * 锁集合
 */
private final ConcurrentMap<String, Object> locks = new ConcurrentHashMap<String, Object>();
/**
 * 序列化类名集合
 */
private final Set<String> optimizers = new ConcurrentHashSet<String>();
//consumer side export a stub service for dispatching event
//servicekey-stubmethods
/**
 * 本地存根服务方法集合
 */
private final ConcurrentMap<String, String> stubServiceMethodsMap = new ConcurrentHashMap<String, String>();
/**
 * 新建一个请求处理器
 */
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
    /**
     * 回复请求结果，返回的是请求结果
     * @param channel
     * @param message
     * @return
     * @throws RemotingException
     */
    @Override
    public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
        // 如果请求消息属于会话域
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            // 获得暴露的invoker
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            // 如果是回调服务
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                // 获得 方法定义
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                // 判断看是否有会话域中的方法
                if (methodsStr == null || methodsStr.indexOf(",") == -1) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    // 如果方法不止一个，则分割后遍历查询，找到了则设置为true
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                // 如果没有该方法，则打印告警日志
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            // 设置远程地址
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            // 调用下一个调用链
            return invoker.invoke(inv);
        }
        // 否则抛出异常
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }

    /**
     * 接收消息
     * @param channel
     * @param message
     * @throws RemotingException
     */
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        // 如果消息是会话域中的消息，则调用reply方法。
        if (message instanceof Invocation) {
            reply((ExchangeChannel) channel, message);
        } else {
            super.received(channel, message);
        }
    }

    @Override
    public void connected(Channel channel) throws RemotingException {
        // 接收连接事件
        invoke(channel, Constants.ON_CONNECT_KEY);
    }

    @Override
    public void disconnected(Channel channel) throws RemotingException {
        if (logger.isInfoEnabled()) {
            logger.info("disconnected from " + channel.getRemoteAddress() + ",url:" + channel.getUrl());
        }
        // 接收断开连接事件
        invoke(channel, Constants.ON_DISCONNECT_KEY);
    }

    /**
     * 接收事件
     * @param channel
     * @param methodKey
     */
    private void invoke(Channel channel, String methodKey) {
        // 创建会话域
        Invocation invocation = createInvocation(channel, channel.getUrl(), methodKey);
        if (invocation != null) {
            try {
                // 接收事件
                received(channel, invocation);
            } catch (Throwable t) {
                logger.warn("Failed to invoke event method " + invocation.getMethodName() + "(), cause: " + t.getMessage(), t);
            }
        }
    }

    /**
     * 创建会话域, 把url内的值加入到会话域的附加值中
     * @param channel
     * @param url
     * @param methodKey
     * @return
     */
    private Invocation createInvocation(Channel channel, URL url, String methodKey) {
        // 获得方法，methodKey是onconnect或者ondisconnect
        String method = url.getParameter(methodKey);
        if (method == null || method.length() == 0) {
            return null;
        }
        // 创建一个rpc会话域
        RpcInvocation invocation = new RpcInvocation(method, new Class<?>[0], new Object[0]);
        // 加入附加值path
        invocation.setAttachment(Constants.PATH_KEY, url.getPath());
        // 加入附加值group
        invocation.setAttachment(Constants.GROUP_KEY, url.getParameter(Constants.GROUP_KEY));
        // 加入附加值interface
        invocation.setAttachment(Constants.INTERFACE_KEY, url.getParameter(Constants.INTERFACE_KEY));
        // 加入附加值version
        invocation.setAttachment(Constants.VERSION_KEY, url.getParameter(Constants.VERSION_KEY));
        // 如果是本地存根服务，则加入附加值dubbo.stub.event为true
        if (url.getParameter(Constants.STUB_EVENT_KEY, false)) {
            invocation.setAttachment(Constants.STUB_EVENT_KEY, Boolean.TRUE.toString());
        }
        return invocation;
    }
};
```

该属性中关键的是实例化了一个请求处理器，其中实现了基于dubbo协议等连接、取消连接、回复请求结果等方法。

#### 2.export

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    // 得到服务key  group+"/"+serviceName+":"+serviceVersion+":"+port
    String key = serviceKey(url);
    // 创建exporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 加入到集合
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    // 如果是本地存根事件而不是回调服务
    if (isStubSupportEvent && !isCallbackservice) {
        // 获得本地存根的方法
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        // 如果为空，则抛出异常
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            // 加入集合
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }


    // 打开服务
    openServer(url);
    // 序列化
    optimizeSerialization(url);
    return exporter;
}
```

该方法是基于dubbo协议的服务暴露，除了对于存根服务和本地服务进行标记以外，打开服务和序列化分别在openServer和optimizeSerialization中实现。

#### 3.openServer

```java
private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client can export a service which's only for server to invoke
    // 客户端是否可以暴露仅供服务器调用的服务
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    // 如果是的话
    if (isServer) {
        // 获得信息交换服务器
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            // 重新创建服务器对象，然后放入集合
            serverMap.put(key, createServer(url));
        } else {
            // server supports reset, use together with override
            // 重置
            server.reset(url);
        }
    }
}
```

该方法就是打开服务。其中的逻辑其实是把服务对象放入集合中进行缓存，如果该地址对应的服务器不存在，则调用createServer创建一个服务器对象。

#### 4.createServer

```java
private ExchangeServer createServer(URL url) {
    // send readonly event when server closes, it's enabled by default
    // 服务器关闭时发送readonly事件，默认情况下启用
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // enable heartbeat by default
    // 心跳默认间隔一分钟
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    // 获得远程通讯服务端实现方式，默认用netty3
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    /**
     * 如果没有该配置，则抛出异常
     */
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    /**
     * 添加编解码器DubboCodec实现
     */
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        // 启动服务器
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    // 获得客户端侧设置的远程通信方式
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        // 获得远程通信的实现集合
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        // 如果客户端侧设置的远程通信方式不在支持的方式中，则抛出异常
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}

```

该方法就是根据url携带的远程通信实现方法来创建一个服务器对象。

#### 5.optimizeSerialization

```java
private void optimizeSerialization(URL url) throws RpcException {
    // 获得类名
    String className = url.getParameter(Constants.OPTIMIZER_KEY, "");
    if (StringUtils.isEmpty(className) || optimizers.contains(className)) {
        return;
    }

    logger.info("Optimizing the serialization process for Kryo, FST, etc...");

    try {
        // 加载类
        Class clazz = Thread.currentThread().getContextClassLoader().loadClass(className);
        if (!SerializationOptimizer.class.isAssignableFrom(clazz)) {
            throw new RpcException("The serialization optimizer " + className + " isn't an instance of " + SerializationOptimizer.class.getName());
        }

        // 强制类型转化为SerializationOptimizer
        SerializationOptimizer optimizer = (SerializationOptimizer) clazz.newInstance();

        if (optimizer.getSerializableClasses() == null) {
            return;
        }

        // 遍历序列化的类，把该类放入到集合进行缓存
        for (Class c : optimizer.getSerializableClasses()) {
            SerializableClassRegistry.registerClass(c);
        }

        // 加入到集合
        optimizers.add(className);
    } catch (ClassNotFoundException e) {
        throw new RpcException("Cannot find the serialization optimizer class: " + className, e);
    } catch (InstantiationException e) {
        throw new RpcException("Cannot instantiate the serialization optimizer class: " + className, e);
    } catch (IllegalAccessException e) {
        throw new RpcException("Cannot instantiate the serialization optimizer class: " + className, e);
    }
}

```

该方法是把序列化的类放入到集合，以便进行序列化

#### 6.refer

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    // 序列化
    optimizeSerialization(url);
    // create rpc invoker. 创建一个DubboInvoker对象
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    // 把该invoker放入集合
    invokers.add(invoker);
    return invoker;
}

```

该方法是服务引用，其中就是新建一个DubboInvoker对象后把它放入到集合。

#### 7.getClients

```java
private ExchangeClient[] getClients(URL url) {
    // whether to share connection
    // 一个连接是否对于一个服务
    boolean service_share_connect = false;
    // 获得url中欢愉连接共享的配置 默认为0
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // if not configured, connection is shared, otherwise, one connection for one service
    // 如果为0，则是共享类，并且连接数为1
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }

    // 创建数组
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        // 如果共享，则获得共享客户端对象，否则新建客户端
        if (service_share_connect) {
            clients[i] = getSharedClient(url);
        } else {
            clients[i] = initClient(url);
        }
    }
    return clients;
}

```

该方法是获得客户端集合的方法，分为共享客户端和非共享客户端。共享客户端是共用同一个连接，非共享客户端是每个客户端都有自己的一个连接。

#### 8.getSharedClient

```java
private ExchangeClient getSharedClient(URL url) {
    String key = url.getAddress();
    // 从集合中取出客户端对象
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    // 如果不为空并且没关闭连接，则计数器加1，返回
    if (client != null) {
        if (!client.isClosed()) {
            client.incrementAndGetCount();
            return client;
        } else {
            // 如果连接断开，则从集合中移除
            referenceClientMap.remove(key);
        }
    }

    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        // 如果集合中有该key
        if (referenceClientMap.containsKey(key)) {
            // 则直接返回client
            return referenceClientMap.get(key);
        }

        // 否则新建一个连接
        ExchangeClient exchangeClient = initClient(url);
        client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
        // 存入集合
        referenceClientMap.put(key, client);
        // 从ghostClientMap中移除
        ghostClientMap.remove(key);
        // 从对象锁中移除
        locks.remove(key);
        return client;
    }
}

```

该方法是获得分享的客户端连接。

#### 9.initClient

```java
private ExchangeClient initClient(URL url) {

    // client type setting.
    // 获得客户端的实现方法 默认netty3
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

    // 添加编码器
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    // enable heartbeat by default
    // 默认开启心跳
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

    // BIO is not allowed since it has severe performance issue.
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: " + str + "," +
                " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
    }

    ExchangeClient client;
    try {
        // connection should be lazy
        // 是否需要延迟连接，，默认不开启
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            // 创建延迟连接的客户端
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            // 否则就直接连接
            client = Exchangers.connect(url, requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
    }
    return client;
}

```

该方法是新建一个客户端连接

#### 10.destroy

```java
@Override
public void destroy() {
    // 遍历服务器逐个关闭
    for (String key : new ArrayList<String>(serverMap.keySet())) {
        ExchangeServer server = serverMap.remove(key);
        if (server != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo server: " + server.getLocalAddress());
                }
                server.close(ConfigUtils.getServerShutdownTimeout());
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }

    // 遍历客户端集合逐个关闭
    for (String key : new ArrayList<String>(referenceClientMap.keySet())) {
        ExchangeClient client = referenceClientMap.remove(key);
        if (client != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo connect: " + client.getLocalAddress() + "-->" + client.getRemoteAddress());
                }
                client.close(ConfigUtils.getServerShutdownTimeout());
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }

    // 遍历懒加载的集合，逐个关闭客户端
    for (String key : new ArrayList<String>(ghostClientMap.keySet())) {
        ExchangeClient client = ghostClientMap.remove(key);
        if (client != null) {
            try {
                if (logger.isInfoEnabled()) {
                    logger.info("Close dubbo connect: " + client.getLocalAddress() + "-->" + client.getRemoteAddress());
                }
                client.close(ConfigUtils.getServerShutdownTimeout());
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    }
    stubServiceMethodsMap.clear();
    super.destroy();
}

```

该方法是销毁的方法重写。

### （四）ChannelWrappedInvoker

该类是对当前通道内的客户端调用消息进行包装

#### 1.属性

```java
/**
 * 通道
 */
private final Channel channel;
/**
 * 服务key
 */
private final String serviceKey;
/**
 * 当前的客户端
 */
private final ExchangeClient currentClient;

```

#### 2.doInvoke

```java
@Override
protected Result doInvoke(Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    // use interface's name as service path to export if it's not found on client side
    // 设置服务path，默认用接口名称
    inv.setAttachment(Constants.PATH_KEY, getInterface().getName());
    // 设置回调的服务key
    inv.setAttachment(Constants.CALLBACK_SERVICE_KEY, serviceKey);

    try {
        // 如果是异步的
        if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) { // may have concurrency issue
            // 直接发送请求消息
            currentClient.send(inv, getUrl().getMethodParameter(invocation.getMethodName(), Constants.SENT_KEY, false));
            return new RpcResult();
        }
        // 获得超时时间
        int timeout = getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (timeout > 0) {
            return (Result) currentClient.request(inv, timeout).get();
        } else {
            return (Result) currentClient.request(inv).get();
        }
    } catch (RpcException e) {
        throw e;
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, e.getMessage(), e);
    } catch (Throwable e) { // here is non-biz exception, wrap it.
        throw new RpcException(e.getMessage(), e);
    }
}

```

该方法是在invoker调用的时候对发送请求消息进行了包装。

#### 3.ChannelWrapper

该类是个内部没，继承了ClientDelegate，其中将编码器变成了dubbo的编码器，其他方法比较简单。

### （五）DecodeableRpcInvocation

该类主要做了对于会话域内的数据进行序列化和解码。

#### 1.属性

```java
private static final Logger log = LoggerFactory.getLogger(DecodeableRpcInvocation.class);

/**
 * 通道
 */
private Channel channel;

/**
 * 序列化类型
 */
private byte serializationType;

/**
 * 输入流
 */
private InputStream inputStream;

/**
 * 请求
 */
private Request request;

/**
 * 是否解码
 */
private volatile boolean hasDecoded;

```

#### 2.decode

```java
@Override
public void decode() throws Exception {
    // 如果没有解码，则进行解码
    if (!hasDecoded && channel != null && inputStream != null) {
        try {
            decode(channel, inputStream);
        } catch (Throwable e) {
            if (log.isWarnEnabled()) {
                log.warn("Decode rpc invocation failed: " + e.getMessage(), e);
            }
            request.setBroken(true);
            request.setData(e);
        } finally {
            // 设置已经解码
            hasDecoded = true;
        }
    }
}

@Override
public Object decode(Channel channel, InputStream input) throws IOException {
    // 对数据进行反序列化
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
            .deserialize(channel.getUrl(), input);

    // dubbo版本
    String dubboVersion = in.readUTF();
    // 请求中放入dubbo版本
    request.setVersion(dubboVersion);
    // 附加值内加入dubbo版本，path以及版本号
    setAttachment(Constants.DUBBO_VERSION_KEY, dubboVersion);

    setAttachment(Constants.PATH_KEY, in.readUTF());
    setAttachment(Constants.VERSION_KEY, in.readUTF());

    // 设置方法名称
    setMethodName(in.readUTF());
    try {
        // 方法参数数组
        Object[] args;
        // 方法参数类型数组
        Class<?>[] pts;
        // 描述
        String desc = in.readUTF();
        // 如果为空，则方法参数数组和对方法参数类型数组都设置为空
        if (desc.length() == 0) {
            pts = DubboCodec.EMPTY_CLASS_ARRAY;
            args = DubboCodec.EMPTY_OBJECT_ARRAY;
        } else {
            // 分割成类，获得类数组
            pts = ReflectUtils.desc2classArray(desc);
            // 创建等长等数组
            args = new Object[pts.length];
            for (int i = 0; i < args.length; i++) {
                try {
                    // 读取对象放入数组中
                    args[i] = in.readObject(pts[i]);
                } catch (Exception e) {
                    if (log.isWarnEnabled()) {
                        log.warn("Decode argument failed: " + e.getMessage(), e);
                    }
                }
            }
        }
        // 设置参数类型
        setParameterTypes(pts);

        Map<String, String> map = (Map<String, String>) in.readObject(Map.class);
        if (map != null && map.size() > 0) {
            // 获得所有附加值
            Map<String, String> attachment = getAttachments();
            if (attachment == null) {
                attachment = new HashMap<String, String>();
            }
            // 把流中读到的配置放入附加值
            attachment.putAll(map);
            // 放回去
            setAttachments(attachment);
        }
        //decode argument ,may be callback
        for (int i = 0; i < args.length; i++) {
            // 如果是回调，则再一次解码
            args[i] = decodeInvocationArgument(channel, this, pts, i, args[i]);
        }

        setArguments(args);

    } catch (ClassNotFoundException e) {
        throw new IOException(StringUtils.toString("Read invocation data failed.", e));
    } finally {
        if (in instanceof Cleanable) {
            ((Cleanable) in).cleanup();
        }
    }
    return this;
}

```

该方法就是处理Invocation内数据的逻辑，其中主要是做了序列化和解码。把读取出来的设置放入对对应位置传递给后面的调用。

### （六）DecodeableRpcResult

该类是做了基于dubbo协议对prc结果的解码

#### 1.属性

```java
private static final Logger log = LoggerFactory.getLogger(DecodeableRpcResult.class);

/**
 * 通道
 */
private Channel channel;

/**
 * 序列化类型
 */
private byte serializationType;

/**
 * 输入流
 */
private InputStream inputStream;

/**
 * 响应
 */
private Response response;

/**
 * 会话域
 */
private Invocation invocation;

/**
 * 是否解码
 */
private volatile boolean hasDecoded;

```

#### 2.decode

```java
@Override
public Object decode(Channel channel, InputStream input) throws IOException {
    // 反序列化
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
            .deserialize(channel.getUrl(), input);
    
    byte flag = in.readByte();
    // 根据返回的不同结果来进行处理
    switch (flag) {
        case DubboCodec.RESPONSE_NULL_VALUE:
            // 返回结果为空
            break;
        case DubboCodec.RESPONSE_VALUE:
            //
            try {
                // 获得返回类型数组
                Type[] returnType = RpcUtils.getReturnTypes(invocation);
                // 根据返回类型读取返回结果并且放入RpcResult
                setValue(returnType == null || returnType.length == 0 ? in.readObject() :
                        (returnType.length == 1 ? in.readObject((Class<?>) returnType[0])
                                : in.readObject((Class<?>) returnType[0], returnType[1])));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        case DubboCodec.RESPONSE_WITH_EXCEPTION:
            // 返回结果有异常
            try {
                Object obj = in.readObject();
                // 把异常放入RpcResult
                if (obj instanceof Throwable == false)
                    throw new IOException("Response data error, expect Throwable, but get " + obj);
                setException((Throwable) obj);
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        case DubboCodec.RESPONSE_NULL_VALUE_WITH_ATTACHMENTS:
            // 返回值为空，但是有附加值
            try {
                // 把附加值加入到RpcResult
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        case DubboCodec.RESPONSE_VALUE_WITH_ATTACHMENTS:
            // 返回值
            try {
                // 设置返回结果
                Type[] returnType = RpcUtils.getReturnTypes(invocation);
                setValue(returnType == null || returnType.length == 0 ? in.readObject() :
                        (returnType.length == 1 ? in.readObject((Class<?>) returnType[0])
                                : in.readObject((Class<?>) returnType[0], returnType[1])));
                // 设置附加值
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        case DubboCodec.RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS:
            // 返回结果有异常并且有附加值
            try {
                // 设置异常
                Object obj = in.readObject();
                if (obj instanceof Throwable == false)
                    throw new IOException("Response data error, expect Throwable, but get " + obj);
                setException((Throwable) obj);
                // 设置附加值
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        default:
            throw new IOException("Unknown result flag, expect '0' '1' '2', get " + flag);
    }
    if (in instanceof Cleanable) {
        ((Cleanable) in).cleanup();
    }
    return this;
}

@Override
public void decode() throws Exception {
    // 如果没有解码
    if (!hasDecoded && channel != null && inputStream != null) {
        try {
            // 进行解码
            decode(channel, inputStream);
        } catch (Throwable e) {
            if (log.isWarnEnabled()) {
                log.warn("Decode rpc result failed: " + e.getMessage(), e);
            }
            response.setStatus(Response.CLIENT_ERROR);
            response.setErrorMessage(StringUtils.toString(e));
        } finally {
            hasDecoded = true;
        }
    }
}

```

该方法是对响应结果的解码，其中根据不同的返回结果来对RpcResult设置不同的值。

### （七）LazyConnectExchangeClient

该类实现了ExchangeClient接口，是ExchangeClient的装饰器，用到了装饰模式，是延迟连接的客户端实现类。

#### 1.属性

```java
// when this warning rises from invocation, program probably have bug.
/**
 * 延迟连接请求错误key
 */
static final String REQUEST_WITH_WARNING_KEY = "lazyclient_request_with_warning";
private final static Logger logger = LoggerFactory.getLogger(LazyConnectExchangeClient.class);
/**
 * 是否在延迟连接请求时错误
 */
protected final boolean requestWithWarning;
/**
 * url对象
 */
private final URL url;
/**
 * 请求处理器
 */
private final ExchangeHandler requestHandler;
/**
 * 连接锁
 */
private final Lock connectLock = new ReentrantLock();
// lazy connect, initial state for connection
/**
 * 初始化状态
 */
private final boolean initialState;
/**
 * 客户端对象
 */
private volatile ExchangeClient client;
/**
 * 错误次数
 */
private AtomicLong warningcount = new AtomicLong(0);

```

可以看到有属性ExchangeClient client，该类中很多方法就直接调用了client的方法。

#### 2.构造方法

```java
public LazyConnectExchangeClient(URL url, ExchangeHandler requestHandler) {
    // lazy connect, need set send.reconnect = true, to avoid channel bad status.
    // 默认有重连
    this.url = url.addParameter(Constants.SEND_RECONNECT_KEY, Boolean.TRUE.toString());
    this.requestHandler = requestHandler;
    // 默认延迟连接初始化成功
    this.initialState = url.getParameter(Constants.LAZY_CONNECT_INITIAL_STATE_KEY, Constants.DEFAULT_LAZY_CONNECT_INITIAL_STATE);
    // 默认没有错误
    this.requestWithWarning = url.getParameter(REQUEST_WITH_WARNING_KEY, false);
}

```

#### 3.initClient

```java
private void initClient() throws RemotingException {
    // 如果客户端已经初始化，则直接返回
    if (client != null)
        return;
    if (logger.isInfoEnabled()) {
        logger.info("Lazy connect to " + url);
    }
    // 获得连接锁
    connectLock.lock();
    try {
        // 二次判空
        if (client != null)
            return;
        // 新建一个客户端
        this.client = Exchangers.connect(url, requestHandler);
    } finally {
        // 释放锁
        connectLock.unlock();
    }
}

```

该方法是初始化客户端的方法。

#### 4.request

```java
@Override
public ResponseFuture request(Object request) throws RemotingException {
    warning(request);
    initClient();
    return client.request(request);
}

```

该方法在调用client.request前调用了前面两个方法，initClient我在上面讲到了，就是用来初始化客户端的。而warning是用来报错的。

#### 5.warning

```java
private void warning(Object request) {
    if (requestWithWarning) {
        // 每5000次报错一次
        if (warningcount.get() % 5000 == 0) {
            logger.warn(new IllegalStateException("safe guard client , should not be called ,must have a bug."));
        }
        warningcount.incrementAndGet();
    }
}

```

每5000次记录报错一次。

### （八）ReferenceCountExchangeClient

该类也是对ExchangeClient的装饰，其中增强了调用次数多功能。

#### 1.属性

```java
/**
 * url对象
 */
private final URL url;
/**
 * 计数
 */
private final AtomicInteger refenceCount = new AtomicInteger(0);

//    private final ExchangeHandler handler;
/**
 * 延迟连接客户端集合
 */
private final ConcurrentMap<String, LazyConnectExchangeClient> ghostClientMap;
/**
 * 客户端对象
 */
private ExchangeClient client;

```

#### 2.replaceWithLazyClient

```java
// ghost client
private LazyConnectExchangeClient replaceWithLazyClient() {
    // this is a defensive operation to avoid client is closed by accident, the initial state of the client is false
    // 设置延迟连接初始化状态、是否重连、是否已经重连等配置
    URL lazyUrl = url.addParameter(Constants.LAZY_CONNECT_INITIAL_STATE_KEY, Boolean.FALSE)
            .addParameter(Constants.RECONNECT_KEY, Boolean.FALSE)
            .addParameter(Constants.SEND_RECONNECT_KEY, Boolean.TRUE.toString())
            .addParameter("warning", Boolean.TRUE.toString())
            .addParameter(LazyConnectExchangeClient.REQUEST_WITH_WARNING_KEY, true)
            .addParameter("_client_memo", "referencecounthandler.replacewithlazyclient");

    // 获得服务地址
    String key = url.getAddress();
    // in worst case there's only one ghost connection.
    // 从集合中获取客户端
    LazyConnectExchangeClient gclient = ghostClientMap.get(key);
    // 如果对应等客户端不存在或者已经关闭连接，则重新创建一个延迟连接等客户端，并且放入集合
    if (gclient == null || gclient.isClosed()) {
        gclient = new LazyConnectExchangeClient(lazyUrl, client.getExchangeHandler());
        ghostClientMap.put(key, gclient);
    }
    return gclient;
}

```

该方法是用延迟连接替代，该方法在close方法中被调用。

#### 3.close

```java
@Override
public void close(int timeout) {
    if (refenceCount.decrementAndGet() <= 0) {
        if (timeout == 0) {
            client.close();
        } else {
            client.close(timeout);
        }
        client = replaceWithLazyClient();
    }
}

```

### （九）FutureAdapter

该类实现了Future接口，是响应的Future适配器。其中是基于ResponseFuture做适配。其中比较简单，我就不多讲解了。

### （十）CallbackServiceCodec

该类是针对回调服务的编解码器。

#### 1.属性

```java
/**
 * 代理工厂
 */
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
/**
 * dubbo协议
 */
private static final DubboProtocol protocol = DubboProtocol.getDubboProtocol();
/**
 * 回调的标志
 */
private static final byte CALLBACK_NONE = 0x0;
/**
 * 回调的创建标志
 */
private static final byte CALLBACK_CREATE = 0x1;
/**
 * 回调的销毁标志
 */
private static final byte CALLBACK_DESTROY = 0x2;
/**
 * 回调参数key
 */
private static final String INV_ATT_CALLBACK_KEY = "sys_callback_arg-";

```

#### 2.encodeInvocationArgument

```java
public static Object encodeInvocationArgument(Channel channel, RpcInvocation inv, int paraIndex) throws IOException {
    // get URL directly
    // 直接获得url
    URL url = inv.getInvoker() == null ? null : inv.getInvoker().getUrl();
    // 设置回调标志
    byte callbackstatus = isCallBack(url, inv.getMethodName(), paraIndex);
    // 获得参数集合
    Object[] args = inv.getArguments();
    // 获得参数类型集合
    Class<?>[] pts = inv.getParameterTypes();
    // 根据不同的回调状态来设置附加值和返回参数
    switch (callbackstatus) {
        case CallbackServiceCodec.CALLBACK_NONE:
            return args[paraIndex];
        case CallbackServiceCodec.CALLBACK_CREATE:
            inv.setAttachment(INV_ATT_CALLBACK_KEY + paraIndex, exportOrunexportCallbackService(channel, url, pts[paraIndex], args[paraIndex], true));
            return null;
        case CallbackServiceCodec.CALLBACK_DESTROY:
            inv.setAttachment(INV_ATT_CALLBACK_KEY + paraIndex, exportOrunexportCallbackService(channel, url, pts[paraIndex], args[paraIndex], false));
            return null;
        default:
            return args[paraIndex];
    }
}

```

该方法是对会话域的信息进行编码。

#### 3.decodeInvocationArgument

```java
public static Object decodeInvocationArgument(Channel channel, RpcInvocation inv, Class<?>[] pts, int paraIndex, Object inObject) throws IOException {
    // if it's a callback, create proxy on client side, callback interface on client side can be invoked through channel
    // need get URL from channel and env when decode
    URL url = null;
    try {
        // 获得url
        url = DubboProtocol.getDubboProtocol().getInvoker(channel, inv).getUrl();
    } catch (RemotingException e) {
        if (logger.isInfoEnabled()) {
            logger.info(e.getMessage(), e);
        }
        return inObject;
    }
    // 获得回调状态
    byte callbackstatus = isCallBack(url, inv.getMethodName(), paraIndex);
    // 根据回调状态来返回结果
    switch (callbackstatus) {
        case CallbackServiceCodec.CALLBACK_NONE:
            return inObject;
        case CallbackServiceCodec.CALLBACK_CREATE:
            try {
                return referOrdestroyCallbackService(channel, url, pts[paraIndex], inv, Integer.parseInt(inv.getAttachment(INV_ATT_CALLBACK_KEY + paraIndex)), true);
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                throw new IOException(StringUtils.toString(e));
            }
        case CallbackServiceCodec.CALLBACK_DESTROY:
            try {
                return referOrdestroyCallbackService(channel, url, pts[paraIndex], inv, Integer.parseInt(inv.getAttachment(INV_ATT_CALLBACK_KEY + paraIndex)), false);
            } catch (Exception e) {
                throw new IOException(StringUtils.toString(e));
            }
        default:
            return inObject;
    }
}

```

该方法是对会话域内的信息进行解码。

#### 4.isCallBack

```java
private static byte isCallBack(URL url, String methodName, int argIndex) {
    // parameter callback rule: method-name.parameter-index(starting from 0).callback
    // 参数的规则：ethod-name.parameter-index(starting from 0).callback
    byte isCallback = CALLBACK_NONE;
    if (url != null) {
        // 获得回调的值
        String callback = url.getParameter(methodName + "." + argIndex + ".callback");
        if (callback != null) {
            // 如果为true，则设置为创建标志
            if (callback.equalsIgnoreCase("true")) {
                isCallback = CALLBACK_CREATE;
                // 如果为false，则设置为销毁标志
            } else if (callback.equalsIgnoreCase("false")) {
                isCallback = CALLBACK_DESTROY;
            }
        }
    }
    return isCallback;
}

```

该方法是根据url携带的参数设置回调的标志，以供执行不同的编解码逻辑。

#### 5.exportOrunexportCallbackService

```java
private static String exportOrunexportCallbackService(Channel channel, URL url, Class clazz, Object inst, Boolean export) throws IOException {
    // 返回对象的hashCode
    int instid = System.identityHashCode(inst);

    Map<String, String> params = new HashMap<String, String>(3);
    // no need to new client again
    // 设置不是服务端标志为否
    params.put(Constants.IS_SERVER_KEY, Boolean.FALSE.toString());
    // mark it's a callback, for troubleshooting
    // 设置是回调服务标志为true
    params.put(Constants.IS_CALLBACK_SERVICE, Boolean.TRUE.toString());
    String group = url.getParameter(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        // 设置是消费侧还是提供侧
        params.put(Constants.GROUP_KEY, group);
    }
    // add method, for verifying against method, automatic fallback (see dubbo protocol)
    // 添加方法，在dubbo的协议里面用到
    params.put(Constants.METHODS_KEY, StringUtils.join(Wrapper.getWrapper(clazz).getDeclaredMethodNames(), ","));

    Map<String, String> tmpmap = new HashMap<String, String>(url.getParameters());
    tmpmap.putAll(params);
    // 移除版本信息
    tmpmap.remove(Constants.VERSION_KEY);// doesn't need to distinguish version for callback
    // 设置接口名
    tmpmap.put(Constants.INTERFACE_KEY, clazz.getName());
    // 创建服务暴露的url
    URL exporturl = new URL(DubboProtocol.NAME, channel.getLocalAddress().getAddress().getHostAddress(), channel.getLocalAddress().getPort(), clazz.getName() + "." + instid, tmpmap);

    // no need to generate multiple exporters for different channel in the same JVM, cache key cannot collide.

    // 获得缓存的key
    String cacheKey = getClientSideCallbackServiceCacheKey(instid);
    // 获得计数的key
    String countkey = getClientSideCountKey(clazz.getName());
    // 如果是暴露服务
    if (export) {
        // one channel can have multiple callback instances, no need to re-export for different instance.
        if (!channel.hasAttribute(cacheKey)) {
            if (!isInstancesOverLimit(channel, url, clazz.getName(), instid, false)) {
                // 获得代理对象
                Invoker<?> invoker = proxyFactory.getInvoker(inst, clazz, exporturl);
                // should destroy resource?
                // 暴露服务
                Exporter<?> exporter = protocol.export(invoker);
                // this is used for tracing if instid has published service or not.
                // 放到通道
                channel.setAttribute(cacheKey, exporter);
                logger.info("export a callback service :" + exporturl + ", on " + channel + ", url is: " + url);
                // 计数器加1
                increaseInstanceCount(channel, countkey);
            }
        }
    } else {
        // 如果通道内已经有该服务的缓存
        if (channel.hasAttribute(cacheKey)) {
            // 则获得该暴露者
            Exporter<?> exporter = (Exporter<?>) channel.getAttribute(cacheKey);
            // 取消暴露
            exporter.unexport();
            // 移除该缓存
            channel.removeAttribute(cacheKey);
            // 计数器减1
            decreaseInstanceCount(channel, countkey);
        }
    }
    return String.valueOf(instid);
}

```

该方法是在客户端侧暴露服务和取消暴露服务。

#### 6.referOrdestroyCallbackService

```java
private static Object referOrdestroyCallbackService(Channel channel, URL url, Class<?> clazz, Invocation inv, int instid, boolean isRefer) {
    Object proxy = null;
    // 获得服务调用的缓存key
    String invokerCacheKey = getServerSideCallbackInvokerCacheKey(channel, clazz.getName(), instid);
    // 获得代理缓存key
    String proxyCacheKey = getServerSideCallbackServiceCacheKey(channel, clazz.getName(), instid);
    // 从通道内获得代理对象
    proxy = channel.getAttribute(proxyCacheKey);
    // 获得计数器key
    String countkey = getServerSideCountKey(channel, clazz.getName());
    // 如果是服务引用
    if (isRefer) {
        // 如果代理对象为空
        if (proxy == null) {
            // 获得服务引用的url
            URL referurl = URL.valueOf("callback://" + url.getAddress() + "/" + clazz.getName() + "?" + Constants.INTERFACE_KEY + "=" + clazz.getName());
            referurl = referurl.addParametersIfAbsent(url.getParameters()).removeParameter(Constants.METHODS_KEY);
            if (!isInstancesOverLimit(channel, referurl, clazz.getName(), instid, true)) {
                @SuppressWarnings("rawtypes")
                Invoker<?> invoker = new ChannelWrappedInvoker(clazz, channel, referurl, String.valueOf(instid));
                // 获得代理类
                proxy = proxyFactory.getProxy(invoker);
                // 设置代理类
                channel.setAttribute(proxyCacheKey, proxy);
                // 设置实体域
                channel.setAttribute(invokerCacheKey, invoker);
                // 计数器加1
                increaseInstanceCount(channel, countkey);

                //convert error fail fast .
                //ignore concurrent problem. 
                Set<Invoker<?>> callbackInvokers = (Set<Invoker<?>>) channel.getAttribute(Constants.CHANNEL_CALLBACK_KEY);
                if (callbackInvokers == null) {
                    // 创建回调的服务实体域集合
                    callbackInvokers = new ConcurrentHashSet<Invoker<?>>(1);
                    // 把该实体域加入集合中
                    callbackInvokers.add(invoker);
                    channel.setAttribute(Constants.CHANNEL_CALLBACK_KEY, callbackInvokers);
                }
                logger.info("method " + inv.getMethodName() + " include a callback service :" + invoker.getUrl() + ", a proxy :" + invoker + " has been created.");
            }
        }
    } else {
        // 销毁
        if (proxy != null) {
            Invoker<?> invoker = (Invoker<?>) channel.getAttribute(invokerCacheKey);
            try {
                Set<Invoker<?>> callbackInvokers = (Set<Invoker<?>>) channel.getAttribute(Constants.CHANNEL_CALLBACK_KEY);
                if (callbackInvokers != null) {
                    // 从集合中移除
                    callbackInvokers.remove(invoker);
                }
                // 销毁该调用
                invoker.destroy();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
            // cancel refer, directly remove from the map
            // 取消引用，直接从集合中移除
            channel.removeAttribute(proxyCacheKey);
            channel.removeAttribute(invokerCacheKey);
            // 计数器减1
            decreaseInstanceCount(channel, countkey);
        }
    }
    return proxy;
}

```

该方法是在服务端侧进行服务引用或者销毁回调服务。

### （十一）DubboCodec

该类是dubbo的编解码器，分别针对dubbo协议的request和response进行编码和解码。

#### 1.属性

```java
/**
 * dubbo名称
 */
public static final String NAME = "dubbo";
/**
 * 协议版本号
 */
public static final String DUBBO_VERSION = Version.getProtocolVersion();
/**
 * 响应携带着异常
 */
public static final byte RESPONSE_WITH_EXCEPTION = 0;
/**
 * 响应
 */
public static final byte RESPONSE_VALUE = 1;
/**
 * 响应结果为空
 */
public static final byte RESPONSE_NULL_VALUE = 2;
/**
 * 响应结果有异常并且带有附加值
 */
public static final byte RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS = 3;
/**
 * 响应结果有附加值
 */
public static final byte RESPONSE_VALUE_WITH_ATTACHMENTS = 4;
/**
 * 响应结果为空并带有附加值
 */
public static final byte RESPONSE_NULL_VALUE_WITH_ATTACHMENTS = 5;
/**
 * 对象空集合
 */
public static final Object[] EMPTY_OBJECT_ARRAY = new Object[0];
/**
 * 空的类集合
 */
public static final Class<?>[] EMPTY_CLASS_ARRAY = new Class<?>[0];
private static final Logger log = LoggerFactory.getLogger(DubboCodec.class);

```

#### 2.decodeBody

```java
@Override
protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
    byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
    // get request id.
    long id = Bytes.bytes2long(header, 4);
    // 如果是response
    if ((flag & FLAG_REQUEST) == 0) {
        // decode response.
        // 创建一个response
        Response res = new Response(id);
        // 如果是事件，则设置事件，这里有个问题，我提交了pr在新版本已经修复
        if ((flag & FLAG_EVENT) != 0) {
            res.setEvent(Response.HEARTBEAT_EVENT);
        }
        // get status.
        // 设置状态
        byte status = header[3];
        res.setStatus(status);
        try {
            // 反序列化
            ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
            // 如果状态是响应成功
            if (status == Response.OK) {
                Object data;
                // 如果是心跳事件，则按照心跳事件解码
                if (res.isHeartbeat()) {
                    data = decodeHeartbeatData(channel, in);
                } else if (res.isEvent()) {
                    // 如果是事件，则
                    data = decodeEventData(channel, in);
                } else {
                    // 否则对结果进行解码
                    DecodeableRpcResult result;
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        result = new DecodeableRpcResult(channel, res, is,
                                (Invocation) getRequestData(id), proto);
                        result.decode();
                    } else {
                        result = new DecodeableRpcResult(channel, res,
                                new UnsafeByteArrayInputStream(readMessageData(is)),
                                (Invocation) getRequestData(id), proto);
                    }
                    data = result;
                }
                // 把结果重新放入response中
                res.setResult(data);
            } else {
                // 否则设置错误信息
                res.setErrorMessage(in.readUTF());
            }
        } catch (Throwable t) {
            if (log.isWarnEnabled()) {
                log.warn("Decode response failed: " + t.getMessage(), t);
            }
            res.setStatus(Response.CLIENT_ERROR);
            res.setErrorMessage(StringUtils.toString(t));
        }
        return res;
    } else {
        // decode request.
        // 如果该消息是request
        Request req = new Request(id);
        // 设置版本
        req.setVersion(Version.getProtocolVersion());
        // 设置是否是双向请求
        req.setTwoWay((flag & FLAG_TWOWAY) != 0);
        // 设置是否是事件，该地方问题也在新版本修复
        if ((flag & FLAG_EVENT) != 0) {
            req.setEvent(Request.HEARTBEAT_EVENT);
        }
        try {
            Object data;
            // 反序列化
            ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
            // 进行解码
            if (req.isHeartbeat()) {
                data = decodeHeartbeatData(channel, in);
            } else if (req.isEvent()) {
                data = decodeEventData(channel, in);
            } else {
                DecodeableRpcInvocation inv;
                if (channel.getUrl().getParameter(
                        Constants.DECODE_IN_IO_THREAD_KEY,
                        Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                    inv = new DecodeableRpcInvocation(channel, req, is, proto);
                    inv.decode();
                } else {
                    inv = new DecodeableRpcInvocation(channel, req,
                            new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                }
                data = inv;
            }
            // 把body数据设置到response
            req.setData(data);
        } catch (Throwable t) {
            if (log.isWarnEnabled()) {
                log.warn("Decode request failed: " + t.getMessage(), t);
            }
            // bad request
            req.setBroken(true);
            req.setData(t);
        }
        return req;
    }
}

```

该方法是对request和response进行解码，用位运算来进行解码，其中的逻辑跟我在[ 《dubbo源码解析（十）远程通信——Exchange层》](https://segmentfault.com/a/1190000017467343)中讲到的编解码器逻辑差不多。

#### 3.encodeRequestData

```java
@Override
protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
    RpcInvocation inv = (RpcInvocation) data;

    // 输出版本
    out.writeUTF(version);
    // 输出path
    out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
    // 输出版本号
    out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));

    // 输出方法名称
    out.writeUTF(inv.getMethodName());
    // 输出参数类型
    out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
    // 输出参数
    Object[] args = inv.getArguments();
    if (args != null)
        for (int i = 0; i < args.length; i++) {
            out.writeObject(encodeInvocationArgument(channel, inv, i));
        }
    // 输出附加值
    out.writeObject(inv.getAttachments());
}

```

该方法是对请求数据的编码。

#### 4.encodeResponseData

```java
@Override
protected void encodeResponseData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
    Result result = (Result) data;
    // currently, the version value in Response records the version of Request
    boolean attach = Version.isSupportResponseAttatchment(version);
    // 获得异常
    Throwable th = result.getException();
    if (th == null) {
        Object ret = result.getValue();
        // 根据结果的不同输出不同的值
        if (ret == null) {
            out.writeByte(attach ? RESPONSE_NULL_VALUE_WITH_ATTACHMENTS : RESPONSE_NULL_VALUE);
        } else {
            out.writeByte(attach ? RESPONSE_VALUE_WITH_ATTACHMENTS : RESPONSE_VALUE);
            out.writeObject(ret);
        }
    } else {
        // 如果有异常，则输出异常
        out.writeByte(attach ? RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS : RESPONSE_WITH_EXCEPTION);
        out.writeObject(th);
    }

    if (attach) {
        // returns current version of Response to consumer side.
        // 在附加值中加入版本号
        result.getAttachments().put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
        // 输出版本号
        out.writeObject(result.getAttachments());
    }
}

```

该方法是对响应数据的编码。

### （十二）DubboCountCodec

该类是对DubboCodec的功能增强，增加了消息长度的限制。

```java
public final class DubboCountCodec implements Codec2 {

    private DubboCodec codec = new DubboCodec();

    @Override
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        codec.encode(channel, buffer, msg);
    }

    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        // 保存读取的标志
        int save = buffer.readerIndex();
        MultiMessage result = MultiMessage.create();
        do {
            Object obj = codec.decode(channel, buffer);
            // 粘包拆包
            if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
                buffer.readerIndex(save);
                break;
            } else {
                // 增加消息
                result.addMessage(obj);
                // 记录消息长度
                logMessageLength(obj, buffer.readerIndex() - save);
                save = buffer.readerIndex();
            }
        } while (true);
        // 如果结果为空，则返回需要更多的输入
        if (result.isEmpty()) {
            return Codec2.DecodeResult.NEED_MORE_INPUT;
        }
        if (result.size() == 1) {
            return result.get(0);
        }
        return result;
    }

    private void logMessageLength(Object result, int bytes) {
        if (bytes <= 0) {
            return;
        }
        // 如果是request类型
        if (result instanceof Request) {
            try {
                // 设置附加值
                ((RpcInvocation) ((Request) result).getData()).setAttachment(
                        Constants.INPUT_KEY, String.valueOf(bytes));
            } catch (Throwable e) {
                /* ignore */
            }
        } else if (result instanceof Response) {
            try {
                // 设置附加值 输出的长度
                ((RpcResult) ((Response) result).getResult()).setAttachment(
                        Constants.OUTPUT_KEY, String.valueOf(bytes));
            } catch (Throwable e) {
                /* ignore */
            }
        }
    }

}

```

### （十三）TraceFilter

该过滤器是增强的功能是通道的跟踪，会在通道内把最大的调用次数和现在的调用数量放进去。方便使用telnet来跟踪服务的调用次数等。

#### 1.属性

```java
/**
 * 跟踪数量的最大值key
 */
private static final String TRACE_MAX = "trace.max";

/**
 * 跟踪的数量
 */
private static final String TRACE_COUNT = "trace.count";

/**
 * 通道集合
 */
private static final ConcurrentMap<String, Set<Channel>> tracers = new ConcurrentHashMap<String, Set<Channel>>();

```

#### 2.addTracer

```java
public static void addTracer(Class<?> type, String method, Channel channel, int max) {
    // 设置最大的数量
    channel.setAttribute(TRACE_MAX, max);
    // 设置当前的数量
    channel.setAttribute(TRACE_COUNT, new AtomicInteger());
    // 获得key
    String key = method != null && method.length() > 0 ? type.getName() + "." + method : type.getName();
    // 获得通道集合
    Set<Channel> channels = tracers.get(key);
    // 如果为空，则新建
    if (channels == null) {
        tracers.putIfAbsent(key, new ConcurrentHashSet<Channel>());
        channels = tracers.get(key);
    }
    channels.add(channel);
}

```

该方法是对某一个通道进行跟踪，把现在的调用数量放到属性里面

#### 3.removeTracer

```java
public static void removeTracer(Class<?> type, String method, Channel channel) {
    // 移除最大值属性
    channel.removeAttribute(TRACE_MAX);
    // 移除数量属性
    channel.removeAttribute(TRACE_COUNT);
    String key = method != null && method.length() > 0 ? type.getName() + "." + method : type.getName();
    Set<Channel> channels = tracers.get(key);
    if (channels != null) {
        // 集合中移除该通道
        channels.remove(channel);
    }
}

```

该方法是移除通道的跟踪。

#### 4.invoke

```java
@Override
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    // 开始时间
    long start = System.currentTimeMillis();
    // 调用下一个调用链 获得结果
    Result result = invoker.invoke(invocation);
    // 调用结束时间
    long end = System.currentTimeMillis();
    // 如果通道跟踪大小大于0
    if (tracers.size() > 0) {
        // 服务key
        String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
        // 获得通道集合
        Set<Channel> channels = tracers.get(key);
        if (channels == null || channels.isEmpty()) {
            key = invoker.getInterface().getName();
            channels = tracers.get(key);
        }
        if (channels != null && !channels.isEmpty()) {
            // 遍历通道集合
            for (Channel channel : new ArrayList<Channel>(channels)) {
                // 如果通道是连接的
                if (channel.isConnected()) {
                    try {
                        // 获得跟踪的最大数
                        int max = 1;
                        Integer m = (Integer) channel.getAttribute(TRACE_MAX);
                        if (m != null) {
                            max = (int) m;
                        }
                        // 获得跟踪数量
                        int count = 0;
                        AtomicInteger c = (AtomicInteger) channel.getAttribute(TRACE_COUNT);
                        if (c == null) {
                            c = new AtomicInteger();
                            channel.setAttribute(TRACE_COUNT, c);
                        }
                        count = c.getAndIncrement();
                        // 如果数量小于最大数量则发送
                        if (count < max) {
                            String prompt = channel.getUrl().getParameter(Constants.PROMPT_KEY, Constants.DEFAULT_PROMPT);
                            channel.send("\r\n" + RpcContext.getContext().getRemoteAddress() + " -> "
                                    + invoker.getInterface().getName()
                                    + "." + invocation.getMethodName()
                                    + "(" + JSON.toJSONString(invocation.getArguments()) + ")" + " -> " + JSON.toJSONString(result.getValue())
                                    + "\r\nelapsed: " + (end - start) + " ms."
                                    + "\r\n\r\n" + prompt);
                        }
                        // 如果数量大于等于max - 1，则移除该通道
                        if (count >= max - 1) {
                            channels.remove(channel);
                        }
                    } catch (Throwable e) {
                        channels.remove(channel);
                        logger.warn(e.getMessage(), e);
                    }
                } else {
                    // 如果未连接，也移除该通道
                    channels.remove(channel);
                }
            }
        }
    }
    return result;
}

```

该方法是当服务被调用时，进行跟踪或者取消跟踪的处理逻辑，是核心的功能增强逻辑。

### （十四）FutureFilter

该类是处理异步和同步调用结果的过滤器。

#### 1.invoke

```java
@Override
public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
    // 是否是异步的调用
    final boolean isAsync = RpcUtils.isAsync(invoker.getUrl(), invocation);

    fireInvokeCallback(invoker, invocation);
    // need to configure if there's return value before the invocation in order to help invoker to judge if it's
    // necessary to return future.
    Result result = invoker.invoke(invocation);
    if (isAsync) {
        // 调用异步处理
        asyncCallback(invoker, invocation);
    } else {
        // 调用同步结果处理
        syncCallback(invoker, invocation, result);
    }
    return result;
}

```

该方法中根据是否为异步调用来分别执行asyncCallback和syncCallback方法。

#### 2.syncCallback

```java
private void syncCallback(final Invoker<?> invoker, final Invocation invocation, final Result result) {
    // 如果有异常
    if (result.hasException()) {
        // 则调用异常的结果处理
        fireThrowCallback(invoker, invocation, result.getException());
    } else {
        // 调用正常的结果处理
        fireReturnCallback(invoker, invocation, result.getValue());
    }
}

```

该方法是同步调用的返回结果处理，比较简单。

#### 3.asyncCallback

```java
private void asyncCallback(final Invoker<?> invoker, final Invocation invocation) {
    Future<?> f = RpcContext.getContext().getFuture();
    if (f instanceof FutureAdapter) {
        ResponseFuture future = ((FutureAdapter<?>) f).getFuture();
        // 设置回调
        future.setCallback(new ResponseCallback() {
            @Override
            public void done(Object rpcResult) {
                // 如果结果为空，则打印错误日志
                if (rpcResult == null) {
                    logger.error(new IllegalStateException("invalid result value : null, expected " + Result.class.getName()));
                    return;
                }
                ///must be rpcResult
                // 如果不是Result则打印错误日志
                if (!(rpcResult instanceof Result)) {
                    logger.error(new IllegalStateException("invalid result type :" + rpcResult.getClass() + ", expected " + Result.class.getName()));
                    return;
                }
                Result result = (Result) rpcResult;
                if (result.hasException()) {
                    // 如果有异常，则调用异常处理方法
                    fireThrowCallback(invoker, invocation, result.getException());
                } else {
                    // 如果正常的返回结果，则调用正常的处理方法
                    fireReturnCallback(invoker, invocation, result.getValue());
                }
            }

            @Override
            public void caught(Throwable exception) {
                fireThrowCallback(invoker, invocation, exception);
            }
        });
    }
}
```

该方法是异步调用的结果处理，把异步返回结果的逻辑写在回调函数里面。

#### 4.fireInvokeCallback

```java
private void fireInvokeCallback(final Invoker<?> invoker, final Invocation invocation) {
    // 获得调用的方法
    final Method onInvokeMethod = (Method) StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_INVOKE_METHOD_KEY));
    // 获得调用的服务
    final Object onInvokeInst = StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_INVOKE_INSTANCE_KEY));

    if (onInvokeMethod == null && onInvokeInst == null) {
        return;
    }
    if (onInvokeMethod == null || onInvokeInst == null) {
        throw new IllegalStateException("service:" + invoker.getUrl().getServiceKey() + " has a onreturn callback config , but no such " + (onInvokeMethod == null ? "method" : "instance") + " found. url:" + invoker.getUrl());
    }
    // 如果不可以访问，则设置为可访问
    if (!onInvokeMethod.isAccessible()) {
        onInvokeMethod.setAccessible(true);
    }

    // 获得参数数组
    Object[] params = invocation.getArguments();
    try {
        // 调用方法
        onInvokeMethod.invoke(onInvokeInst, params);
    } catch (InvocationTargetException e) {
        fireThrowCallback(invoker, invocation, e.getTargetException());
    } catch (Throwable e) {
        fireThrowCallback(invoker, invocation, e);
    }
}
```

该方法是调用方法的执行。

#### 5.fireReturnCallback

```java
private void fireReturnCallback(final Invoker<?> invoker, final Invocation invocation, final Object result) {
    final Method onReturnMethod = (Method) StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_RETURN_METHOD_KEY));
    final Object onReturnInst = StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_RETURN_INSTANCE_KEY));

    //not set onreturn callback
    if (onReturnMethod == null && onReturnInst == null) {
        return;
    }

    if (onReturnMethod == null || onReturnInst == null) {
        throw new IllegalStateException("service:" + invoker.getUrl().getServiceKey() + " has a onreturn callback config , but no such " + (onReturnMethod == null ? "method" : "instance") + " found. url:" + invoker.getUrl());
    }
    if (!onReturnMethod.isAccessible()) {
        onReturnMethod.setAccessible(true);
    }

    Object[] args = invocation.getArguments();
    Object[] params;
    // 获得返回结果类型
    Class<?>[] rParaTypes = onReturnMethod.getParameterTypes();
    // 设置参数和返回结果
    if (rParaTypes.length > 1) {
        if (rParaTypes.length == 2 && rParaTypes[1].isAssignableFrom(Object[].class)) {
            params = new Object[2];
            params[0] = result;
            params[1] = args;
        } else {
            params = new Object[args.length + 1];
            params[0] = result;
            System.arraycopy(args, 0, params, 1, args.length);
        }
    } else {
        params = new Object[]{result};
    }
    try {
        // 调用方法
        onReturnMethod.invoke(onReturnInst, params);
    } catch (InvocationTargetException e) {
        fireThrowCallback(invoker, invocation, e.getTargetException());
    } catch (Throwable e) {
        fireThrowCallback(invoker, invocation, e);
    }
}
```

该方法是正常的返回结果的处理。

#### 6.fireThrowCallback

```java
private void fireThrowCallback(final Invoker<?> invoker, final Invocation invocation, final Throwable exception) {
    final Method onthrowMethod = (Method) StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_THROW_METHOD_KEY));
    final Object onthrowInst = StaticContext.getSystemContext().get(StaticContext.getKey(invoker.getUrl(), invocation.getMethodName(), Constants.ON_THROW_INSTANCE_KEY));

    //onthrow callback not configured
    if (onthrowMethod == null && onthrowInst == null) {
        return;
    }
    if (onthrowMethod == null || onthrowInst == null) {
        throw new IllegalStateException("service:" + invoker.getUrl().getServiceKey() + " has a onthrow callback config , but no such " + (onthrowMethod == null ? "method" : "instance") + " found. url:" + invoker.getUrl());
    }
    if (!onthrowMethod.isAccessible()) {
        onthrowMethod.setAccessible(true);
    }
    // 获得抛出异常的类型
    Class<?>[] rParaTypes = onthrowMethod.getParameterTypes();
    if (rParaTypes[0].isAssignableFrom(exception.getClass())) {
        try {
            Object[] args = invocation.getArguments();
            Object[] params;

            // 把类型和抛出的异常值放入返回结果
            if (rParaTypes.length > 1) {
                if (rParaTypes.length == 2 && rParaTypes[1].isAssignableFrom(Object[].class)) {
                    params = new Object[2];
                    params[0] = exception;
                    params[1] = args;
                } else {
                    params = new Object[args.length + 1];
                    params[0] = exception;
                    System.arraycopy(args, 0, params, 1, args.length);
                }
            } else {
                params = new Object[]{exception};
            }
            // 调用下一个调用连
            onthrowMethod.invoke(onthrowInst, params);
        } catch (Throwable e) {
            logger.error(invocation.getMethodName() + ".call back method invoke error . callback method :" + onthrowMethod + ", url:" + invoker.getUrl(), e);
        }
    } else {
        logger.error(invocation.getMethodName() + ".call back method invoke error . callback method :" + onthrowMethod + ", url:" + invoker.getUrl(), exception);
    }
}

```

该方法是异常抛出时的结果处理。

### （十五）ServerStatusChecker

该类是对于服务状态的监控设置。

```java
public class ServerStatusChecker implements StatusChecker {

    @Override
    public Status check() {
        // 获得服务集合
        Collection<ExchangeServer> servers = DubboProtocol.getDubboProtocol().getServers();
        // 如果为空则返回UNKNOWN的状态
        if (servers == null || servers.isEmpty()) {
            return new Status(Status.Level.UNKNOWN);
        }
        // 设置状态为ok
        Status.Level level = Status.Level.OK;
        StringBuilder buf = new StringBuilder();
        // 遍历集合
        for (ExchangeServer server : servers) {
            // 如果服务没有绑定到本地端口
            if (!server.isBound()) {
                // 状态改为error
                level = Status.Level.ERROR;
                // 加入服务本地地址
                buf.setLength(0);
                buf.append(server.getLocalAddress());
                break;
            }
            if (buf.length() > 0) {
                buf.append(",");
            }
            // 如果服务绑定了本地端口，拼接clients数量
            buf.append(server.getLocalAddress());
            buf.append("(clients:");
            buf.append(server.getChannels().size());
            buf.append(")");
        }
        return new Status(level, buf.toString());
    }

}

```

### （十六）ThreadPoolStatusChecker

该类是对于线程池的状态进行监控。

```java
@Activate
public class ThreadPoolStatusChecker implements StatusChecker {

    @Override
    public Status check() {
        // 获得数据中心
        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        // 获得线程池集合
        Map<String, Object> executors = dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY);

        StringBuilder msg = new StringBuilder();
        // 设置为ok
        Status.Level level = Status.Level.OK;
        // 遍历线程池集合
        for (Map.Entry<String, Object> entry : executors.entrySet()) {
            String port = entry.getKey();
            ExecutorService executor = (ExecutorService) entry.getValue();

            if (executor != null && executor instanceof ThreadPoolExecutor) {
                ThreadPoolExecutor tp = (ThreadPoolExecutor) executor;
                boolean ok = tp.getActiveCount() < tp.getMaximumPoolSize() - 1;
                Status.Level lvl = Status.Level.OK;
                // 如果活跃数量超过了最大的线程数量，则设置warn
                if (!ok) {
                    level = Status.Level.WARN;
                    lvl = Status.Level.WARN;
                }

                if (msg.length() > 0) {
                    msg.append(";");
                }
                // 输出线程池相关信息
                msg.append("Pool status:" + lvl
                        + ", max:" + tp.getMaximumPoolSize()
                        + ", core:" + tp.getCorePoolSize()
                        + ", largest:" + tp.getLargestPoolSize()
                        + ", active:" + tp.getActiveCount()
                        + ", task:" + tp.getTaskCount()
                        + ", service port: " + port);
            }
        }
        return msg.length() == 0 ? new Status(Status.Level.UNKNOWN) : new Status(level, msg.toString());
    }

}

```

逻辑比较简单，我就不赘述了。

关于telnet下的相关实现请感兴趣的朋友直接查看，里面都是对于telnet命令的实现，内容比较独立。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-dubbo/src/main/java/com/alibaba/dubbo/rpc/protocol/dubbo

该文章讲解了远程调用中关于dubbo协议的部分，dubbo协议是官方推荐使用的协议，并且对于telnet命令也做了很好的支持，要看懂这部分的逻辑，必须先对于之前的一些接口设计了解的很清楚。接下来我将开始对rpc模块关于hessian协议部分进行讲解。