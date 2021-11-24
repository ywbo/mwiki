# Dubbo解析(五)-服务引用

上一章介绍了Dubbo的服务与注册中心交互图

![image](https://oscimg.oschina.net/oscnet/cbdd657d246ff48aae96872fd73be44e541.jpg)

交互过程如下：

1. 提供者向注册中心注册，并暴露本地服务
2. 消费者向注册中心注册，并订阅提供者列表
3. 消费者获取提供者列表，
4. 消费者按照负载均衡选择一个提供者，直接调用其服务

上一章介绍了第一个操作，即服务的发布；这一章将介绍第二、三两个操作，简单的说，就是服务的引用。

消费者引用dubbo服务，可以通过spring的xml配置

```xml
<dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>
```

*dubbo:reference*将转换成ReferenceBean，最终形成对远程提供者服务的代理，即对服务的引用。ReferenceBean继承ReferenceConfig抽象配置类，同时实现多个Spring相关的容器接口。

![img](https://oscimg.oschina.net/oscnet/bfb70abf31066a5e2732db5b11a7f9e27b2.jpg)

其中最重要的是FactoryBean接口，它是spring的一个重要的扩展方式。对于*dubbo:reference*创建的spring Bean，并不是想要ReferenceBean这个对象本身，而是想获取对远程服务的代理。通过FactoryBean.getObject()来创建基于服务接口的代理，从而将复杂的实现封装，使用户可以不关注底层的实现。

getObject方法调用ReferenceConfig的get方法，继而调用init方法初始化服务引用。init方法对消费者的配置进行校验和组装，和服务发布一样，形成创建URL的map对象，然后调用createProxy方法创建代理。

## createProxy创建代理

1. 判断是否为JVM内部引用，如果是，构建jvm引用URL, 通过protocol.refer获得invoker执行器

```java
if (isJvmRefer) {
    // 组装localhost本地URL，通过InjvmProtocol进行服务引用
    URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
    invoker = refprotocol.refer(interfaceClass, url);
    if (logger.isInfoEnabled()) {
        logger.info("Using injvm service " + interfaceClass.getName());
    }
}
```

1. 加载注册中心配置，根据RegistryConfig的配置，组装registryURL，形成的URL格式如下

```java
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1528&qos.port=22222&registry=zookeeper&timestamp=1530743640901
```

1. java如果有监控配置，在registryUrl加上MONITOR_KEY

```java
URL monitorUrl = loadMonitor(u);
if (monitorUrl != null) {
    map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
}
```

1. 将组装好的配置map以REFER_KEY为key拼接到registryUrl

```
u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map))
```

组装完的registryUrl如下:

```xml
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=4772&qos.port=33333&refer=application%3Ddemo-consumer%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D4772%26qos.port%3D33333%26register.ip%3D192.168.199.180%26side%3Dconsumer%26timestamp%3D1531358326151&registry=zookeeper&timestamp=1531358326266
```

1. 遍历registryUrl，使用**protocol.refer**方法，由**RegistryProtocol**构建基于目录服务的集群策略Invoker。对于存在多个registryUrl，设置集群策略AvailableCluster，将所有通过registryUrl返回的Invoker执行器伪装成一个Invoker

```java
List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
URL registryURL = null;
// 遍历registryUrl，创建对远程服务的引用Invoker执行器
for (URL url : urls) {
    invokers.add(refprotocol.refer(interfaceClass, url));
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        registryURL = url; // use last registry url
    }
}
// 将多个Invoker通过集群策略伪装成一个Invoker
if (registryURL != null) { // registry url is available
    // use AvailableCluster only when register's cluster is available
    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
    invoker = cluster.join(new StaticDirectory(u, invokers));
} else { // not a registry url
    invoker = cluster.join(new StaticDirectory(invokers));
}
```

1. 通过ProxyFactory.getProxy(Invoker)创建远程服务代理供消费方透明调用

```
return (T) proxyFactory.getProxy(invoker);
```

## RegistryProtocol.refer 注册与订阅

上面第5步，将registryUrl通过protocol.refer获取服务引用，根据SPI机制，会先经过ProtocolListenerWrapper， ProtocolFilterWrapper，但这两个Wrapper都过滤了注册中心Url，直接调用Registry.refer方法。

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 设置url的protocol为registry key的值
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    // 根据url的protocol从注册中心工厂返回对应的Registry
    // 如zookeeper即返回ZookeeperRegistry
    Registry registry = registryFactory.getRegistry(url);
    // 如果服务引用interface就是RegistryService，则直接返回Invoker
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // 如果存在多个group分组，使用MergeableCluster
    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                || "*".equals(group)) {
            return doRefer(getMergeableCluster(), registry, type, url);
        }
    }
    return doRefer(cluster, registry, type, url);
}
```

注册和订阅都在doRefer中完成

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
	// 创建目录服务
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    // 设置注册中心服务
    directory.setRegistry(registry);
    // 设置协议服务
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    // RegistryDirectory获取url的parameters为refer_key对应的url中的parameters
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    // 创建URL，根据不同的category_key分别作为consumerUrl(注册)和subscribeUrl(订阅)
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
    	// 向注册中心注册，url中增加category=consumers&check=false
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    // 向注册中心订阅，订阅目录为category=providers,configurators,routers，回调RegistryDirectory的notify方法
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));

    // 通过集群策略将目录服务统一暴露成一个Invoker执行器，默认集群策略为failover
    Invoker invoker = cluster.join(directory);
    // 本地注册表注册消费者
    ProviderConsumerRegTable.registerConsuemr(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

执行步骤如下：

1. 设置url的protocol为registry_key的值，根据url从RegistryFactory注册工厂返回对应的Registry，比如zookeeper protocol对应ZookeeperRegistry
2. 如果服务存在多个group分组，设置集群策略为Mergeable
3. 创建目录服务RegistryDirectory，并设置注册器和protocol服务
4. 构建consumerUrl(category=consumers)，通过注册器向注册中心注册
5. 构建subscribeUrl(category=providers,configurators,routers)，通过注册器订阅，并**回调RegistryDirectory的notify方法**
6. 通过集群策略将目录服务统一暴露成一个Invoker执行器，默认集群策略为failover

这里介绍下提供者和消费者服务在zookeeper上对应节点path：

1. 提供者向zookeeper注册服务，在节点/dubbo/com.alibaba.dubbo.demo.DemoService/providers/写下自己的URL
2. 消费者向zookeeper注册服务，在节点/dubbo/com.alibaba.dubbo.demo.DemoService/consumers/下写下自己的URL
3. 消费者向zookeeper订阅服务，订阅节点/dubbo/com.alibaba.dubbo.demo.DemoService/providers/下所有服务提供者URL地址

官方提供的zookeeper节点图：

![img](https://oscimg.oschina.net/oscnet/8990a19bf630bbd0ec488bca0e2ec93f117.jpg)

**zookeeper通过watcher机制实现对节点的监听，节点数据变化通过节点上的watcher回调客户端。dubbo的消费者订阅了提供者URL的目录节点，如果提供者节点数据变化，会主动回调NotifyListener（RegistryDirectory实现了NotifyListener）的notify方法，将节点下所有提供者的url返回，从而重新生成对提供者服务的引用的可执行invokers，供目录服务持有**。

RegistryDirectory.notify(urls)方法过滤category=providers的提供者url(invokerUrl)，然后调用refreshInvoker(invokerUrls)方法，进行基础校验后，调用toInvokers(invokerUrls)方法，通过protocol.refer方法转换成Invoker。

```java
RegistryDirectory

notify(urls) -> refreshInvoker(invokerUrls) -> toInvokers(invokerUrls) ->

invoker = new InvokerDelegate<T>(protocol.refer(serviceType, url), url, providerUrl);
```

这里的protocol根据providerUrl的protocol基于SPI机制获取，一般就是DubboProtocol。因而最终形成对提供者服务的引用是通过DubboProtocol.refer方法。

## DubboProtocol.refer 构建服务引用

1. protocol.refer会先经过ProtocolListenerWrapper和ProtocolFilterWrapper构建监听器链和过滤器链
2. 根据url获取ExchangeClient对象，构建消费者和提供者的底层通信链接
3. 创建DubboInvoker，它包含对远程提供者的长链接，从而可以真正执行远程调用，并返回给目录服务RegistryDirectory持有

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // create rpc invoker.
    // getClients方法返回ExchangeClient对象
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

## 服务引用系列图及活动图

![img](https://oscimg.oschina.net/oscnet/dcb514f7d2071d0eb9a69dc80739e18d782.jpg)

![img](https://oscimg.oschina.net/oscnet/480c46843812d30026b30264a1463360a58.jpg)

参考：[https://blog.csdn.net/quhongwei_zhanqiu/article/details/41651487](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fblog.csdn.net%2Fquhongwei_zhanqiu%2Farticle%2Fdetails%2F41651487)