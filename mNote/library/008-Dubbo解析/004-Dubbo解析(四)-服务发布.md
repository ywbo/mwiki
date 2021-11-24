# Dubbo解析(四)-服务发布

RPC处理的就是远程服务的调用，一个消费者通过网络调用一个提供者。而单消费者和单提供者解决不了项目的高负载，延伸出的RPC框架增加了注册中心，从而对多个消费者和提供者进行协调。而消费者、提供者和注册中心的交互也就是服务的发布和引用。

Dubbo的服务与注册中心之间的交互图如下：

![img](https://oscimg.oschina.net/oscnet/cbdd657d246ff48aae96872fd73be44e541.jpg)

Dubbo的消费者和提供者的交互过程：

1. 提供者向注册中心注册，并暴露本地服务
2. 消费者向注册中心注册，并订阅提供者列表
3. 消费者获取提供者列表，
4. 消费者按照负载均衡选择一个提供者，直接调用其服务

这一篇主要介绍第一个操作。

Dubbo的提供者([dubbo:service](dubbo:service))转换成ServiceBean将服务发布出去，ServiceBean继承ServiceConfig抽象配置类，同时实现多个Spring相关的容器接口。

![img](https://oscimg.oschina.net/oscnet/b0e713c10cff3658881348563b35f543752.jpg)

当Spring启动后触发ContextRefreshedEvent事件，会调用onApplicationEvent方法

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {java
        export();
    }
}
```

从而执行export方法。export方法定义在ServiceConfig类中，经过一系列的校验和配置组装，最终调用doExportUrls，发布所有的url。

## ServiceConfig.doExportUrls执行服务export

对于dubbo的配置，简单的介绍一下，如同[dubbo:service](dubbo:service)对应于ServiceConfig一样,dubbo的xml标签都对应于一个配置类

[dubbo:application](dubbo:application) : ApplicationConfig，应用配置类

[dubbo:registry](dubbo:registry) : RegistryConfig，注册中心配置类

[dubbo:protocol](dubbo:protocol) : ProtocolConfig，服务协议配置类

而doExportUrls方法分成两部分

1. 获取所有注册中心的URL
2. 遍历所有协议ProtocolConfig，将每个protocol发布到所有注册中心上

```java
private void doExportUrls() {
	// 1.获取注册中心URL
    List<URL> registryURLs = loadRegistries(true);
    // 2.遍历所有协议，export protocol并注册到所有注册中心
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

### loadRegistries获取注册中心URL

首先执行checkRegistry，判断是否有配置注册中心，如果没有，则从默认配置文件dubbo.properties中读取dubbo.registry.address组装成RegistryConfig。

```java
AbstractInterfaceConfig.checkRegistry()

if (registries == null || registries.isEmpty()) {
    String address = ConfigUtils.getProperty("dubbo.registry.address");
    if (address != null && address.length() > 0) {
        registries = new ArrayList<RegistryConfig>();
        String[] as = address.split("\\s*[|]+\\s*");
        for (String a : as) {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setAddress(a);
            registries.add(registryConfig);
        }
    }
}
```

然后根据RegistryConfig的配置，组装registryURL，形成的URL格式如下

```java
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=1528&qos.port=22222&registry=zookeeper&timestamp=1530743640901
```

这个URL表示它是一个registry协议(RegistryProtocol)，地址是注册中心的ip:port，服务接口是RegistryService，registry的类型为zookeeper。

### doExportUrlsFor1Protocol发布服务和注册

因为dubbo支持多协议配置，对于每个ProtocolConfig配置，组装protocolURL，注册到每个注册中心上。

首先根据ProtocolConfig构建协议的URL

1. 设置side=provider,dubbo={dubboVersion},timestamp=时间戳,pid=进程id
2. 从application，module，provider，protocol配置中添加URL的parameter
3. 遍历[dubbo:service](dubbo:service)下的[dubbo:method](dubbo:method)及[dubbo:argument](dubbo:argument)添加URL的parameter
4. 判断是否泛型暴露，设置generic，及methods=*，否则获取服务接口的所有method
5. 获取host及port，host及port都是通过多种方式获取，保证最终不为空

```java
// 获取绑定的ip，1从系统配置获取2从protocolConfig获取3从providerConfig获取
// 4获取localhost对应得ipv45连接注册中心获取6直接获取localhost对应的ip(127.0.0.1)
String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
// 获取绑定的port，1从系统配置2从protocolConfig3从providerConfig4从defaultPort之上随机取可用的
Integer port = this.findConfigedPorts(protocolConfig, name, map);
```

最终构建URL对象

```java
// 创建protocol export url
URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
```

构建出的protocolURL格式如下

```xml
dubbo://192.168.199.180:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.199.180&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=5744&qos.port=22222&side=provider&timestamp=1530746052546
```

这个URL表示它是一个dubbo协议(DubboProtocol)，地址是当前服务器的ip，端口是要暴露的服务的端口号，可以从[dubbo:protocol](dubbo:protocol)配置，服务接口为[dubbo:service](dubbo:service)配置发布的接口。

然后从url中获取scope属性，如果scope!="none"则发布服务，默认scope为null。如果scope不为none，判断是否为local或remote，从而发布Local服务或Remote服务，默认两个都发布。

这里的Local服务只是injvm的服务，提供一种消费者和提供者都在一个jvm内的调用方式。

主要来看Remote服务，遍历所有的registryURL，执行以下操作:

1. 如果配置了monitor，则组装monitorURL，并给registryURL设置MONITOR_KEY=monitorURL
2. 给registryURL设置EXPORT_KEY为上面构建的protocolURL
3. 根据实现对象，服务接口Class和registryuRL通过ProxyFactory获取代理Invoker(继承于AbstractProxyInvoker)，详见上一篇[动态代理和包装](https://my.oschina.net/u/2377110/blog/1835417)。
4. 将Invoker对象和ServiceConfig组装成MetaDataInvoker，通过protocol.export(invoker)暴露出去。

```java
for (URL registryURL : registryURLs) {
    url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
    // 组装监控URL
    URL monitorUrl = loadMonitor(registryURL);
    if (monitorUrl != null) {
        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
    }
    // 以registryUrl创建Invoker
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
    // 包装Invoker和ServiceConfig
    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

    // 以RegistryProtocol为主，注册和订阅注册中心，并暴露本地服务端口
    Exporter<?> exporter = protocol.export(wrapperInvoker);
    exporters.add(exporter);
}
```

根据SPI机制，这里的procotol.export执行时，从Invoker获取URL的protocol为registry,由RegistryProtocol处理export过程。

## RegistryProtocol.export暴露服务

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // export dubbo invoker
	// 发布本地invoker，暴露本地服务，打开服务器端口
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    // 获取registryUrl，protocol从registryUrl的registry参数获取
    // 修改registryUrl的protocol，并移除registryUrl的registry参数
    // 即将registry://改成类似zookeeper://
    URL registryUrl = getRegistryUrl(originInvoker);
    
    // 根据url从registryFactory中获取对应的registry
    final Registry registry = getRegistry(originInvoker);
    // 获取要注册的providerUrl
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    // 判断是否要将provider url注册到注册中心
    boolean register = registedProviderUrl.getParameter("register", true);

    // 本地提供者和消费者注册表
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

    if (register) {
    	// 向注册中心注册providerUrl
        register(registryUrl, registedProviderUrl);
        // 本地注册表设置此provider注册完成
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // 组装提供者节点的url
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    // 创建Override监听器
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    // 添加到监听器列表
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 向注册中心订阅提供者url节点
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    // 返回一个全新的Exporter
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
}
```

1. 从Invoker中获取providerURL，同传入的Invoker对象组装成InvokerDelegete，通过protocol.export根据providerURL(一般为dubbo协议)暴露服务，打开服务器端口，获得Exporter缓存到本地。

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
	// 获取cache key
    String key = getCacheKey(originInvoker);
    // 是否存在已绑定的exporter
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
            	// 封装invoker和providerUrl
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                // export provider invoker，Protocol的具体实现是由Url中的protocol属性决定的
                // 封装创建出的exporter和origin invoker
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return exporter;
}
```

1. 修改registryUrl，将其协议由registry改为具体的注册中心协议，即[registry://改为zookeeper://。](registry://改为zookeeper://。)

```java
private URL getRegistryUrl(Invoker<?> originInvoker) {
    URL registryUrl = originInvoker.getUrl();
    if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
        String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
        registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
    }
    return registryUrl;
}
```

1. 根据registryUrl从RegistryFactory根据SPI机制获取具体的Registry

```java
private Registry getRegistry(final Invoker<?> originInvoker) {
    URL registryUrl = getRegistryUrl(originInvoker);
    return registryFactory.getRegistry(registryUrl);
}
```

1. 获取要注册的providerUrl，移除不需要在注册中心看到的providerUrl中部分参数

```java
private URL getRegistedProviderUrl(final Invoker<?> originInvoker) {
    URL providerUrl = getProviderUrl(originInvoker);
    //The address you see at the registry
    final URL registedProviderUrl = providerUrl.removeParameters(getFilteredKeys(providerUrl))
            .removeParameter(Constants.MONITOR_KEY)
            .removeParameter(Constants.BIND_IP_KEY)
            .removeParameter(Constants.BIND_PORT_KEY)
            .removeParameter(QOS_ENABLE)
            .removeParameter(QOS_PORT)
            .removeParameter(ACCEPT_FOREIGN_IP);
    return registedProviderUrl;
}
```

得到的providerUrl格式如下

```java
dubbo://192.168.199.180:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=3012&side=provider&timestamp=1530749502888
```

1. 将providerUrl注册到注册中心，如果注册中心为zookeeper，则使用ZookeeperRegistry注册器进行注册

```java
public void register(URL registryUrl, URL registedProviderUrl) {
	// registryFactory为动态生成类RegistryFactory$Adapative，根据url.getProtocol获取实际的RegistryFactory
	// 如zookeeper对应的是ZookeeperRegistryFactory，获取的Registry对象为ZookeeperRegistry
    Registry registry = registryFactory.getRegistry(registryUrl);
    registry.register(registedProviderUrl);
}
```

1. 从registryUrl组装overrideSubscribeUrl，并构建OverrideListener，向注册中心订阅overrideSubscribeUrl，用于当配置数据变化时，触发overrideListener的notify方法通知提供者重新暴露服务。
2. 将本地暴露的exporter，传入的参数originInvoker以及overrideSubscribeUrl和registedProviderUrl封装成新的DestroyableExporter返回，供消费者调用时获取。

对于第1步中的providerUrl构建的Invoker，通过protocol.export暴露本地服务，一般都是dubbo协议，从而使用DubboProtocol暴露服务。

## DubboProtocol暴露本地服务

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	// 获取提供者url
    URL url = invoker.getUrl();

    // export service.
    // 从url获取service key > group/service:version:port
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    // 对本地存根stub及回调的初始处理
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

    // 打开服务器端口，暴露本地服务
    openServer(url);
    // 优化序列化操作
    optimizeSerialization(url);
    return exporter;
}
```

1. 从Invoker获取providerUrl，构建serviceKey(group/service:version:port)，构建DubboExporter并以serviceKey为key放入本地map缓存
2. 处理url携带的本地存根和callback回调
3. 根据url打开服务器端口，暴露本地服务。先以url.getAddress为key查询本地缓存serverMap获取ExchangeServer，如果不存在，则通过createServer创建。

```java
private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client can export a service which's only for server to invoke
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            serverMap.put(key, createServer(url));
        } else {
            // server supports reset, use together with override
            server.reset(url);
        }
    }
}
```

1. createServer方法，设置心跳时间，判断url中的传输方式(key=server,对应Transporter服务)是否支持，设置codec=dubbo，最后根据url和ExchangeHandler对象绑定server返回，**这里的ExchangeHandler非常重要，它就是消费方调用时，底层通信层回调的Handler，从而获取包含实际Service实现的Invoker执行器**，它是定义在DubboProtocol类中的ExchangeHandlerAdapter内部类。

```java
private ExchangeServer createServer(URL url) {
    // send readonly event when server closes, it's enabled by default
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // enable heartbeat by default
    // 设置心跳时间
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    // 获取server方式，netty or mina
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);
    // 设置codec=dubbo
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        // 绑定url和ExchangeHandler，返回ExchangeServer
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

1. 返回DubboExporter对象

## 服务发布序列图及活动图

![img](https://oscimg.oschina.net/oscnet/099ddd313144115ee87a0a89a3f80afb874.jpg)

![img](https://oscimg.oschina.net/oscnet/dd34730fe174bc9a90db04c425ede243581.jpg)

参考：[https://blog.csdn.net/quhongwei_zhanqiu/article/details/41651205](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fblog.csdn.net%2Fquhongwei_zhanqiu%2Farticle%2Fdetails%2F41651205)