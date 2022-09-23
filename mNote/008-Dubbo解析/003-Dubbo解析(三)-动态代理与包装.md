# Dubbo解析(三)-动态代理与包装

Dubbo作为RPC框架，首先要完成的就是跨系统，跨网络的服务调用。消费方与提供方遵循统一的接口定义，消费方调用接口时，Dubbo将其转换成统一格式的数据结构，通过网络传输，提供方根据规则找到接口实现，通过反射完成调用。也就是说，消费方获取的是对远程服务的一个代理(Proxy)，而提供方因为要支持不同的接口实现，需要一个包装层(Wrapper)。调用的过程大概是这样：

![img](https://oscimg.oschina.net/oscnet/bef19cd5a31b5ae13aff35a8cb4898faaf0.jpg)

消费方的Proxy和提供方的Wrapper得以让Dubbo构建出复杂、统一的体系。而这种动态代理与包装也是通过基于SPI的插件方式实现的，它的接口就是**ProxyFactory**。

## 1. ProxyFactory接口定义

```java
@SPI("javassist")
public interface ProxyFactory {

    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```

ProxyFactory有两个方法，分别是消费方生成代理(getProxy方法)和提供方创建Invoker对象(getInvoker方法)。

它的类结构如下：

![image](https://img-blog.csdn.net/20141129170712640)

ProxyFactory有两种实现方式，一种是基于JDK的代理实现，一种是基于javassist的实现。ProxyFactory接口上定义了@SPI("javassist")，默认为javassist的实现。为什么选择了javassist，可以看下[动态代理方案性能对比](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fjavatar.iteye.com%2Fblog%2F814426%2F)。

## 2. getProxy方法

消费方基于调用接口获取代理对象，使用动态代理，动态代理有多种方式，Dubbo默认支持了JDK和javassist两种。

AbstractProxyFactory是代理工厂的公共抽象，主要用来在getProxy方法中获取代理接口，先从Invoker的url中获取，如果没有，则直接使用Invoker的接口。

```java
AbstractProxyFactory.java

public <T> T getProxy(Invoker<T> invoker) throws RpcException {
    Class<?>[] interfaces = null;
    String config = invoker.getUrl().getParameter("interfaces");
    if (config != null && config.length() > 0) {
        String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
        if (types != null && types.length > 0) {
            interfaces = new Class<?>[types.length + 2];
            interfaces[0] = invoker.getInterface();
            interfaces[1] = EchoService.class;
            for (int i = 0; i < types.length; i++) {
                interfaces[i + 1] = ReflectUtils.forName(types[i]);
            }
        }
    }
    if (interfaces == null) {
        interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
    }
    return getProxy(invoker, interfaces);
}
```

JdkProxyFactory中getProxy直接使用JDK的Proxy.newProxyInstance方法生成代理对象。这里的Proxy是java.lang.reflect.Proxy。

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
}
```

而JavassistProxyFactory的getProxy实现虽然也只有一行，但内部实现都是由Dubbo基于javassist实现的。而这里的Proxy是com.alibaba.dubbo.common.bytecode.Proxy，是生成代理对象的工具类。

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

Dubbo的Proxy.getProxy源码：

```java
public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
	if (ics.length > 65535)
		throw new IllegalArgumentException("interface limit exceeded");

	// 遍历代理接口，获取接口的全限定名，并以分号分隔连接成字符串
	StringBuilder sb = new StringBuilder();
	for (int i = 0; i < ics.length; i++) {
		String itf = ics[i].getName();
		if (!ics[i].isInterface())
			throw new RuntimeException(itf + " is not a interface.");

		Class<?> tmp = null;
		try {
			tmp = Class.forName(itf, false, cl);
		} catch (ClassNotFoundException e) {
		}

		if (tmp != ics[i])
			throw new IllegalArgumentException(ics[i] + " is not visible from class loader");

		sb.append(itf).append(';');
	}

	// use interface class name list as key.
	String key = sb.toString();

	// get cache by class loader.
	Map<String, Object> cache;
	synchronized (ProxyCacheMap) {
		cache = ProxyCacheMap.get(cl);
		if (cache == null) {
			cache = new HashMap<String, Object>();
			ProxyCacheMap.put(cl, cache);
		}
	}

	// 查找缓存map，如果缓存存在，则获取代理对象直接返回
	Proxy proxy = null;
	synchronized (cache) {
		do {
			Object value = cache.get(key);
			if (value instanceof Reference<?>) {
				proxy = (Proxy) ((Reference<?>) value).get();
				if (proxy != null)
					return proxy;
			}

			if (value == PendingGenerationMarker) {
				try {
					cache.wait();
				} catch (InterruptedException e) {
				}
			} else {
				cache.put(key, PendingGenerationMarker);
				break;
			}
		}
		while (true);
	}

	// AtomicLong自增生成代理类类名后缀id，防止冲突
	long id = PROXY_CLASS_COUNTER.getAndIncrement();
	String pkg = null;
	ClassGenerator ccp = null, ccm = null;
	try {
		ccp = ClassGenerator.newInstance(cl);

		Set<String> worked = new HashSet<String>();
		List<Method> methods = new ArrayList<Method>();

		for (int i = 0; i < ics.length; i++) {
			if (!Modifier.isPublic(ics[i].getModifiers())) {
				String npkg = ics[i].getPackage().getName();
				if (pkg == null) {
					pkg = npkg;
				} else {
					if (!pkg.equals(npkg))
						throw new IllegalArgumentException("non-public interfaces from different packages");
				}
			}
			ccp.addInterface(ics[i]);

			// 遍历接口中的方法，获取返回类型和参数类型，构建方法体
			for (Method method : ics[i].getMethods()) {
				String desc = ReflectUtils.getDesc(method);
				if (worked.contains(desc))
					continue;
				worked.add(desc);

				int ix = methods.size();
				Class<?> rt = method.getReturnType();
				Class<?>[] pts = method.getParameterTypes();

				StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
				for (int j = 0; j < pts.length; j++)
					code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
				code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");
				if (!Void.TYPE.equals(rt))
					code.append(" return ").append(asArgument(rt, "ret")).append(";");

				methods.add(method);
				ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
			}
		}

		if (pkg == null)
			pkg = PACKAGE_NAME;

		// create ProxyInstance class.
		// 生成服务接口的代理对象字节码
		String pcn = pkg + ".proxy" + id;
		ccp.setClassName(pcn);
		ccp.addField("public static java.lang.reflect.Method[] methods;");
		ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
		ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
		ccp.addDefaultConstructor();
		Class<?> clazz = ccp.toClass();
		clazz.getField("methods").set(null, methods.toArray(new Method[0]));

		// create Proxy class.
		// 生成Proxy的实现类
		String fcn = Proxy.class.getName() + id;
		ccm = ClassGenerator.newInstance(cl);
		ccm.setClassName(fcn);
		ccm.addDefaultConstructor();
		ccm.setSuperClass(Proxy.class);
		ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
		Class<?> pc = ccm.toClass();
		proxy = (Proxy) pc.newInstance();
	} catch (RuntimeException e) {
		throw e;
	} catch (Exception e) {
		throw new RuntimeException(e.getMessage(), e);
	} finally {
		// release ClassGenerator
		if (ccp != null)
			ccp.release();
		if (ccm != null)
			ccm.release();
		synchronized (cache) {
			if (proxy == null)
				cache.remove(key);
			else
				// 加入缓存
				cache.put(key, new WeakReference<Proxy>(proxy));
			cache.notifyAll();
		}
	}
	return proxy;
}
```

getProxy生成代理对象步骤如下：

a. 遍历代理接口，获取接口的全限定名，并以分号分隔连接成字符串，以此字符串为key，查找缓存map，如果缓存存在，则获取代理对象直接返回。

b. 由一个AtomicLong自增生成代理类类名后缀id，防止冲突

c. 遍历接口中的方法，获取返回类型和参数类型，构建主要内容如下的方法体

```java
return  ret= handler.invoke(this, methods[ix], args);
```

以一个DemoService为例，它只有一个方法sayHello(String name)，生成的方法体如下：

```java
public String sayHello(String arg0) {
    Object[] args = new Object[1];
    args[0] = arg0;

    Object ret = handler.invoke(this, methods[0], args);

    return (String) ret;
}
```

另外使用worked的Set集合对方法去重

d. 创建工具类ClassGenerator实例，添加静态字段Method[] methods，添加实例对象InvokerInvocationHandler hanler，添加参数为InvokerInvocationHandler的构造器，添加无参构造器，然后使用toClass方法生成对应的字节码。还是DemoService为例，生成的字节码对象如下：

```java
import com.alibaba.dubbo.demo.DemoService;
import com.alibaba.dubbo.rpc.service.EchoService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class proxy0
  implements ClassGenerator.DC, EchoService, DemoService
{
  public static Method[] methods;
  private InvocationHandler handler;

  public String sayHello(String paramString)
  {
    Object[] arrayOfObject = new Object[1];
    arrayOfObject[0] = paramString;
    Object localObject = this.handler.invoke(this, methods[0], arrayOfObject);
    return ((String)localObject);
  }

  public Object $echo(Object paramObject)
  {
    Object[] arrayOfObject = new Object[1];
    arrayOfObject[0] = paramObject;
    Object localObject = this.handler.invoke(this, methods[1], arrayOfObject);
    return ((Object)localObject);
  }

  public proxy0()
  {
  }

  public proxy0(InvocationHandler paramInvocationHandler)
  {
    this.handler = paramInvocationHandler;
  }
}
```

e. d中生成的字节码对象为服务接口的代理对象，而Proxy类本身是抽象类，需要实现newInstance(InvocationHandler handler)方法，生成Proxy的实现类，其中proxy0即上面生成的服务接口的代理对象。

```java
package com.alibaba.dubbo.common.bytecode;

public class Proxy0 implements Proxy {

    public void Proxy0() {
    }

    public Object newInstance(InvocationHandler h){
		return new proxy0(h);
	}
}
```

再回头看看JavassistProxyFactory的getPrxoy方法， InvokerInvocationHandler封装了Invoker对象，通过Proxy的newInstance方法，最终返回通过javassist生成的代理对象。

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

## 3. getInvoker方法

Invoker，Dubbo的核心模型，代表一个可执行体。在我理解，它就是**一个指向最终服务实现的执行路径**，在Dubbo的实现中，提供方的服务实现首先被包装成一个ProxyInvoker(代理执行器)，然后这个ProxyInvoker被Filter封装成链式结构，被Listener包装，被Cluster封装，而消费方的Invoker也是一个执行远程调用的执行器，区别在于执行路径的不同。

ProxyFactory的getInvoker方法，所做的就是第一步，将服务实现封装成代理执行器，有JdkProxyFactory和JavassistProxyFactory两种实现，两种都是创建匿名内部类AbstractProxyInvoker，实现doInvoke方法。

JDK的实现比较简单，匹配到调用的方法，使用反射执行。

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            Method method = proxy.getClass().getMethod(methodName, parameterTypes);
            return method.invoke(proxy, arguments);
        }
    };
}
```

而JavassistProxyFactory则是通过javassist生成字节码对象Wrapper类进行调用。

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

Wrapper类本身是抽象类，是对java类的一种包装，将类中的field和method抽象出propertyName和methodName，真正调用时根据传入的方法名和参数进行匹配。

```java
public abstract class Wrapper {

    abstract public String[] getPropertyNames();

    abstract public Class<?> getPropertyType(String pn);

    abstract public boolean hasProperty(String name);

    abstract public Object getPropertyValue(Object instance, String pn) throws NoSuchPropertyException, IllegalArgumentException;

    abstract public void setPropertyValue(Object instance, String pn, Object pv) throws NoSuchPropertyException, IllegalArgumentException;

    abstract public String[] getMethodNames();

    abstract public String[] getDeclaredMethodNames();

    abstract public Object invokeMethod(Object instance, String mn, Class<?>[] types, Object[] args) throws NoSuchMethodException, InvocationTargetException;
}
```

Wrapper.getWrapper方法基于不同的服务实现对象生成一个Wrapper实现对象。getWrapper方法先查询缓存是否已存在，存在则返回，否则调用makeWrapper生成Wrapper对象。

```java
public static Wrapper getWrapper(Class<?> c) {
    while (ClassGenerator.isDynamicClass(c)) // can not wrapper on dynamic class.
        c = c.getSuperclass();

    if (c == Object.class)
        return OBJECT_WRAPPER;

    Wrapper ret = WRAPPER_MAP.get(c);
    if (ret == null) {
        ret = makeWrapper(c);
        WRAPPER_MAP.put(c, ret);
    }
    return ret;
}
```

makeWrapper方遍历传入的Class对象的所有public field和public method，构建字节码对象字符串。

1. public字段组建getPropertyValue和setPropertyValue方法，比如有public字段name

```java
// getPropertyValue：
// $2代表传入的字段名称，w为原始对象，$w为name的类型
if ($2.equals("name")) { return ($w) w.name;} 

// setPropertyValue
// $2代表传入的字段名称，w为原始对象，$3为传入的字段值
if ($2.equals("name")) { w.name = ($w) $3; return;} 
```

1. 处理public方法，区分setter/getter方法，转化成属性加入到getPropertyValue和setPropertyValue方法中，另外所有方法加入到invokeMethod方法中。

```java
// 以sayHello为例，转化到invokeMethod方法中
// $2为传入方法名称，$3为传入方法参数类型数组，$4为传入方法参数值数组
if ("sayHello".equals($2) && ($3.length == 1)) {
    return ($w) w.sayHello((java.lang.String) $4[0]);
}
```

以DemoServiceImpl为例

```java
public class DemoServiceImpl implements DemoService {
	private String name;
	
	public String sayHello(String name) {
        return "Hello";
    }

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}
}
```

生成的Wrapper类如下：

```java
public class Wrapper0 extends Wrapper {
    public static String[] pns = new String[] { "name" };
    public static Map pts = new HashMap() {

            {
                put("name", "java.lang.String");
            }
        };

    public static String[] mns = new String[] { "sayHello" };
    public static String[] dmns = new String[] { "sayHello" };
    public static Class[] mts0 = new Class[] { String.class };

    public String[] getPropertyNames() {
        return pns;
    }

    public boolean hasProperty(String n) {
        return pts.containsKey(n);
    }

    public Class getPropertyType(String n) {
        return (Class) pts.get(n);
    }

    public String[] getMethodNames() {
        return mns;
    }

    public String[] getDeclaredMethodNames() {
        return dmns;
    }

    public void setPropertyValue(Object o, String n, Object v) {
        com.alibaba.dubbo.demo.provider.DemoServiceImpl w;

        try {
            w = ((com.alibaba.dubbo.demo.provider.DemoServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }

        if ($2.equals("name")) {
            w.setName((java.lang.String) $3);

            return;
        }

        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException(
            "Not found property \"" + $2 +
            "\" filed or setter method in class com.alibaba.dubbo.demo.provider.DemoServiceImpl.");
    }

    public Object getPropertyValue(Object o, String n) {
        com.alibaba.dubbo.demo.provider.DemoServiceImpl w;

        try {
            w = ((com.alibaba.dubbo.demo.provider.DemoServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }

        if ($2.equals("name")) {
            return ($w) w.getName();
        }

        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException(
            "Not found property \"" + $2 +
            "\" filed or setter method in class com.alibaba.dubbo.demo.provider.DemoServiceImpl.");
    }

    public Object invokeMethod(Object o, String n, Class[] p, Object[] v)
        throws java.lang.reflect.InvocationTargetException {
        com.alibaba.dubbo.demo.provider.DemoServiceImpl w;

        try {
            w = ((com.alibaba.dubbo.demo.provider.DemoServiceImpl) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }

        try {
            if ("getName".equals($2) && ($3.length == 0)) {
                return ($w) w.getName();
            }

            if ("setName".equals($2) && ($3.length == 1)) {
                w.setName((java.lang.String) $4[0]);

                return null;
            }

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
```

在JavassistProxyFactory的getInvoker方法创建的AbstractProxyInvoker中，doInvoke方法最终通过Wrapper的invokeMethod方法完成对服务实现原始对象的调用。

```java
wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments)
```

现在你可以回头看下文章开始前的那张调用图，Dubbo使用ProxyFactory对消费方的调用和提供者的执行进行了代理和包装，从而为Dubbo的内部逻辑实现奠定了基础。