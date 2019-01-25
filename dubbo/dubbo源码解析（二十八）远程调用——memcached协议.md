# 远程调用——memcached协议

> 目标：介绍memcached协议的设计和实现，介绍dubbo-rpc-memcached的源码。

## 前言

dubbo实现memcached协议是基于Memcached，[Memcached](http://memcached.org/) 是一个高效的 KV 缓存服务器，在dubbo中没有涉及到关于memcached协议的服务暴露，只有服务引用，因为在访问Memcached服务器时，Memcached客户端可以在服务器上存储也可以获取。

## 源码分析

### （一）MemcachedProtocol

该类继承AbstractProtocol，是memcached协议实现的核心。

#### 1.属性

```java
/**
 * 默认端口号
 */
public static final int DEFAULT_PORT = 11211;
```

#### 2.export

```java
@Override
public <T> Exporter<T> export(final Invoker<T> invoker) throws RpcException {
    // 不支持memcached服务暴露
    throw new UnsupportedOperationException("Unsupported export memcached service. url: " + invoker.getUrl());
}
```

可以看到，服务暴露方法直接抛出异常。

#### 3.refer

```java
@Override
public <T> Invoker<T> refer(final Class<T> type, final URL url) throws RpcException {
    try {
        // 获得地址
        String address = url.getAddress();
        // 获得备用地址
        String backup = url.getParameter(Constants.BACKUP_KEY);
        // 把备用地址拼接上
        if (backup != null && backup.length() > 0) {
            address += "," + backup;
        }
        // 创建Memcached客户端构造器
        MemcachedClientBuilder builder = new XMemcachedClientBuilder(AddrUtil.getAddresses(address));
        // 创建客户端
        final MemcachedClient memcachedClient = builder.build();
        // 到期时间参数配置
        final int expiry = url.getParameter("expiry", 0);
        // 获得值命令
        final String get = url.getParameter("get", "get");
        // 添加值命令根据类型来取决是put还是set
        final String set = url.getParameter("set", Map.class.equals(type) ? "put" : "set");
        // 删除值命令
        final String delete = url.getParameter("delete", Map.class.equals(type) ? "remove" : "delete");
        return new AbstractInvoker<T>(type, url) {
            @Override
            protected Result doInvoke(Invocation invocation) throws Throwable {
                try {
                    // 如果是获取方法名的值
                    if (get.equals(invocation.getMethodName())) {
                        // 如果参数长度不等于1，则抛出异常
                        if (invocation.getArguments().length != 1) {
                            throw new IllegalArgumentException("The memcached get method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        // 否则调用get方法来获取
                        return new RpcResult(memcachedClient.get(String.valueOf(invocation.getArguments()[0])));
                    } else if (set.equals(invocation.getMethodName())) {
                        // 如果参数长度不为2，则抛出异常
                        if (invocation.getArguments().length != 2) {
                            throw new IllegalArgumentException("The memcached set method arguments mismatch, must be two arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        // 无论任何现有值如何，都在缓存中设置一个对象
                        memcachedClient.set(String.valueOf(invocation.getArguments()[0]), expiry, invocation.getArguments()[1]);
                        return new RpcResult();
                    } else if (delete.equals(invocation.getMethodName())) {
                        // 删除操作只有一个参数，如果参数长度不等于1，则抛出异常
                        if (invocation.getArguments().length != 1) {
                            throw new IllegalArgumentException("The memcached delete method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        // 删除某个值
                        memcachedClient.delete(String.valueOf(invocation.getArguments()[0]));
                        return new RpcResult();
                    } else {
                        // 不支持的操作
                        throw new UnsupportedOperationException("Unsupported method " + invocation.getMethodName() + " in memcached service.");
                    }
                } catch (Throwable t) {
                    RpcException re = new RpcException("Failed to invoke memcached service method. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url + ", cause: " + t.getMessage(), t);
                    if (t instanceof TimeoutException || t instanceof SocketTimeoutException) {
                        re.setCode(RpcException.TIMEOUT_EXCEPTION);
                    } else if (t instanceof MemcachedException || t instanceof IOException) {
                        re.setCode(RpcException.NETWORK_EXCEPTION);
                    }
                    throw re;
                }
            }

            @Override
            public void destroy() {
                super.destroy();
                try {
                    // 关闭客户端
                    memcachedClient.shutdown();
                } catch (Throwable e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        };
    } catch (Throwable t) {
        throw new RpcException("Failed to refer memcached service. interface: " + type.getName() + ", url: " + url + ", cause: " + t.getMessage(), t);
    }
}
```

该方法是服务引用方法，基于MemcachedClient的get、set、delete方法来对应Memcached的get、set、delete命令进行对值的操作。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-memcached/src/main/java/com/alibaba/dubbo/rpc/protocol/memcached

该文章讲解了远程调用中关于memcached协议实现的部分，逻辑比较简单。接下来我将开始对rpc模块关于redis协议部分进行讲解。