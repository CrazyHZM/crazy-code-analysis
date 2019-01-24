# 远程调用——injvm本地调用

> 目标：介绍injvm本地调用的设计和实现，介绍dubbo-rpc-injvm的源码。

## 前言

dubbo是一个远程调用的框架，但是它没有理由不支持本地调用，本文就要讲解dubbo关于本地调用的实现。本地调用要比远程调用简单的多。

## 源码分析

### （一）InjvmExporter

该类继承了AbstractExporter，是本地服务的暴露者封装，其中实现比较简单。只是实现了unexport方法，并且维护了一份保存暴露者的集合。

```java
class InjvmExporter<T> extends AbstractExporter<T> {

    /**
     * 服务key
     */
    private final String key;

    /**
     * 暴露者集合
     */
    private final Map<String, Exporter<?>> exporterMap;

    InjvmExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
        exporterMap.put(key, this);
    }

    /**
     * 取消暴露
     */
    @Override
    public void unexport() {
        // 调用父类的取消暴露方法
        super.unexport();
        // 从集合中移除
        exporterMap.remove(key);
    }

}
```

### （二）InjvmInvoker

该类继承了AbstractInvoker类，是本地调用的invoker实现。

```java
class InjvmInvoker<T> extends AbstractInvoker<T> {

    /**
     * 服务key
     */
    private final String key;

    /**
     * 暴露者集合
     */
    private final Map<String, Exporter<?>> exporterMap;

    InjvmInvoker(Class<T> type, URL url, String key, Map<String, Exporter<?>> exporterMap) {
        super(type, url);
        this.key = key;
        this.exporterMap = exporterMap;
    }

    /**
     * 服务是否活跃
     * @return
     */
    @Override
    public boolean isAvailable() {
        InjvmExporter<?> exporter = (InjvmExporter<?>) exporterMap.get(key);
        if (exporter == null) {
            return false;
        } else {
            return super.isAvailable();
        }
    }

    /**
     * invoke方法
     * @param invocation
     * @return
     * @throws Throwable
     */
    @Override
    public Result doInvoke(Invocation invocation) throws Throwable {
        // 获得暴露者
        Exporter<?> exporter = InjvmProtocol.getExporter(exporterMap, getUrl());
        // 如果为空，则抛出异常
        if (exporter == null) {
            throw new RpcException("Service [" + key + "] not found.");
        }
        // 设置远程地址为127.0.0.1
        RpcContext.getContext().setRemoteAddress(NetUtils.LOCALHOST, 0);
        // 调用下一个调用链
        return exporter.getInvoker().invoke(invocation);
    }
}
```

其中重写了isAvailable和doInvoke方法。

### （三）InjvmProtocol

该类是本地调用的协议实现，其中实现了服务调用和服务暴露方法，并且封装了一个判断是否是本地调用的方法。

#### 1.属性

```java
/**
 * 本地调用  Protocol的实现类key
 */
public static final String NAME = Constants.LOCAL_PROTOCOL;

/**
 * 默认端口
 */
public static final int DEFAULT_PORT = 0;
/**
 * 单例
 */
private static InjvmProtocol INSTANCE;
```

#### 2.getExporter

```java
static Exporter<?> getExporter(Map<String, Exporter<?>> map, URL key) {
    Exporter<?> result = null;

    // 如果服务key不是*
    if (!key.getServiceKey().contains("*")) {
        // 直接从集合中取出
        result = map.get(key.getServiceKey());
    } else {
        // 如果 map不为空，则遍历暴露者，来找到对应的exporter
        if (map != null && !map.isEmpty()) {
            for (Exporter<?> exporter : map.values()) {
                // 如果是服务key
                if (UrlUtils.isServiceKeyMatch(key, exporter.getInvoker().getUrl())) {
                    // 赋值
                    result = exporter;
                    break;
                }
            }
        }
    }

    // 如果没有找到exporter
    if (result == null) {
        // 则返回null
        return null;
    } else if (ProtocolUtils.isGeneric(
            result.getInvoker().getUrl().getParameter(Constants.GENERIC_KEY))) {
        // 如果是泛化调用，则返回null
        return null;
    } else {
        return result;
    }
}
```

该方法是获得相关的暴露者。

#### 3.export

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 创建InjvmExporter 并且返回
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

#### 4.refer

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    // 创建InjvmInvoker 并且返回
    return new InjvmInvoker<T>(serviceType, url, url.getServiceKey(), exporterMap);
}
```

#### 5.isInjvmRefer

```java
public boolean isInjvmRefer(URL url) {
    final boolean isJvmRefer;
    // 获得scope配置
    String scope = url.getParameter(Constants.SCOPE_KEY);
    // Since injvm protocol is configured explicitly, we don't need to set any extra flag, use normal refer process.
    if (Constants.LOCAL_PROTOCOL.toString().equals(url.getProtocol())) {
        // 如果是injvm，则不是本地调用
        isJvmRefer = false;
    } else if (Constants.SCOPE_LOCAL.equals(scope) || (url.getParameter("injvm", false))) {
        // if it's declared as local reference
        // 'scope=local' is equivalent to 'injvm=true', injvm will be deprecated in the future release
        // 如果它被声明为本地引用 scope = local'相当于'injvm = true'，将在以后的版本中弃用injvm
        isJvmRefer = true;
    } else if (Constants.SCOPE_REMOTE.equals(scope)) {
        // it's declared as remote reference
        // 如果被声明为远程调用
        isJvmRefer = false;
    } else if (url.getParameter(Constants.GENERIC_KEY, false)) {
        // generic invocation is not local reference
        // 泛化的调用不是本地调用
        isJvmRefer = false;
    } else if (getExporter(exporterMap, url) != null) {
        // by default, go through local reference if there's the service exposed locally
        // 默认情况下，如果本地暴露服务，请通过本地引用
        isJvmRefer = true;
    } else {
        isJvmRefer = false;
    }
    return isJvmRefer;
}
```

该方法是判断是否为本地调用。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-injvm/src/main/java/com/alibaba/dubbo/rpc/protocol/injvm

该文章讲解了远程调用中关于injvm本地调用的部分，三种抽象的角色还是比较鲜明的，服务暴露相关的exporter、服务引用相关的invoker、以及协议相关的protocol，关键还是弄清楚再设计上的意图，以及他们分别代表的是什么。那么看这些不同的协议实现会很容易看懂。接下来我将开始对rpc模块关于memcached协议部分进行讲解。

