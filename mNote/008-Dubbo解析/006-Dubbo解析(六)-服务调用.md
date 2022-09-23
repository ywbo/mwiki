# Dubbo解析(六)-服务调用

当dubbo消费方和提供方都发布和引用完成后，第四步就是消费方调用提供方。

![image](https://oscimg.oschina.net/oscnet/cbdd657d246ff48aae96872fd73be44e541.jpg)

还是以dubbo的DemoService举例

```xml
-- 提供方
<dubbo:application name="demo-provider"/>
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<dubbo:protocol name="dubbo" port="20880"/>
<bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

-- 消费方
<dubbo:application name="demo-consumer"/>
<dubbo:registry address="zookeeper://127.0.0.1:2181"/>
<dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>
```

整个服务的调用过程都是**以Invoker接口为核心**，通过不同的Invoker实现对象层层调用，完成整个RPC的调用。**在调用过程中，消费方构建包含调用信息的RpcInvocation，并封装在Request中，传输到调用方，调用方解析出Request中的RpcInvocation完成调用，并将调用结果RpcResult封装到Response中返回，然后由消费方接收后再从Response中解析出RpcResult中携带的调用结果完成整个调用。**

## 服务消费方发起请求

消费方获取的demoService实例对象实际是代理工厂ProxyFactory.getProxy创建的代理对象

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

因而调用demoService.sayHello方法时，实际调用的是javassist生成的代理对象。消费方的调用堆栈如下

![image](https://oscimg.oschina.net/oscnet/f96f741614706c7d68319a71889da8ce8b4.jpg)

### 1. 代理对象执行sayHello方法

代理对象是由javassist生成，生成的sayHello方法如下

```java
public String sayHello(String paramString){
    Object[] arrayOfObject = new Object[1];
    arrayOfObject[0] = paramString;
    Object localObject = this.handler.invoke(this, methods[0], arrayOfObject);
    return ((String)localObject);
}
```

### 2. 将方法名和方法参数传入InvokerInvocationHandler.invoke执行

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    if ("toString".equals(methodName) && parameterTypes.length == 0) {
        return invoker.toString();
    }
    if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
        return invoker.hashCode();
    }
    if ("equals".equals(methodName) && parameterTypes.length == 1) {
        return invoker.equals(args[0]);
    }
    return invoker.invoke(new RpcInvocation(method, args)).recreate();
}
```

判断非Object的方法，将方法和参数组装成RpcInvocation，作为Invoker.invoke的参数。

### 3. 执行MockClusterInvoker.invoke方法，MockClusterInvoker是Cluster服务接口的Wrapper包装类MockClusterWrapper调用join方法返回的Invoker对象。

```java
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new MockClusterInvoker<T>(directory,
            this.cluster.join(directory));
}
```

根据url获取方法参数上的MOCK_KEY的值，判断是否要执行mock调用：

a. 如果mock为null或false，直接调用FailoverClusterInvoker b. 如果mock以force开头，强制执行mock调用 c. 以上都不是，则先调用FailoverClusterInvoker，调用失败再执行mock调用

### 4. 执行FailoverClusterInvoker.invoke，Failover是Cluster集群的默认策略。invoke方法由AbstractClusterInvoker执行，然后调用FailoverClusterInvoker的doInvoke实现。

```java
// 执行Directory.list 和 router.route
List<Invoker<T>> invokers = list(invocation);
// 执行LoadBalance.select
Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
```

a. 通过Directory目录服务的list方法获取订阅返回的服务提供者的Invoker对象集合，如果存在路由，路由服务根据策略过滤Invoker对象集合 b. 根据负载均衡策略LoadBalance来选择一个Invoker

### 5. 执行Filter过滤器链和Listener监听器链的invoke方法

Filter的Invoker执行器链由Protocol的Wrapper类ProtocolFilterWrapper.refer方法再调用buildInvokerChain构建。

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (!filters.isEmpty()) {
    	// 倒序循环，顺序执行
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                public URL getUrl() {
                    return invoker.getUrl();
                }

                public boolean isAvailable() {
                    return invoker.isAvailable();
                }

                public Result invoke(Invocation invocation) throws RpcException {
                    return filter.invoke(next, invocation);
                }

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
```

而Listener的Invoker执行器则是Protocol的Wrapper类ProtocolListenerWrapper.refer方法返回的ListenerInvokerWrapper。

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        return protocol.refer(type, url);
    }
    return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
            Collections.unmodifiableList(
                    ExtensionLoader.getExtensionLoader(InvokerListener.class)
                            .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
}
```

### 6. 最终由DubboInvoker执行对远程服务的调用

服务消费方调用远程服务时，传递的参数是Request对象，封装了RpcInvocation。

```java
public class RpcInvocation implements Invocation, Serializable {

    private static final long serialVersionUID = -4355285085441097045L;

    // 方法名称
    private String methodName;

    // 参数类型
    private Class<?>[] parameterTypes;

    // 参数值
    private Object[] arguments;

    // 附件
    private Map<String, String> attachments;

    private transient Invoker<?> invoker;
}
```

RpcInvocation包含了服务调用执行的方法名和参数，以及在附件中封装了服务的path、interface、group和version。

在DubboInvoker.invoke中，先获取交互层客户端ExchangeClient，其中包含了和服务提供者的长连接，传入RpcInvocation参数，由通信层HeaderExchangeChannel将RpcInvocation封装成Request，然后传输出去。

```java
HeaderExchangeChannel

public void send(Object message, boolean sent) throws RemotingException {
    if (message instanceof Request
            || message instanceof Response
            || message instanceof String) {
        channel.send(message, sent);
    } else {
        Request request = new Request();
        request.setVersion("2.0.0");
        request.setTwoWay(false);
        request.setData(message);
        channel.send(request, sent);
    }
}
```

对于DemoService.sayHello方法，消费方传输的RpcInvocation的内容如下

```java
RpcInvocation [methodName=sayHello, parameterTypes=[class java.lang.String], arguments=[world], attachments={path=com.alibaba.dubbo.demo.DemoService, interface=com.alibaba.dubbo.demo.DemoService, version=0.0.0}]
```

然后由netty将数据传输到服务提供者，远程调用的类型分为同步，异步或oneway模式，对应的调用结果是：

1.oneway返回空RpcResult，不接收返回结果

2.异步方式直接返回空RpcResult，而异步获取ResponseFuture回调

3.同步方式，调用方式还是异步，通过等待ResponseFuture.get()返回结果

整个服务消费方的活动图如下

![image](https://oscimg.oschina.net/oscnet/9a944032a7082631a3e63386d762fe1276d.jpg)

## 服务提供方接收调用请求

服务提供方在export服务后，就打开端口等待服务消费方的调用。当服务消费方调用发送调用时，服务提供方netty接收到MessageEvent，调用NettyHandler的messageReceived方法，然后向上一直调用到DubboProtocol的requestHandler的reply方法，获取到对应的Invoker，执行invoke方法调用服务的实际实现。

服务提供者的调用堆栈，调用从下到上，分为两个部分，由两个线程来执行。

![img](https://oscimg.oschina.net/oscnet/37706f9edbf3071e4dc5d22ca1442165358.jpg) ![img](https://oscimg.oschina.net/oscnet/606856355435e8c606e86d15ea3d9b61981.jpg)

### 1. 从netty接收请求交予NettyHandler处理，到DubboProtocol的requestHandler，中间经过很多ChannelHandler，对请求的消息进行不同的处理。

在AllChannelHandler中，会把请求放入事件处理线程池中，由ChannelEventRunnable线程执行。这是一个分发机制(Dispatch服务)，用于指定事件执行的线程池的模型。

```java
AllChannelHandler

cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
```

在HeaderExchangeHandler中，handleRequest方法从Request中获取RpcInvocation，然后调用DubboProtocol的requestHandler。

```java
Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    // find handler by message class.
    Object msg = req.getData();
    try {
        // handle data.
        Object result = handler.reply(channel, msg);
        res.setStatus(Response.OK);
        res.setResult(result);
    } catch (Throwable e) {
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
    }
    return res;
}
```

### 2. DubboProtocol的requestHandler是一个ExchangeHandlerAdapter的内部类，received方法又调用reply方法，判断message是Invocation类型，根据Invocation获取服务调用Invoker。

方法参数Invocation，就是服务消费方发送的消息

```java
RpcInvocation [methodName=sayHello, parameterTypes=[class java.lang.String], arguments=[world], attachments={path=com.alibaba.dubbo.demo.DemoService, input=201, dubbo=2.0.0, interface=com.alibaba.dubbo.demo.DemoService, version=0.0.0}]
```

从中获取path、group和version，组装成serviceKey，从exporterMap中获取DubboExporter，从而得到Invoker。

```java
public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
    if (message instanceof Invocation) {
        Invocation inv = (Invocation) message;
        // 根据invocation匹配Invoker
        Invoker<?> invoker = getInvoker(channel, inv);
        // 部分省略
        RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
        return invoker.invoke(inv);
    }
}

Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
    int port = channel.getLocalAddress().getPort();
    String path = inv.getAttachments().get(Constants.PATH_KEY);
    // 组装serviceKey
    String serviceKey = serviceKey(port, path, inv.getAttachments().get(Constants.VERSION_KEY), inv.getAttachments().get(Constants.GROUP_KEY));

    // 根据serviceKey从exporterMap中获取DubboExporter，再获得Invoker
    DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

    return exporter.getInvoker();
}
```

### 3. 经过提供方Filter Invoker链，执行前置拦截，由Protocol的包装类ProtocolFilterWrapper.export再调用buildInvokerChain构建。

Filter分为消费方Filter和提供方Filter，是通过Filter实现类上@Activate的group设置。如消费方的ConsumerContextFilter的group=consumer，而提供方的ContextFilter的group=provider

```java
@Activate(group = Constants.CONSUMER, order = -10000)
public class ConsumerContextFilter implements Filter {}

@Activate(group = Constants.PROVIDER, order = -10000)
public class ContextFilter implements Filter {}
```

### 4.调用JavassistProxyFactory.getInvoker生成的AbstractProxyInvoker.invoke

JavassistProxyFactory.getInvoker创建实际对象的Wrapper类，并返回AbstractProxyInvoker内部类对象。

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
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

AbstractProxyInvoker.invoke执行时，调用上面内部类的doInvoke方法，并将执行结果封装成RpcResult。

```java
public Result invoke(Invocation invocation) throws RpcException {
    try {
        return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
    } catch (InvocationTargetException e) {
        return new RpcResult(e.getTargetException());
    } catch (Throwable e) {
        throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

### 5.通过Wrapper包装类，执行真正的demoService的方法

```java
public class Wrapper0 extends Wrapper {
    public Object invokeMethod(Object o, String n, Class[] p, Object[] v)
    throws java.lang.reflect.InvocationTargetException {
        com.alibaba.dubbo.demo.provider.DemoServiceImpl w;
        
        try {
            w = ((com.alibaba.dubbo.demo.provider.DemoServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        
        try {
            if ("sayHello".equals($2) && ($3.length == 1)) {
                return ($w) w.sayHello((java.lang.String) $4[0]);
            }
        } catch (Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
        }
        
        throw new com.alibaba.dubbo.common.bytecode.NoSuchMethodException(
            "Not found method \"" + $2 +
            "\" in class com.alibaba.dubbo.demo.provider.DemoServiceImpl.");
        }
    }
}
```

服务提供方执行的活动图

![image](https://oscimg.oschina.net/oscnet/13a17c4aa32ef18089b534324dcda034b24.jpg)

## 服务调用结果返回

当提供方执行完真正服务实现的方法后，需要将返回值传输给消费方

1. 提供方在AbstractProxyInvoker中组装返回结果成RpcResult
2. 提供方部分Filter链还有后置拦截操作，如处理异常的ExceptionFilter
3. 在HeaderExchangeHandler.handleRequest方法中，将RpcResult封装到Response返回
4. 经过网络传输，回到消费方，DefaultFuture.returnFromResponse方法从Response中解析出RpcResult
5. 经过层层返回，来到InvokerInvocationHandler，调用RpcResult.recreate方法返回调用结果

## Request和Response的组装和解析

1. 消费方执行HeaderExchangeChannel.request时将RpcInvocation组装成Request

```java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    // create request.
    Request req = new Request();
    req.setVersion("2.0.0");
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try {
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

1. 提供方执行HeaderExchangeHandler.handleRequest时，解析出Request的RpcInvocation，并将返回结果RpcResult封装到Response中

```java
Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    // find handler by message class.
    Object msg = req.getData();
    try {
        // handle data.
        Object result = handler.reply(channel, msg);
        res.setStatus(Response.OK);
        res.setResult(result);
    } catch (Throwable e) {
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
    }
    return res;
}
```

1. 消费方等待提供方返回然后在DefaultFuture.get方法中执行returnFromResponse方法，从Response中获取RpcResult

```java
private Object returnFromResponse() throws RemotingException {
    Response res = response;
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) {
        return res.getResult();
    }
    if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        throw new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage());
    }
    throw new RemotingException(channel, res.getErrorMessage());
}
```

整个Dubbo服务调用的活动图

![image](https://oscimg.oschina.net/oscnet/b699728baecd6acb33b5313351bc943ede5.jpg)

参考：[https://blog.csdn.net/quhongwei_zhanqiu/article/details/41701979](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fblog.csdn.net%2Fquhongwei_zhanqiu%2Farticle%2Fdetails%2F41701979)