# 远程调用——directory

> 目标：介绍dubbo中集群的目录，介绍dubbo-cluster下directory包的源码。

## 前言

我在前面的文章中也提到了Directory可以看成是多个Invoker的集合，Directory 的用途是保存 Invoker，其实现类 RegistryDirectory 是一个动态服务目录，可感知注册中心配置的变化，它所持有的 Inovker 列表会随着注册中心内容的变化而变化。每次变化后，RegistryDirectory 会动态增删 Inovker，那在之前文章中我忽略了RegistryDirectory的源码分析，在本文中来补充。

## 源码分析

### （一）AbstractDirectory

该类实现了Directory接口，

#### 1.属性

```java
// logger
private static final Logger logger = LoggerFactory.getLogger(AbstractDirectory.class);

/**
 * url对象
 */
private final URL url;

/**
 * 是否销毁
 */
private volatile boolean destroyed = false;

/**
 * 消费者端url
 */
private volatile URL consumerUrl;

/**
 * 路由集合
 */
private volatile List<Router> routers;
```

#### 2.list

```java
@Override
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    // 如果销毁，则抛出异常
    if (destroyed) {
        throw new RpcException("Directory already destroyed .url: " + getUrl());
    }
    // 调用doList来获得Invoker集合
    List<Invoker<T>> invokers = doList(invocation);
    // 获得路由集合
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && !localRouters.isEmpty()) {
        // 遍历路由
        for (Router router : localRouters) {
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    // 根据路由规则选择符合规则的invoker集合
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
    }
    return invokers;
}
```

该方法是生成invoker集合的逻辑实现。其中doList是抽象方法，交由子类来实现。

#### 3.setRouters

```java
protected void setRouters(List<Router> routers) {
    // copy list
    // 复制路由集合
    routers = routers == null ? new ArrayList<Router>() : new ArrayList<Router>(routers);
    // append url router
    // 获得路由的配置
    String routerkey = url.getParameter(Constants.ROUTER_KEY);
    if (routerkey != null && routerkey.length() > 0) {
        // 加载路由工厂
        RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);
        // 加入集合
        routers.add(routerFactory.getRouter(url));
    }
    // append mock invoker selector
    // 加入服务降级路由
    routers.add(new MockInvokersSelector());
    // 排序
    Collections.sort(routers);
    this.routers = routers;
}
```

### （二）StaticDirectory

静态 Directory 实现类，将传入的 invokers 集合，封装成静态的 Directory 对象。

```java
public class StaticDirectory<T> extends AbstractDirectory<T> {

    private final List<Invoker<T>> invokers;

    public StaticDirectory(List<Invoker<T>> invokers) {
        this(null, invokers, null);
    }

    public StaticDirectory(List<Invoker<T>> invokers, List<Router> routers) {
        this(null, invokers, routers);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers) {
        this(url, invokers, null);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers, List<Router> routers) {
        super(url == null && invokers != null && !invokers.isEmpty() ? invokers.get(0).getUrl() : url, routers);
        if (invokers == null || invokers.isEmpty())
            throw new IllegalArgumentException("invokers == null");
        this.invokers = invokers;
    }

    @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();
    }

    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        // 遍历invokers，如果有一个可用，则可用
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        super.destroy();
        // 遍历invokers，销毁所有的invoker
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        // 清除集合
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {

        return invokers;
    }

}
```

该类我就不多讲解，比较简单。

### （三）RegistryDirectory

该类继承了AbstractDirectory类，是基于注册中心的动态 Directory 实现类，会根据注册中心的推送变更 List\<Invoker\>

#### 1.属性

```java
private static final Logger logger = LoggerFactory.getLogger(RegistryDirectory.class);

/**
 * cluster实现类对象
 */
private static final Cluster cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();

/**
 * 路由工厂
 */
private static final RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getAdaptiveExtension();

/**
 * 配置规则工厂
 */
private static final ConfiguratorFactory configuratorFactory = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class).getAdaptiveExtension();

/**
 * 服务key
 */
private final String serviceKey; // Initialization at construction time, assertion not null
/**
 * 服务类型
 */
private final Class<T> serviceType; // Initialization at construction time, assertion not null
/**
 * 消费者URL的配置项 Map
 */
private final Map<String, String> queryMap; // Initialization at construction time, assertion not null
/**
 * 原始的目录 URL
 */
private final URL directoryUrl; // Initialization at construction time, assertion not null, and always assign non null value
/**
 * 服务方法集合
 */
private final String[] serviceMethods;
/**
 * 是否使用多分组
 */
private final boolean multiGroup;
/**
 * 协议
 */
private Protocol protocol; // Initialization at the time of injection, the assertion is not null
/**
 * 注册中心
 */
private Registry registry; // Initialization at the time of injection, the assertion is not null
/**
 *  是否禁止访问
 */
private volatile boolean forbidden = false;

/**
 * 覆盖目录的url
 */
private volatile URL overrideDirectoryUrl; // Initialization at construction time, assertion not null, and always assign non null value

/**
 * override rules
 * Priority: override>-D>consumer>provider
 * Rule one: for a certain provider <ip:port,timeout=100>
 * Rule two: for all providers <* ,timeout=5000>
 * 配置规则数组
 */
private volatile List<Configurator> configurators; // The initial value is null and the midway may be assigned to null, please use the local variable reference

// Map<url, Invoker> cache service url to invoker mapping.
/**
 * url与服务提供者 Invoker 集合的映射缓存
 */
private volatile Map<String, Invoker<T>> urlInvokerMap; // The initial value is null and the midway may be assigned to null, please use the local variable reference

// Map<methodName, Invoker> cache service method to invokers mapping.
/**
 * 方法名和服务提供者 Invoker 集合的映射缓存
 */
private volatile Map<String, List<Invoker<T>>> methodInvokerMap; // The initial value is null and the midway may be assigned to null, please use the local variable reference

// Set<invokerUrls> cache invokeUrls to invokers mapping.
/**
 * 服务提供者Invoker 集合缓存
 */
private volatile Set<URL> cachedInvokerUrls; // The initial value is null and the midway may be assigned to null, please use the local variable reference
```

#### 2.toConfigurators

```java
public static List<Configurator> toConfigurators(List<URL> urls) {
    // 如果为空，则返回空集合
    if (urls == null || urls.isEmpty()) {
        return Collections.emptyList();
    }

    List<Configurator> configurators = new ArrayList<Configurator>(urls.size());
    // 遍历url集合
    for (URL url : urls) {
        //如果是协议是empty的值，则清空配置集合
        if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
            configurators.clear();
            break;
        }
        // 覆盖的参数集合
        Map<String, String> override = new HashMap<String, String>(url.getParameters());
        //The anyhost parameter of override may be added automatically, it can't change the judgement of changing url
        // 覆盖的anyhost参数可以自动添加，也不能改变更改url的判断
        override.remove(Constants.ANYHOST_KEY);
        // 如果需要覆盖添加的值为0，则清空配置
        if (override.size() == 0) {
            configurators.clear();
            continue;
        }
        // 加入配置规则集合
        configurators.add(configuratorFactory.getConfigurator(url));
    }
    // 排序
    Collections.sort(configurators);
    return configurators;
}
```

该方法是处理配置规则url集合，转换覆盖url映射以便在重新引用时使用，每次发送所有规则，网址将被重新组装和计算。

#### 3.destroy

```java
@Override
public void destroy() {
    // 如果销毁了，则返回
    if (isDestroyed()) {
        return;
    }
    // unsubscribe.
    try {
        if (getConsumerUrl() != null && registry != null && registry.isAvailable()) {
            // 取消订阅
            registry.unsubscribe(getConsumerUrl(), this);
        }
    } catch (Throwable t) {
        logger.warn("unexpeced error when unsubscribe service " + serviceKey + "from registry" + registry.getUrl(), t);
    }
    super.destroy(); // must be executed after unsubscribing
    try {
        // 清空所有的invoker
        destroyAllInvokers();
    } catch (Throwable t) {
        logger.warn("Failed to destroy service " + serviceKey, t);
    }
}
```

该方法是销毁方法。

#### 4.destroyAllInvokers

```java
private void destroyAllInvokers() {
    Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
    // 如果invoker集合不为空
    if (localUrlInvokerMap != null) {
        // 遍历
        for (Invoker<T> invoker : new ArrayList<Invoker<T>>(localUrlInvokerMap.values())) {
            try {
                // 销毁invoker
                invoker.destroy();
            } catch (Throwable t) {
                logger.warn("Failed to destroy service " + serviceKey + " to provider " + invoker.getUrl(), t);
            }
        }
        // 清空集合
        localUrlInvokerMap.clear();
    }
    methodInvokerMap = null;
}
```

该方法是关闭所有的invoker服务。

#### 5.notify

```java
@Override
public synchronized void notify(List<URL> urls) {
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    // 遍历url
    for (URL url : urls) {
        // 获得协议
        String protocol = url.getProtocol();
        // 获得类别
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        // 如果是路由规则
        if (Constants.ROUTERS_CATEGORY.equals(category)
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            // 则在路由规则集合中加入
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            // 如果是配置规则，则加入配置规则集合
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            // 如果是服务提供者，则加入服务提供者集合
            invokerUrls.add(url);
        } else {
            logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
        }
    }
    // configurators
    if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
        // 处理配置规则url集合
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers
    if (routerUrls != null && !routerUrls.isEmpty()) {
        // 处理路由规则 URL 集合
        List<Router> routers = toRouters(routerUrls);
        if (routers != null) { // null - do nothing
            // 并且设置路由集合
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // merge override parameters
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        // 遍历配置规则集合 逐个进行配置
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    // providers
    // 处理服务提供者 URL 集合
    refreshInvoker(invokerUrls);
}
```

当服务有变化的时候，执行该方法。首先将url根据路由规则、服务提供者和配置规则三种类型分开，分别放入三个集合，然后对每个集合进行修改或者通知

#### 6.refreshInvoker

```java
private void refreshInvoker(List<URL> invokerUrls) {
    if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
            && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        // 设置禁止访问
        this.forbidden = true; // Forbid to access
        // methodInvokerMap 置空
        this.methodInvokerMap = null; // Set the method invoker map to null
        // 关闭所有的invoker
        destroyAllInvokers(); // Close all invokers
    } else {
        // 关闭禁止访问
        this.forbidden = false; // Allow to access
        // 引用老的 urlInvokerMap
        Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
        // 传入的 invokerUrls 为空，说明是路由规则或配置规则发生改变，此时 invokerUrls 是空的，直接使用 cachedInvokerUrls 。
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            // 否则把所有的invokerUrls加入缓存
            this.cachedInvokerUrls = new HashSet<URL>();
            this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
        }
        // 如果invokerUrls为空，则直接返回
        if (invokerUrls.isEmpty()) {
            return;
        }
        // 将传入的 invokerUrls ，转成新的 urlInvokerMap
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map
        // 转换出新的 methodInvokerMap
        Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // Change method name to map Invoker Map
        // state change
        // If the calculation is wrong, it is not processed.
        // 如果为空，则打印错误日志并且返回
        if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
            logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
            return;
        }
        // 若服务引用多 group ，则按照 method + group 聚合 Invoker 集合
        this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
        this.urlInvokerMap = newUrlInvokerMap;
        try {
            // 销毁不再使用的 Invoker 集合
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
        } catch (Exception e) {
            logger.warn("destroyUnusedInvokers error. ", e);
        }
    }
}
```

该方法是处理服务提供者 URL 集合。根据 invokerURL 列表转换为 invoker 列表。转换规则如下：

1. 如果 url 已经被转换为 invoker ，则不在重新引用，直接从缓存中获取，注意如果 url 中任何一个参数变更也会重新引用。
2. 如果传入的 invoker 列表不为空，则表示最新的 invoker 列表。
3. 如果传入的 invokerUrl 列表是空，则表示只是下发的 override 规则或 route 规则，需要重新交叉对比，决定是否需要重新引用。

#### 7.toMergeMethodInvokerMap

```java
private Map<String, List<Invoker<T>>> toMergeMethodInvokerMap(Map<String, List<Invoker<T>>> methodMap) {
    // 循环方法，按照 method + group 聚合 Invoker 集合
    Map<String, List<Invoker<T>>> result = new HashMap<String, List<Invoker<T>>>();
    // 遍历方法集合
    for (Map.Entry<String, List<Invoker<T>>> entry : methodMap.entrySet()) {
        // 获得方法
        String method = entry.getKey();
        // 获得invoker集合
        List<Invoker<T>> invokers = entry.getValue();
        // 获得组集合
        Map<String, List<Invoker<T>>> groupMap = new HashMap<String, List<Invoker<T>>>();
        // 遍历invoker集合
        for (Invoker<T> invoker : invokers) {
            // 获得url携带的组配置
            String group = invoker.getUrl().getParameter(Constants.GROUP_KEY, "");
            // 获得该组对应的invoker集合
            List<Invoker<T>> groupInvokers = groupMap.get(group);
            // 如果为空，则新创建一个，然后加入集合
            if (groupInvokers == null) {
                groupInvokers = new ArrayList<Invoker<T>>();
                groupMap.put(group, groupInvokers);
            }
            groupInvokers.add(invoker);
        }
        // 如果只有一个组
        if (groupMap.size() == 1) {
            // 返回该组的invoker集合
            result.put(method, groupMap.values().iterator().next());
        } else if (groupMap.size() > 1) {
            // 如果不止一个组
            List<Invoker<T>> groupInvokers = new ArrayList<Invoker<T>>();
            // 遍历组
            for (List<Invoker<T>> groupList : groupMap.values()) {
                // 每次从集群中选择一个invoker加入groupInvokers
                groupInvokers.add(cluster.join(new StaticDirectory<T>(groupList)));
            }
            // 加入需要返回的集合
            result.put(method, groupInvokers);
        } else {
            result.put(method, invokers);
        }
    }
    return result;
}
```

该方法是通过按照 method + group 来聚合 Invoker 集合。

#### 8.toRouters

```java
private List<Router> toRouters(List<URL> urls) {
    List<Router> routers = new ArrayList<Router>();
    // 如果为空，则直接返回空集合
    if (urls == null || urls.isEmpty()) {
        return routers;
    }
    if (urls != null && !urls.isEmpty()) {
        // 遍历url集合
        for (URL url : urls) {
            // 如果为empty协议，则直接跳过
            if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
                continue;
            }
            // 获得路由规则
            String routerType = url.getParameter(Constants.ROUTER_KEY);
            if (routerType != null && routerType.length() > 0) {
                // 设置协议
                url = url.setProtocol(routerType);
            }
            try {
                // 获得路由
                Router router = routerFactory.getRouter(url);
                if (!routers.contains(router))
                    // 加入集合
                    routers.add(router);
            } catch (Throwable t) {
                logger.error("convert router url to router error, url: " + url, t);
            }
        }
    }
    return routers;
}
```

该方法是对url集合进行路由的解析，返回路由集合。

#### 9.toInvokers

```java
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
    Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
    // 如果为空，则返回空集合
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    Set<String> keys = new HashSet<String>();
    // 获得引用服务的协议
    String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
    // 遍历url
    for (URL providerUrl : urls) {
        // If protocol is configured at the reference side, only the matching protocol is selected
        // 如果在参考侧配置协议，则仅选择匹配协议
        if (queryProtocols != null && queryProtocols.length() > 0) {
            boolean accept = false;
            // 分割协议
            String[] acceptProtocols = queryProtocols.split(",");
            // 遍历协议
            for (String acceptProtocol : acceptProtocols) {
                // 如果匹配，则是接受的协议
                if (providerUrl.getProtocol().equals(acceptProtocol)) {
                    accept = true;
                    break;
                }
            }
            if (!accept) {
                continue;
            }
        }
        // 如果协议是empty，则跳过
        if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
            continue;
        }
        // 如果该协议不是dubbo支持的，则打印错误日志，跳过
        if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
            logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost()
                    + ", supported protocol: " + ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
            continue;
        }
        // 合并url参数
        URL url = mergeUrl(providerUrl);

        String key = url.toFullString(); // The parameter urls are sorted
        if (keys.contains(key)) { // Repeated url
            continue;
        }
        // 添加到keys
        keys.add(key);
        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        // 如果服务端 URL 发生变化，则重新 refer 引用
        Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
        Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
        if (invoker == null) { // Not in the cache, refer again
            try {
                // 判断是否开启
                boolean enabled = true;
                // 获得enabled配置
                if (url.hasParameter(Constants.DISABLED_KEY)) {
                    enabled = !url.getParameter(Constants.DISABLED_KEY, false);
                } else {
                    enabled = url.getParameter(Constants.ENABLED_KEY, true);
                }
                // 若开启，创建 Invoker 对象
                if (enabled) {
                    invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            // 添加到 newUrlInvokerMap 中
            if (invoker != null) { // Put new invoker in cache
                newUrlInvokerMap.put(key, invoker);
            }
        } else {
            newUrlInvokerMap.put(key, invoker);
        }
    }
    // 清空 keys
    keys.clear();
    return newUrlInvokerMap;
}
```

该方法是将url转换为调用者，如果url已被引用，则不会重新引用。

#### 10.mergeUrl

```java
private URL mergeUrl(URL providerUrl) {
    // 合并消费端参数
    providerUrl = ClusterUtils.mergeUrl(providerUrl, queryMap); // Merge the consumer side parameters

    // 合并配置规则
    List<Configurator> localConfigurators = this.configurators; // local reference
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        for (Configurator configurator : localConfigurators) {
            providerUrl = configurator.configure(providerUrl);
        }
    }

    // 不检查连接是否成功，总是创建 Invoker
    providerUrl = providerUrl.addParameter(Constants.CHECK_KEY, String.valueOf(false)); // Do not check whether the connection is successful or not, always create Invoker!

    // The combination of directoryUrl and override is at the end of notify, which can't be handled here
    // 合并提供者参数，因为 directoryUrl 与 override 合并是在 notify 的最后，这里不能够处理
    this.overrideDirectoryUrl = this.overrideDirectoryUrl.addParametersIfAbsent(providerUrl.getParameters()); // Merge the provider side parameters

    // 1.0版本兼容
    if ((providerUrl.getPath() == null || providerUrl.getPath().length() == 0)
            && "dubbo".equals(providerUrl.getProtocol())) { // Compatible version 1.0
        //fix by tony.chenl DUBBO-44
        String path = directoryUrl.getParameter(Constants.INTERFACE_KEY);
        if (path != null) {
            int i = path.indexOf('/');
            if (i >= 0) {
                path = path.substring(i + 1);
            }
            i = path.lastIndexOf(':');
            if (i >= 0) {
                path = path.substring(0, i);
            }
            providerUrl = providerUrl.setPath(path);
        }
    }
    return providerUrl;
}
```

该方法是合并 URL 参数，优先级为配置规则 > 服务消费者配置 > 服务提供者配置.

#### 11.toMethodInvokers

```java
private Map<String, List<Invoker<T>>> toMethodInvokers(Map<String, Invoker<T>> invokersMap) {
    Map<String, List<Invoker<T>>> newMethodInvokerMap = new HashMap<String, List<Invoker<T>>>();
    // According to the methods classification declared by the provider URL, the methods is compatible with the registry to execute the filtered methods
    List<Invoker<T>> invokersList = new ArrayList<Invoker<T>>();
    if (invokersMap != null && invokersMap.size() > 0) {
        // 遍历调用者列表
        for (Invoker<T> invoker : invokersMap.values()) {
            String parameter = invoker.getUrl().getParameter(Constants.METHODS_KEY);
            // 按服务提供者 URL 所声明的 methods 分类
            if (parameter != null && parameter.length() > 0) {
                // 分割参数得到方法集合
                String[] methods = Constants.COMMA_SPLIT_PATTERN.split(parameter);
                if (methods != null && methods.length > 0) {
                    // 遍历方法集合
                    for (String method : methods) {
                        if (method != null && method.length() > 0
                                && !Constants.ANY_VALUE.equals(method)) {
                            // 获得该方法对应的invoker，如果为空，则创建
                            List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
                            if (methodInvokers == null) {
                                methodInvokers = new ArrayList<Invoker<T>>();
                                newMethodInvokerMap.put(method, methodInvokers);
                            }
                            methodInvokers.add(invoker);
                        }
                    }
                }
            }
            invokersList.add(invoker);
        }
    }
    // 根据路由规则，匹配合适的 Invoker 集合。
    List<Invoker<T>> newInvokersList = route(invokersList, null);
    // 添加 `newInvokersList` 到 `newMethodInvokerMap` 中，表示该服务提供者的全量 Invoker 集合
    newMethodInvokerMap.put(Constants.ANY_VALUE, newInvokersList);
    if (serviceMethods != null && serviceMethods.length > 0) {
        // 循环方法，获得每个方法路由匹配的invoker集合
        for (String method : serviceMethods) {
            List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
            if (methodInvokers == null || methodInvokers.isEmpty()) {
                methodInvokers = newInvokersList;
            }
            newMethodInvokerMap.put(method, route(methodInvokers, method));
        }
    }
    // sort and unmodifiable
    // 循环排序每个方法的 Invoker 集合，排序
    for (String method : new HashSet<String>(newMethodInvokerMap.keySet())) {
        List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
        Collections.sort(methodInvokers, InvokerComparator.getComparator());
        newMethodInvokerMap.put(method, Collections.unmodifiableList(methodInvokers));
    }
    // 设置为不可变
    return Collections.unmodifiableMap(newMethodInvokerMap);
}
```

该方法是将调用者列表转换为与方法的映射关系。

#### 12.destroyUnusedInvokers

```java
private void destroyUnusedInvokers(Map<String, Invoker<T>> oldUrlInvokerMap, Map<String, Invoker<T>> newUrlInvokerMap) {
    if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
        destroyAllInvokers();
        return;
    }
    // check deleted invoker
    // 记录已经删除的invoker
    List<String> deleted = null;
    if (oldUrlInvokerMap != null) {
        Collection<Invoker<T>> newInvokers = newUrlInvokerMap.values();
        // 遍历旧的invoker集合
        for (Map.Entry<String, Invoker<T>> entry : oldUrlInvokerMap.entrySet()) {
            if (!newInvokers.contains(entry.getValue())) {
                if (deleted == null) {
                    deleted = new ArrayList<String>();
                }
                // 加入该invoker
                deleted.add(entry.getKey());
            }
        }
    }

    if (deleted != null) {
        // 遍历需要删除的invoker  url集合
        for (String url : deleted) {
            if (url != null) {
                // 移除该url
                Invoker<T> invoker = oldUrlInvokerMap.remove(url);
                if (invoker != null) {
                    try {
                        // 销毁invoker
                        invoker.destroy();
                        if (logger.isDebugEnabled()) {
                            logger.debug("destroy invoker[" + invoker.getUrl() + "] success. ");
                        }
                    } catch (Exception e) {
                        logger.warn("destroy invoker[" + invoker.getUrl() + "] faild. " + e.getMessage(), e);
                    }
                }
            }
        }
    }
}
```

该方法是销毁不再使用的 Invoker 集合。

#### 13.doList

```java
@Override
public List<Invoker<T>> doList(Invocation invocation) {
    // 如果禁止访问，则抛出异常
    if (forbidden) {
        // 1. No service provider 2. Service providers are disabled
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
            "No provider available from registry " + getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +  NetUtils.getLocalHost()
                    + " use dubbo version " + Version.getVersion() + ", please check status of providers(disabled, not registered or in blacklist).");
    }
    List<Invoker<T>> invokers = null;
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        // 获得方法名
        String methodName = RpcUtils.getMethodName(invocation);
        // 获得参数名
        Object[] args = RpcUtils.getArguments(invocation);
        if (args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            // 根据第一个参数枚举路由
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // The routing can be enumerated according to the first parameter
        }
        if (invokers == null) {
            // 根据方法名获得 Invoker 集合
            invokers = localMethodInvokerMap.get(methodName);
        }
        if (invokers == null) {
            // 使用全量 Invoker 集合。例如，`#$echo(name)` ，回声方法
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        if (invokers == null) {
            // 使用 `methodInvokerMap` 第一个 Invoker 集合。防御性编程。
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

该方法是通过会话域来获得Invoker集合。

## 后记

> 该部分相关的源码解析地址：https://github.com/CrazyHZM/incubator-dubbo/tree/analyze-2.6.x/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/directory

该文章讲解了集群中关于directory实现的部分，关键是RegistryDirectory，其中涉及到众多方法，需要好好品味。接下来我将开始对集群模块关于Loadbalance部分进行讲解。