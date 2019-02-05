# 集群——merger

> 目标：介绍dubbo中集群的配置规则，介绍dubbo-cluster下merger包的源码。

## 前言

按组合并返回结果 ，比如菜单服务，接口一样，但有多种实现，用group区分，现在消费方需从每种group中调用一次返回结果，合并结果返回，这样就可以实现聚合菜单项。这个时候就要用到分组聚合。

## 源码分析

### （一）MergeableCluster

```java
public class MergeableCluster implements Cluster {

    public static final String NAME = "mergeable";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        // 创建MergeableClusterInvoker
        return new MergeableClusterInvoker<T>(directory);
    }

}
```

该类实现了Cluster接口，是分组集合的集群实现。

### （二）MergeableClusterInvoker

该类是分组聚合的实现类，其中最关机的就是invoke方法。

```java
@Override
@SuppressWarnings("rawtypes")
public Result invoke(final Invocation invocation) throws RpcException {
    // 获得invoker集合
    List<Invoker<T>> invokers = directory.list(invocation);

    /**
     * 获得是否merger
     */
    String merger = getUrl().getMethodParameter(invocation.getMethodName(), Constants.MERGER_KEY);
    // 如果没有设置需要聚合，则只调用一个invoker的
    if (ConfigUtils.isEmpty(merger)) { // If a method doesn't have a merger, only invoke one Group
        // 只要有一个可用就返回
        for (final Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return invoker.invoke(invocation);
            }
        }
        return invokers.iterator().next().invoke(invocation);
    }

    // 返回类型
    Class<?> returnType;
    try {
        // 获得返回类型
        returnType = getInterface().getMethod(
                invocation.getMethodName(), invocation.getParameterTypes()).getReturnType();
    } catch (NoSuchMethodException e) {
        returnType = null;
    }

    // 结果集合
    Map<String, Future<Result>> results = new HashMap<String, Future<Result>>();
    // 循环invokers
    for (final Invoker<T> invoker : invokers) {
        // 获得每次调用的future
        Future<Result> future = executor.submit(new Callable<Result>() {
            @Override
            public Result call() throws Exception {
                // 回调，把返回结果放入future
                return invoker.invoke(new RpcInvocation(invocation, invoker));
            }
        });
        // 加入集合
        results.put(invoker.getUrl().getServiceKey(), future);
    }

    Object result = null;

    List<Result> resultList = new ArrayList<Result>(results.size());

    // 获得超时时间
    int timeout = getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    // 遍历每一个结果
    for (Map.Entry<String, Future<Result>> entry : results.entrySet()) {
        Future<Result> future = entry.getValue();
        try {
            // 获得调用返回的结果
            Result r = future.get(timeout, TimeUnit.MILLISECONDS);
            if (r.hasException()) {
                log.error("Invoke " + getGroupDescFromServiceKey(entry.getKey()) + 
                                " failed: " + r.getException().getMessage(), 
                        r.getException());
            } else {
                // 加入集合
                resultList.add(r);
            }
        } catch (Exception e) {
            throw new RpcException("Failed to invoke service " + entry.getKey() + ": " + e.getMessage(), e);
        }
    }

    // 如果为空，则返回空的结果
    if (resultList.isEmpty()) {
        return new RpcResult((Object) null);
    } else if (resultList.size() == 1) {
        // 如果只有一个结果，则返回该结果
        return resultList.iterator().next();
    }

    // 如果返回类型是void，也就是没有返回值，那么返回空结果
    if (returnType == void.class) {
        return new RpcResult((Object) null);
    }

    // 根据方法来合并，将调用返回结果的指定方法进行合并
    if (merger.startsWith(".")) {
        merger = merger.substring(1);
        Method method;
        try {
            // 获得方法
            method = returnType.getMethod(merger, returnType);
        } catch (NoSuchMethodException e) {
            throw new RpcException("Can not merge result because missing method [ " + merger + " ] in class [ " + 
                    returnType.getClass().getName() + " ]");
        }

        // 有 Method ，进行合并
        if (!Modifier.isPublic(method.getModifiers())) {
            method.setAccessible(true);
        }
        // 从集合中移除
        result = resultList.remove(0).getValue();
        try {
            // 方法返回类型匹配，合并时，修改 result
            if (method.getReturnType() != void.class
                    && method.getReturnType().isAssignableFrom(result.getClass())) {
                for (Result r : resultList) {
                    result = method.invoke(result, r.getValue());
                }
            } else {
                // 方法返回类型不匹配，合并时，不修改 result
                for (Result r : resultList) {
                    method.invoke(result, r.getValue());
                }
            }
        } catch (Exception e) {
            throw new RpcException("Can not merge result: " + e.getMessage(), e);
        }
    } else {
        // 基于 Merger
        Merger resultMerger;
        // 如果是默认的方式
        if (ConfigUtils.isDefault(merger)) {
            // 获得该类型的合并方式
            resultMerger = MergerFactory.getMerger(returnType);
        } else {
            // 如果不是默认的，则配置中指定获得Merger的实现类
            resultMerger = ExtensionLoader.getExtensionLoader(Merger.class).getExtension(merger);
        }
        if (resultMerger != null) {
            List<Object> rets = new ArrayList<Object>(resultList.size());
            // 遍历返回结果
            for (Result r : resultList) {
                // 加入到rets
                rets.add(r.getValue());
            }
            // 合并
            result = resultMerger.merge(
                    rets.toArray((Object[]) Array.newInstance(returnType, 0)));
        } else {
            throw new RpcException("There is no merger to merge result.");
        }
    }
    // 返回结果
    return new RpcResult(result);
}
```

前面部分在讲获得调用的结果，后面部分是对结果的合并，合并有两种方式，根据配置不同可用分为基于方法的合并和基于merger的合并。

### （三）MergerFactory

Merger 工厂类，获得指定类型的Merger 对象。

```java
public class MergerFactory {

    /**
     * Merger 对象缓存
     */
    private static final ConcurrentMap<Class<?>, Merger<?>> mergerCache =
            new ConcurrentHashMap<Class<?>, Merger<?>>();

    /**
     * 获得指定类型的Merger对象
     * @param returnType
     * @param <T>
     * @return
     */
    public static <T> Merger<T> getMerger(Class<T> returnType) {
        Merger result;
        // 如果类型是集合
        if (returnType.isArray()) {
            // 获得类型
            Class type = returnType.getComponentType();
            // 从缓存中获得该类型的Merger对象
            result = mergerCache.get(type);
            // 如果为空，则
            if (result == null) {
                // 初始化所有的 Merger 扩展对象，到 mergerCache 缓存中。
                loadMergers();
                // 从集合中取出对应的Merger对象
                result = mergerCache.get(type);
            }
            // 如果结果为空，则直接返回ArrayMerger的单例
            if (result == null && !type.isPrimitive()) {
                result = ArrayMerger.INSTANCE;
            }
        } else {
            // 否则直接从mergerCache中取出
            result = mergerCache.get(returnType);
            // 如果为空
            if (result == null) {
                // 初始化所有的 Merger 扩展对象，到 mergerCache 缓存中。
                loadMergers();
                // 从集合中取出
                result = mergerCache.get(returnType);
            }
        }
        return result;
    }

    /**
     * 初始化所有的 Merger 扩展对象，到 mergerCache 缓存中。
     */
    static void loadMergers() {
        // 获得Merger所有的扩展对象名
        Set<String> names = ExtensionLoader.getExtensionLoader(Merger.class)
                .getSupportedExtensions();
        // 遍历
        for (String name : names) {
            // 加载每一个扩展实现，然后放入缓存。
            Merger m = ExtensionLoader.getExtensionLoader(Merger.class).getExtension(name);
            mergerCache.putIfAbsent(ReflectUtils.getGenericClass(m.getClass()), m);
        }
    }

}
```

逻辑比较简单。

### （四）ArrayMerger

因为不同的类型有不同的Merger实现，我们可以来看看这个图片：

![merger](https://github.com/CrazyHZM/crazy-code-analysis/blob/master/dubbo/image/%E4%B8%89%E5%8D%81%E4%B9%9D/merger.png?raw=true)

可以看到有好多好多，我就讲解其中的一种，偷懒一下，其他的麻烦有兴趣的去看看源码了。

```java
public class ArrayMerger implements Merger<Object[]> {

    /**
     * 单例
     */
    public static final ArrayMerger INSTANCE = new ArrayMerger();

    @Override
    public Object[] merge(Object[]... others) {
        // 如果长度为0  则直接返回
        if (others.length == 0) {
            return null;
        }
        // 总长
        int totalLen = 0;
        // 遍历所有需要合并的对象
        for (int i = 0; i < others.length; i++) {
            Object item = others[i];
            // 如果为数组
            if (item != null && item.getClass().isArray()) {
                // 累加数组长度
                totalLen += Array.getLength(item);
            } else {
                throw new IllegalArgumentException((i + 1) + "th argument is not an array");
            }
        }

        if (totalLen == 0) {
            return null;
        }

        // 获得数组类型
        Class<?> type = others[0].getClass().getComponentType();

        // 创建长度
        Object result = Array.newInstance(type, totalLen);
        int index = 0;
        // 遍历需要合并的对象
        for (Object array : others) {
            // 遍历每个数组中的数据
            for (int i = 0; i < Array.getLength(array); i++) {
                // 加入到最终结果中
                Array.set(result, index++, Array.get(array, i));
            }
        }
        return (Object[]) result;
    }

}
```

是不是很简单，就是循环合并就可以了。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/merger

该文章讲解了集群中关于分组聚合实现的部分。接下来我将开始对集群模块关于路由部分进行讲解。