# 远程调用——redis协议

> 目标：介绍redis协议的设计和实现，介绍dubbo-rpc-redis的源码。

## 前言

dubbo支持的redis协议是基于Redis的，[Redis](http://redis.io/) 是一个高效的 KV 存储服务器，跟memcached协议实现差不多，在dubbo中也没有涉及到关于redis协议的服务暴露，只有服务引用，因为在访问服务器时，Redis客户端可以在服务器上存储也可以获取。

## 源码分析

### （一）RedisProtocol

该类继承了AbstractProtocol类，是redis协议实现的核心。

#### 1.属性

```java
/**
 * 默认端口号
 */
public static final int DEFAULT_PORT = 6379;
```

#### 2.export

```java
@Override
public <T> Exporter<T> export(final Invoker<T> invoker) throws RpcException {
    // 不支持redis协议的服务暴露，抛出异常
    throw new UnsupportedOperationException("Unsupported export redis service. url: " + invoker.getUrl());
}
```

可以看到不支持服务暴露。

#### 3.refer

```java
@Override
public <T> Invoker<T> refer(final Class<T> type, final URL url) throws RpcException {
    try {
        // 实例化对象池
        GenericObjectPoolConfig config = new GenericObjectPoolConfig();
        // 如果 testOnBorrow 被设置，pool 会在 borrowObject 返回对象之前使用 PoolableObjectFactory的 validateObject 来验证这个对象是否有效
        // 要是对象没通过验证，这个对象会被丢弃，然后重新选择一个新的对象。
        config.setTestOnBorrow(url.getParameter("test.on.borrow", true));
        // 如果 testOnReturn 被设置， pool 会在 returnObject 的时候通过 PoolableObjectFactory 的validateObject 方法验证对象
        // 如果对象没通过验证，对象会被丢弃，不会被放到池中。
        config.setTestOnReturn(url.getParameter("test.on.return", false));
        // 指定空闲对象是否应该使用 PoolableObjectFactory 的 validateObject 校验，如果校验失败，这个对象会从对象池中被清除。
        // 这个设置仅在 timeBetweenEvictionRunsMillis 被设置成正值（ >0） 的时候才会生效。
        config.setTestWhileIdle(url.getParameter("test.while.idle", false));
        if (url.getParameter("max.idle", 0) > 0)
            // 控制一个pool最多有多少个状态为空闲的jedis实例。
            config.setMaxIdle(url.getParameter("max.idle", 0));
        if (url.getParameter("min.idle", 0) > 0)
            // 控制一个pool最少有多少个状态为空闲的jedis实例。
            config.setMinIdle(url.getParameter("min.idle", 0));
        if (url.getParameter("max.active", 0) > 0)
            // 控制一个pool最多有多少个jedis实例。
            config.setMaxTotal(url.getParameter("max.active", 0));
        if (url.getParameter("max.total", 0) > 0)
            config.setMaxTotal(url.getParameter("max.total", 0));
        if (url.getParameter("max.wait", 0) > 0)
            //表示当引入一个jedis实例时，最大的等待时间，如果超过等待时间，则直接抛出JedisConnectionException；
            config.setMaxWaitMillis(url.getParameter("max.wait", 0));
        if (url.getParameter("num.tests.per.eviction.run", 0) > 0)
            // 设置驱逐线程每次检测对象的数量。这个设置仅在 timeBetweenEvictionRunsMillis 被设置成正值（ >0）的时候才会生效。
            config.setNumTestsPerEvictionRun(url.getParameter("num.tests.per.eviction.run", 0));
        if (url.getParameter("time.between.eviction.runs.millis", 0) > 0)
            // 指定驱逐线程的休眠时间。如果这个值不是正数（ >0），不会有驱逐线程运行。
            config.setTimeBetweenEvictionRunsMillis(url.getParameter("time.between.eviction.runs.millis", 0));
        if (url.getParameter("min.evictable.idle.time.millis", 0) > 0)
            // 指定最小的空闲驱逐的时间间隔（空闲超过指定的时间的对象，会被清除掉）。
            // 这个设置仅在 timeBetweenEvictionRunsMillis 被设置成正值（ >0）的时候才会生效。
            config.setMinEvictableIdleTimeMillis(url.getParameter("min.evictable.idle.time.millis", 0));
        // 创建redis连接池
        final JedisPool jedisPool = new JedisPool(config, url.getHost(), url.getPort(DEFAULT_PORT),
                url.getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT),
                StringUtils.isBlank(url.getPassword()) ? null : url.getPassword(),
                url.getParameter("db.index", 0));
        // 获得值的过期时间
        final int expiry = url.getParameter("expiry", 0);
        // 获得get命令
        final String get = url.getParameter("get", "get");
        // 获得set命令
        final String set = url.getParameter("set", Map.class.equals(type) ? "put" : "set");
        // 获得delete命令
        final String delete = url.getParameter("delete", Map.class.equals(type) ? "remove" : "delete");
        return new AbstractInvoker<T>(type, url) {
            @Override
            protected Result doInvoke(Invocation invocation) throws Throwable {
                Jedis resource = null;
                try {
                    resource = jedisPool.getResource();

                    // 如果是get命令
                    if (get.equals(invocation.getMethodName())) {
                        // get 命令必须只有一个参数
                        if (invocation.getArguments().length != 1) {
                            throw new IllegalArgumentException("The redis get method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        // 获得值
                        byte[] value = resource.get(String.valueOf(invocation.getArguments()[0]).getBytes());
                        if (value == null) {
                            return new RpcResult();
                        }
                        // 反序列化
                        ObjectInput oin = getSerialization(url).deserialize(url, new ByteArrayInputStream(value));
                        return new RpcResult(oin.readObject());
                    } else if (set.equals(invocation.getMethodName())) {
                        // 如果是set命令，参数长度必须是2
                        if (invocation.getArguments().length != 2) {
                            throw new IllegalArgumentException("The redis set method arguments mismatch, must be two arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        //
                        byte[] key = String.valueOf(invocation.getArguments()[0]).getBytes();
                        ByteArrayOutputStream output = new ByteArrayOutputStream();
                        // 对需要存入对值进行序列化
                        ObjectOutput value = getSerialization(url).serialize(url, output);
                        value.writeObject(invocation.getArguments()[1]);
                        // 存入值
                        resource.set(key, output.toByteArray());
                        // 设置该key过期时间，不能大于1000s
                        if (expiry > 1000) {
                            resource.expire(key, expiry / 1000);
                        }
                        return new RpcResult();
                    } else if (delete.equals(invocation.getMethodName())) {
                        // 如果是删除命令，则参数长度必须是1
                        if (invocation.getArguments().length != 1) {
                            throw new IllegalArgumentException("The redis delete method arguments mismatch, must only one arguments. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url);
                        }
                        // 删除该值
                        resource.del(String.valueOf(invocation.getArguments()[0]).getBytes());
                        return new RpcResult();
                    } else {
                        // 否则抛出该操作不支持的异常
                        throw new UnsupportedOperationException("Unsupported method " + invocation.getMethodName() + " in redis service.");
                    }
                } catch (Throwable t) {
                    RpcException re = new RpcException("Failed to invoke redis service method. interface: " + type.getName() + ", method: " + invocation.getMethodName() + ", url: " + url + ", cause: " + t.getMessage(), t);
                    if (t instanceof TimeoutException || t instanceof SocketTimeoutException) {
                        // 抛出超时异常
                        re.setCode(RpcException.TIMEOUT_EXCEPTION);
                    } else if (t instanceof JedisConnectionException || t instanceof IOException) {
                        // 抛出网络异常
                        re.setCode(RpcException.NETWORK_EXCEPTION);
                    } else if (t instanceof JedisDataException) {
                        // 抛出序列化异常
                        re.setCode(RpcException.SERIALIZATION_EXCEPTION);
                    }
                    throw re;
                } finally {
                    if (resource != null) {
                        try {
                            jedisPool.returnResource(resource);
                        } catch (Throwable t) {
                            logger.warn("returnResource error: " + t.getMessage(), t);
                        }
                    }
                }
            }

            @Override
            public void destroy() {
                super.destroy();
                try {
                    // 关闭连接池
                    jedisPool.destroy();
                } catch (Throwable e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        };
    } catch (Throwable t) {
        throw new RpcException("Failed to refer redis service. interface: " + type.getName() + ", url: " + url + ", cause: " + t.getMessage(), t);
    }
}
```

可以看到首先是对连接池的配置赋值，然后创建连接池后，根据redis的get、set、delete命令来进行相关操作。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-rpc/dubbo-rpc-redis/src/main/java/com/alibaba/dubbo/rpc/protocol/redis

该文章讲解了远程调用中关于redis协议实现的部分，逻辑比较简单。接下来我将开始对rpc模块关于rest协议部分进行讲解。