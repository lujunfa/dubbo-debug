# 							dubbo 服务提供者和消费者启动过程源码走读	



dubbo自定义标签的命名空间是***DubboNamespaceHandler***，自定义标签解析从***DubboBeanDefinitionParser***

![image-20200917200535625](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200917200535625.png)

开始解析。

服务提供者端***ServiceBean***在Spring容器启动后会监听容器刷新事件，然后开始导出服务。

![image-20200921102130762](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200921102130762.png)

导出服务的主要逻辑在父类***ServiceConfig***中，首先加载配置的注册中心地址，然后根据配置的协议列表循环导出服务到注册中心中。

![image-20200921103456287](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200921103456287.png)

```java
 private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        //****信息填充部分代码省略
        // 导出服务开始
        String contextPath = protocolConfig.getContextpath();
        if ((contextPath == null || contextPath.length() == 0) && provider != null) {
            contextPath = provider.getContextpath();
        }
        // 获得host,port,url
        String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
        URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(Constants.SCOPE_KEY);
        // 如果 scope = none，则什么都不做
        if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

            // scope != remote，导出本地服务
            if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            //scope != local，导出远程服务
            if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (registryURLs != null && !registryURLs.isEmpty()) {
                    for (URL registryURL : registryURLs) {
                        url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        }

                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(Constants.PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                        }

                        // 为服务提供类(ref)生成 Invoker
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        // 导出服务，并生成 Exporter
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            }
        }
        this.urls.add(url);
    }
```

导出服务时，dubbo会先根据JVM协议导出本地服务

```java
@SuppressWarnings({"unchecked", "rawtypes"})
    private void exportLocal(URL url) {
        if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
            URL local = URL.valueOf(url.toFullString())
                    .setProtocol(Constants.LOCAL_PROTOCOL)
                    .setHost(LOCALHOST)
                    .setPort(0);
            StaticContext.getContext(Constants.SERVICE_IMPL_CLASS).put(url.getServiceKey(), getServiceClass(ref));
            Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
            exporters.add(exporter);
            logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
        }
    }
```

经过层层协议包装器的包装，最终返回一个InJvm导出器保存在***ServiceConfig***实例变量**exporters**中。

![image-20200921115045313](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200921115045313.png)

然后远程导出服务，即将服务导出到注册中心。获取**Invoker**是最终使用JavaAssist技术将一个**Wrapper**类

封装进***AbstractProxyInvoker***进行代理的。

![image-20200921150807372](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200921150807372.png)

```java
 @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 为目标类创建 Wrapper
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        // 创建匿名 Invoker 类对象，并实现 doInvoke 方法。
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```



​												**RegistryProtocol导出服务逻辑**

```java
 @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 导出服务，启动本地服务器
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
        //本地服务器启动后，将地址注册到注册中心，例如Zk中间件
        // 获取注册中心 URL，以 zookeeper 注册中心为例，得到的示例 URL 如下：
        // zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F172.17.48.52%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider
        URL registryUrl = getRegistryUrl(originInvoker);

        // 根据 URL 加载 Registry 实现类，比如 ZookeeperRegistry
        final Registry registry = getRegistry(originInvoker);
        // 获取已注册的服务提供者 URL，比如：
        // dubbo://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
        final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

        // 获取 register 参数
        boolean register = registeredProviderUrl.getParameter("register", true);
        // 向服务提供者与消费者注册表中注册服务提供者
        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);
        // 根据 register 的值决定是否注册服务
        if (register) {
            // 向注册中心注册服务
            register(registryUrl, registeredProviderUrl);
            //标识该服务提供者已经注册
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
        }

        // 获取订阅 URL，比如：
        // provider://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?category=configurators&check=false&anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
        // 创建监听器
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        // 向注册中心进行订阅
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        // 创建并返回 DestroyableExporter
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
    }
```

​			

ZK注册中心的订阅监听逻辑。	

```java
//ZookeeperRegistry

@Override
    protected void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        @Override
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            for (String child : currentChilds) {
                                child = URL.decode(child);
                                if (!anyServices.contains(child)) {
                                    anyServices.add(child);
                                    subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                                }
                            }
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (services != null && !services.isEmpty()) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<URL>();
                //提供者：订阅configurators节点，处理服务治理
                //消费者：从 providers（服务提供）、configurators（服务治理）、routers（路由配置）三个节点查找符合consumer调用接口的信息
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        // 创建子节点监听器，监听 /configurators 节点下 子节点变化
                        listeners.putIfAbsent(listener, new ChildListener() {
                            @Override
                            public void childChanged(String parentPath, List<String> currentChilds) {
                                ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                            }
                        });
                        zkListener = listeners.get(listener);
                    }
                    //创建节点
                    zkClient.create(path, false);
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        // 将consumer调用接口信息与注册中心的 providers、configurators、routers 三个节点的信息，进行对比判断是否匹配（interface、gourp、version等）
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                //告知各监听器节点配置变更
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

​										



​											**doLocalExport方法调用  Dubbo协议导出堆栈**

![image-20200921154555623](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200921154555623.png)

```java

//dubbo 协议服务导出方法
@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // 获取 host:port，并将其作为服务器实例的 key，用于标识当前的服务器实例
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        // 将 <key, exporter> 键值对放入缓存中
        exporterMap.put(key, exporter);

        // 本地存根相关代码
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        // 启动服务器
        openServer(url);
        //自定义序列化协议
        optimizeSerialization(url);
        return exporter;
    }
```







对于服务消费者端对应的Bean是***ReferenceBean***，





![image-20200917201150021](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200917201150021.png)

由于***ReferenceBean***实现了***FactoryBean***了接口，所以Spring会调用***getObject***这个方法获取对应服务提供者对象的代理类，然后会调用从***ReferenceConfig***继承来的***get***方法，继而触发继承的初始化方法。

![image-20200917201933416](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200917201933416.png)

```java
@SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    if (isInjvm() == null) {
        // 根据 url 的协议、scope 以及 injvm 等参数检测是否需要本地引用
        // 比如如果用户显式配置了 scope=local，此时 isInjvmRefer 返回 true
        if (url != null && url.length() > 0) { // if a url is specified, don't do local reference
            isJvmRefer = false;
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            // by default, reference local service if there is
            isJvmRefer = true;
        } else {
            ******
         // 单个注册中心或服务提供者(服务直连，下同)
            if (urls.size() == 1) {
                // 调用 RegistryProtocol 的 refer 构建 Invoker 实例
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
```

由RegistryProtocol 协议对象生成Invoker

![image-20200918102549539](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200918102549539.png)

```java
 private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // 创建 RegistryDirectory 实例
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        // 设置注册中心和协议
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        // 生成服务消费者链接
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            URL registeredConsumerUrl = getRegisteredConsumerUrl(subscribeUrl, url);
            // 注册服务消费者，在 consumers 目录下创建节点
            registry.register(registeredConsumerUrl);
            directory.setRegisteredConsumerUrl(registeredConsumerUrl);
        }
        // 订阅 providers、configurators、routers 等节点数据
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                Constants.PROVIDERS_CATEGORY
                        + "," + Constants.CONFIGURATORS_CATEGORY
                        + "," + Constants.ROUTERS_CATEGORY));

        // 一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个 Invoker(加入集群路由)
        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```

```java
//如果服务提供者是集群部署，则需要将多个服务提供者合并为一个 Invoker(加入集群路由
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```

init方法最后创建一个包含可***MockClusterInvoker***实例对象的目标接口的代理实例。

![image-20200918112916948](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200918112916948.png)



方法调用时，将方法名和参数封装到***RpcInvocation***中，然后由***MockClusterInvoker***的**Invoke**执行，但是真正执行的是**FailOverClusterInvoker**继承**AbstractClusterInvoker**的**invoke**方法。

![image-20200918114734670](https://github.com/lujunfa/dubbo-debug/blob/master/img/dubbo/image-20200918114734670.png)

```java
@Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();
        //负载均衡器
        LoadBalance loadbalance = null;

        // binding attachments into invocation.
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addAttachments(contextAttachments);
        }

        // 列举 Invoker
        List<Invoker<T>> invokers = list(invocation);
        if (invokers != null && !invokers.isEmpty()) {
            // 加载 LoadBalance，根据URL中的以方法名为key指定负载均衡策略，没有的话则选择默认的随机策略
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
        }
        //如果是异步，则保存InvocaionId
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        // 调用 doInvoke 进行后续操作
        return doInvoke(invocation, invokers, loadbalance);
    }

```

在获取**Invoker**列表时，**AbstractDirectory**的**list**还会走路由逻辑。



```java
@Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
        // 调用 doList 方法列举 Invoker，doList 是模板方法，由子类实现
        List<Invoker<T>> invokers = doList(invocation);
        // 获取路由 Router 列表，有TagRouter和MockInvokersSelector等几种
        List<Router> localRouters = this.routers;
        if (localRouters != null && !localRouters.isEmpty()) {
            for (Router router : localRouters) {
                try {
                    if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                        // 进行服务路由
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

```java
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;

    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }

    
    //包装过滤器链
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        //由过滤器执行Invocation
                        return filter.invoke(next, invocation);
                    }

                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }
    
    
public class FutureFilter implements Filter {

    protected static final Logger logger = LoggerFactory.getLogger(FutureFilter.class);

    @Override
    public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
        //判断是否需要异步调用
        final boolean isAsync = RpcUtils.isAsync(invoker.getUrl(), invocation);

        fireInvokeCallback(invoker, invocation);
        // need to configure if there's return value before the invocation in order to help invoker to judge if it's
        // necessary to return future.
        Result result = invoker.invoke(invocation);
        if (isAsync) {
            asyncCallback(invoker, invocation);
        } else {
            syncCallback(invoker, invocation, result);
        }
        return result;
    }

```

经过**FutureFilter,MonitorFilter**等多层Filter过滤后，调用**DubboInvoker**的doInvoke

```java
//类DubboInvoker 
@Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
		//各个客户端，默认是netty
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            if (isOneway) { // 异步，无返回值
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) { // 异步，有返回值
                ResponseFuture future = currentClient.request(inv, timeout);
                //填充Future，在FutrueFilter中asyncCallback方法会异步回调回去Future中的结果
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else { // 默认：异步变同步
                //DefaultFuture有一个Map类变量用来保存以requestId为key，Future为Value，将所有的异步请求都保存在这个map中，然后某次请求的响应结果一到达就设置这次请求对应Futrue的Response,并标记为已结束，，否则为未结束，请求一直阻塞直到超时或得到响应。
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

