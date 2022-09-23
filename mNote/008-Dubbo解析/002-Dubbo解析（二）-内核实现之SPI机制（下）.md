# Dubbo解析（二）-内核实现之SPI机制（下）



上一章我们介绍了JDK的SPI机制，它旨在建立一种服务发现的规范。而Dubbo基于此根据框架的整体设计做了一些改进：

- JDK的SPI机制会一次性实例化所有服务提供者实现，如果有提供者的初始化很耗时，但并不会使用会很耗费资源。**Dubbo则只存储了所有提供者的Class对象，实际使用时才构造对象**。
- JDK的SPI机制只在配置文件中记录了实现类的全限定名，并没有定义一个配置名。而Dubbo的服务接口往往提供多个实现方式，需要在每个服务接口中定义一个匹配方法去选择要使用哪种实现方式。这种方式不利于框架的扩展，因而规定在提供者实现类的配置文件中**对每个实现类定义一个配置名**，用"="隔开，形成key-value的方式。
- 增加了对服务接口的IOC和AOP的支持，服务接口内的其他服务接口成员直接通过SPI的方式加载注入。

Dubbo作为一个微内核+插件的框架设计，其内核就是基于SPI机制动态地组装插件。插件都是以接口的方式定义在框架中，每个接口提供的服务也称为**扩展点**，表示每个服务接口都可以根据不同的条件进行动态扩展，**Dubbo的扩展点接口以@SPI注解标识**。Dubbo自身的功能也是通过扩展点实现的，也就是Dubbo的所有功能点都可以被用户自定义扩展所代替。那么Dubbo的SPI机制究竟是怎么工作的呢？

## 1.SPI约定

在classpath下放置META-INF/dubbo/以接口全限定名定义的文件，内容为配置名=扩展实现类全限定名，多个实现类用换行符分隔。

META-INF/dubbo目录是针对二次开发者的，dubbo自身加载扩展点配置的目录有三个，依次是：

1. META-INF/dubbo/internal/
2. META-INF/dubbo/
3. META-INF/services/

以Dubbo的协议接口Protocol为例，在dubbo-rpc-default模块中定义了DubboProtocol实现，目录为

```xml
META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol
```

文件中内容为

```xml
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
```

对应于dubbo的xml配置，即

```xml
<dubbo:protocol name="dubbo" />
```

## 2.扩展点适配器

JDK自带的SPI机制使用ServiceLoader加载和获取，**Dubbo对于扩展点的的加载和获取则是使用ExtensionLoader**。大部分的扩展点都是通过ExtensionLoader预加载一个适配类，以Protocol为例，调用方式如下：

```java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

getAdaptiveExtension方法创建扩展点的适配类，创建的方式分为两种

1. 如果有且仅有一个实现类上有@Adaptive注解，则直接由其Class对象的newInstance方法创建适配类对象
2. 如果不存在类上有@Adaptive注解的实现类，则使用字符串拼接类名为**接口名$Adaptive**的Class代码，默认**使用javassist编译生成Class对象**，然后使用newInstance方法创建适配类对象。

对于1的方式很好理解，比如Compiler接口，它的实现类AdaptiveCompiler类上就有@Adaptive注解

```java
@Adaptive
public class AdaptiveCompiler implements Compiler {

    // 实现代码省略

}
```

2的方式就较为复杂，还是以Protocol举例，先看下Protocol接口的源码：

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

Protocol接口定义了四个方法，其中export和refer方法都有@Adaptive注解，不同于1中类上的注解，这里的注解是在方法上，这个地方要着重强调一下@Adaptive注解作用的位置。对于实现类上没有此注解，而方法上有此注解的，适用于2方式。Protocol基于2方式生成的代码如下：

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;


public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    // 没有@Adaptive注解的直接抛出UnsupportedOperationException
    public void destroy() {
        throw new UnsupportedOperationException(
            "method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException(
            "method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    // 有@Adaptive的方法，从参数里获取URL对象，
    // 获取扩展点配置名，ExtensionLoader.getExtension(extName)匹配创建扩展点实现类对象
    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,
        com.alibaba.dubbo.common.URL arg1)
        throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) {
            throw new IllegalArgumentException("url == null");
        }

        com.alibaba.dubbo.common.URL url = arg1;
        String extName = ((url.getProtocol() == null) ? "dubbo"
                                                      : url.getProtocol());

        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" +
                url.toString() + ") use keys([protocol])");
        }

        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                                                                                                   .getExtension(extName);

        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(
        com.alibaba.dubbo.rpc.Invoker arg0)
        throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) {
            throw new IllegalArgumentException(
                "com.alibaba.dubbo.rpc.Invoker argument == null");
        }

        if (arg0.getUrl() == null) {
            throw new IllegalArgumentException(
                "com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        }

        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = ((url.getProtocol() == null) ? "dubbo"
                                                      : url.getProtocol());

        if (extName == null) {
            throw new IllegalStateException(
                "Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" +
                url.toString() + ") use keys([protocol])");
        }

        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
                                                                                                   .getExtension(extName);

        return extension.export(arg0);
    }
}
```

生成的适配类对象，以扩展点接口+Adaptive命名(如ProtocolAdaptive命名(如ProtocolAdaptive)，实现了扩展点接口。没有@Adaptive的方法直接抛出UnsupportedOperationException，对于**[有@Adaptive的方法，从参数中获取URL对象，然后根据URL获取扩展点配置名，再使用ExtensionLoader.getExtension](https://www.oschina.net/action/GoToLink?url=mailto%3A有%40Adaptive的方法，从参数中获取URL对象，然后根据URL获取扩展点配置名，再使用ExtensionLoader.getExtension)(extName)匹配创建扩展点实现类对象。**

这里的URL是Dubbo自定义的类，它是Dubbo的统一数据模型，穿插于系统的整个执行过程。URL的数据格式如下

```xml
dubbo://192.168.2.100:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-consumer&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&owner=william&pid=7356&side=consumer&timestamp=1416971340626
```

a. URL获取扩展点配置名，通过key从URL的parameter中获取

```
String extName = url.getParameter(key, defalutValue);
```

defalutValue取值于扩展点接口的@SPI的值，比如Protocol接口的defaultValue=dubbo

```java
@SPI("dubbo")
public interface Protocol {}
```

key的值来自于@Adaptive的value值，如果@Adaptive没有设置value，默认为扩展点接口的类的simpleName。如果key的值为protocol时特殊处理：

```java
String extName = url.getProtocol()
```

比如Transporter扩展点的bind方法

```java
@SPI("netty")
public interface Transporter {

    @Adaptive({"server", "transporter"})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;
}
```

defaultValue=netty，key=server,transporter，获取配置名的方法：

```java
String extName = url.getParameter("server",url.getParameter("transporter", "netty"));
```

b. 扩展点创建完$Adaptive对象，具体调用时，根据URL参数获取的配置名extName，查询对应的扩展点实现，还是以Protocol扩展点举例

```java
ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
```

getExtension(extName)方法匹配扩展点的实现类Class的配置名，找到对应的Class对象，执行newInstance方法创建实际的操作对象。extName=dubbo的Protocol扩展点实现类为DubboProtocol。创建完实现类对象，会对此对象执行**injectExtension**方法，即对对象内的以set开头，并且只有一个参数的public方法，执行IOC注入。

IOC注入也是以ExtensionFactory扩展点的方式实现，默认的方式是先以SPI机制获取方法参数类型的实现，如果此方法参数类型非接口或没有@SPI注解，则从Spring上下文中获取。

```java
if (objectFactory != null) {
    for (Method method : instance.getClass().getMethods()) {
        if (method.getName().startsWith("set")
                && method.getParameterTypes().length == 1
                && Modifier.isPublic(method.getModifiers())) {
            Class<?> pt = method.getParameterTypes()[0];
            try {
                String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                Object object = objectFactory.getExtension(pt, property);
                if (object != null) {
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("fail to inject via method " + method.getName()
                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
            }
        }
    }
}
```

接下来判断扩展点的实现类中是否存在**包装类(Wrapper)**，Wrapper类指存在参数为扩展点接口的构造方法的扩展点实现类。Wrapper类不是扩展点的真正实现，主要用于对真正的扩展点实现进行包装。比如Protocol的Filter包装类实现ProtocolFilterWrapper。

如果扩展点存在包装类的Class，反射调用其构造方法，将真正的扩展点实现传入，并再此执行injectExtension方法进行IOC注入。

```java
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
```

至此，扩展点的调用来到了真正对应的扩展点实现类对象中。

## 3.扩展点自动激活

如果需要同时加载多个扩展点实现，可以使用自动激活。当扩展点的实现类上有@Activate注解，标识它是一个激活类。@Activate可以配置group和value值，ExtensionLoader的getActivateExtension方法匹配并获取符合条件的所有激活类。Dubbo内部使用激活类的主要在Protocol的Filter和Listener，后面再详细介绍。

## 4.配置文件的加载

以上讨论的扩展点的适配类，包装类，激活类的实现，都要基于对SPI机制的配置文件的加载。Dubbo在获取扩展点的任何扩展时，都会先判断是否加载了配置文件，如果没有，即执行loadExtensionClasses方法加载，loadExtensionClasses方法中对Dubbo的三个配置目录分别加载，调用loadFile方法。

loadFile方法查找目录下扩展点接口的全限定名的文件，过滤掉"#"的注释内容，然后每行以"="分隔，获取配置名和扩展点实现类的全限定名。

依次判断是否为Adaptive适配类，是否为Wrapper包装类，是否为Activate激活类，并存储到对应的变量或集合中去。详细操作可见ExtensionLoader.loadFile方法。

最后贴上getExtension(name)方法的整体活动图（右击图片打开新标签页查看大图），来自大神的博客[https://blog.csdn.net/quhongwei_zhanqiu/article/details/41577235](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fblog.csdn.net%2Fquhongwei_zhanqiu%2Farticle%2Fdetails%2F41577235)