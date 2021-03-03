



## 一 Dubbo调用链

![](..\images\dubbo\dubbo-extension.jpg)

## 各层说明

- **config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类。
- **proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`，**Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。**  。
- **registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`。
- **cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`。
- **monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`。
- **protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`。
- **exchange 信息交换层**：封装请求响应模式，同步转异步，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`。
- **transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`。
- **serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`。



## 二 Dubbo 服务暴露

dubbo中服务暴露的大致流程为：加载检测配置，暴露服务（本地和远程），注册至注册中心。

```java
public synchronized void export() {

    // 配置元数据的初始化赋值

    if (shouldDelay()) {
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }
	
    // 服务暴露之后的一些处理
    exported();
}

protected synchronized void doExport() {
    // 暴露url
    doExportUrls();
}

private void doExportUrls() {
    ServiceRepository repository = ApplicationModel.getServiceRepository();
    ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
    repository.registerProvider(getUniqueServiceName(), ref, serviceDescriptor, this, serviceMetadata);
	// 解析获取需要注册的url
    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);
	// 根据配置的协议进行暴露
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig)
                                      .map(p -> p + "/" + path)
                                      .orElse(path), group, version);
        // In case user specified path, register service one more time to map it to path.
        repository.registerService(pathKey, interfaceClass);
        // TODO, uncomment this line once service key is unified
        serviceMetadata.setServiceKey(pathKey);
        // 根据协议进行暴露
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

解析出来的URL地址如图，基于扩展点自适应机制，通过 URL 的 `registry://` 协议头识别，就会调用 `RegistryProtocol` 的 `export()` 方法，将 `export` 参数中的提供者 URL，先注册到注册中心。

`ExtensionLoader` 注入的依赖扩展点是一个 `Adaptive` 实例，直到扩展点方法执行时才决定调用是哪一个扩展点实现。

![](..\images\dubbo\registryURLs.png)



> #### 根据协议暴露服务

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    // 获取协议配置，默认为dubbo
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }
    
	/**
	 * 构建ServiceConfig和AbstractConfig一些属性
	 */

    // 开始暴露服务
    String host = findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = findConfigedPorts(protocolConfig, name, map);
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

    /**
     * 处理自定义的配置器
     */
    
    String scope = url.getParameter(SCOPE_KEY);
    // 此时scope为null
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

        // 没有配置远程的时候，进行本地暴露
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // 没有配置scope，也会走入这个分支
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            if (CollectionUtils.isNotEmpty(registryURLs)) {
                for (URL registryURL : registryURLs) {
                    // 协议是injvm跳过
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }
                    url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                    // 处理配置得监控中心
                    URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }
					// 根据接口实现，接口类型，url由javassist生成代理，转化为invoker然后做层包装
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
					
                    // 根据协议进行远程暴露（DubboProtocol#export）
                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                exporters.add(exporter);
            }
            /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
            WritableMetadataService metadataService = WritableMetadataService.getExtension(url.getParameter(METADATA_KEY, DEFAULT_METADATA_STORAGE_TYPE));
            if (metadataService != null) {
                metadataService.publishServiceDefinition(url);
            }
        }
    }
    this.urls.add(url);
}
```

最终得到的`wrapperInvoker`如图，可以看到在invoker中持有实现类的代理。

![](..\images\dubbo\wrapperInvoker.png)



> #### 暴露服务时序图

![](..\images\dubbo\export.png)



> #### 调用栈

```
export:282, DubboProtocol (org.apache.dubbo.rpc.protocol.dubbo)
// 监听和拦截器的包装
export:64, ProtocolListenerWrapper (org.apache.dubbo.rpc.protocol)
export:155, ProtocolFilterWrapper (org.apache.dubbo.rpc.protocol)
export:66, QosProtocolWrapper (org.apache.dubbo.qos.protocol)
export:-1, Protocol$Adaptive (org.apache.dubbo.rpc)
lambda$doLocalExport$2:255, RegistryProtocol (org.apache.dubbo.registry.integration)
apply:-1, 1844334363 (org.apache.dubbo.registry.integration.RegistryProtocol$$Lambda$115)
computeIfAbsent:1660, ConcurrentHashMap (java.util.concurrent)
// 本地暴露
doLocalExport:253, RegistryProtocol (org.apache.dubbo.registry.integration)
export:205, RegistryProtocol (org.apache.dubbo.registry.integration)
export:62, ProtocolListenerWrapper (org.apache.dubbo.rpc.protocol)
export:153, ProtocolFilterWrapper (org.apache.dubbo.rpc.protocol)
export:64, QosProtocolWrapper (org.apache.dubbo.qos.protocol)
export:-1, Protocol$Adaptive (org.apache.dubbo.rpc)
// 解析构建URL准备暴露
doExportUrlsFor1Protocol:492, ServiceConfig (org.apache.dubbo.config)
doExportUrls:325, ServiceConfig (org.apache.dubbo.config)
doExport:300, ServiceConfig (org.apache.dubbo.config)
export:206, ServiceConfig (org.apache.dubbo.config)
main:28, UserProvider (com.dubbo.quickstart)
```





## 三 Dubbo调用模块基本组成

dubbo调用模块核心功能是发起调用并获取结果，其体系如下：

1. **透明代理**：通过动态代理，屏蔽远程调用细节；
2. **负载均衡**：当有多个提供者时，通过负载均衡算法选择提供者；
3. **容错机制**：当服务调用失败时，采用的容错策略；
4. **调用方式：**支持同步、异步调用。

![](..\images\dubbo\dubbo-provider.png) 





> #### Dubbo 调用模块的调用栈（省略包装类）

```
// ... 发送网络请求，建立连接
// 6. 协议调用
doInvoke:96, DubboInvoker (org.apache.dubbo.rpc.protocol.dubbo)
invoke:163, AbstractInvoker (org.apache.dubbo.rpc.protocol)
// 5. 异步转同步
invoke:52, AsyncToSyncInvoker (org.apache.dubbo.rpc.protocol)
// 4. 拦截器链
invoke:89, MonitorFilter (org.apache.dubbo.monitor.support)
invoke:83, ProtocolFilterWrapper$1 (org.apache.dubbo.rpc.protocol)
invoke:51, FutureFilter (org.apache.dubbo.rpc.protocol.dubbo.filter)
invoke:69, ConsumerContextFilter (org.apache.dubbo.rpc.filter)
// 3.集群处理
doInvoke:82, FailoverClusterInvoker (org.apache.dubbo.rpc.cluster.support)
invoke:260, AbstractClusterInvoker (org.apache.dubbo.rpc.cluster.support)
intercept:47, ClusterInterceptor (org.apache.dubbo.rpc.cluster.interceptor)
invoke:92, AbstractCluster$InterceptorInvokerNode (org.apache.dubbo.rpc.cluster.support.wrapper)
// mock服务
invoke:88, MockClusterInvoker (org.apache.dubbo.rpc.cluster.support.wrapper)
// 2. 动态代理
invoke:74, InvokerInvocationHandler (org.apache.dubbo.rpc.proxy)
getUserById:-1, proxy0 (org.apache.dubbo.common.bytecode)
// 1. 客户端发起调用
main:31, UserConsumer (com.dubbo.quickstart)
```





### 1. 透明代理

```java
//org.apache.dubbo.config.ReferenceConfig#createProxy
private T createProxy(Map<String, String> map) {
    
    // 省略...

    if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
        checkRegistry();
        // 根据配置文件解析获取地址
        // 在该方法中将注册地址zookeeper://ip:port转化为了register://ip:port
        List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
        if (CollectionUtils.isNotEmpty(us)) {
            for (URL u : us) {
                URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                if (monitorUrl != null) {
                    map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                }
                urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
            }
        }
        // 省略...
    }

    if (urls.size() == 1) {
        // 根据接口类型和url地址动态生成客户端invoker
        invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
    } else {
        // 省略...
    }

	// 省略
    
    // 生成客户端的代理，默认使用的javassist，最终获取到的代理为MockClusterInvoker
    return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
}
```

最终拿到的代理类`ref`。

![](..\images\dubbo\ref.png)

