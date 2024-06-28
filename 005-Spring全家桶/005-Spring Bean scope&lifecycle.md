# ☁ Spring Bean scope&lifecycle

# 一、Spring Bean scope（作用域）

在使用 XML 方式配置 IoC 容器时，`<Bean>` 标签的 `scope` 属性可以指定 Bean 的作用域，如下所示。若不指定 `scope` 属性，则默认 `scope="singleton"`，即单例作用域。

```xml
<bean id="xmlinstancesingleton" class="test.model.XMLInstance"
    scope="singleton">
    <property name="name" value="abc"/>
</bean>

<bean id="xmlinstanceprototype" class="test.model.XMLInstance"
    scope="prototype">
    <property name="name" value="abc"/>
</bean>
```

### 1. 5种作用域

`scope` 属性值有下面 5 个可选，即 Spring Bean 的作用域 有 5 种

1. `singleton`：唯一 Bean 实例，Spring 中的 Bean 默认都是单例的。
2. `prototype`：每次请求都会创建一个新的 Bean 实例。
3. `request`：每一次 HTTP 请求都会产生一个新的 Bean，该 Bean 仅在当前 HTTP request 内有效。
4. `session`：每一次 HTTP 请求都会产生一个新的 Bean，该 Bean 仅在当前 HTTP session 内有效。
5. `global-session`：全局 session 作用域，仅仅在基于 Portlet 的 web 应用中才有意义，Spring 5 已经没有了。Portlet 是能够生成语义代码（如 HTML）片段的小型 Java Web 插件。它们基于 portlet 容器，可以像 servlet 一样处理 HTTP 请求。但是，与 servlet 不同，每个 portlet 都有不同的会话。

在开发过程中，对有状态的 Bean 建议使用 `Prototype`，对无状态建议使用 `Singleton`。

### 2. 单例模式的实现

- 单例模式下，可以设置类的构造函数为私有，这样外界就不能调用该类的构造函数来创建多个对象。
- 单例模式下，可以设置 get 方法为静态，由类直接调用。
- 单例模式的类实现方法有「饿汉式」和「懒汉式」，如下代码所示。

```java
//饿汉式

//singleton1作为类变量直接得到初始化，优点是在多线程环境下能够保证同步，不可能被实例化两次
//但是如果singleton1在很长一段时间后才使用，意味着singleton1实例开辟的堆内存会驻留很长时间，不能延迟加载，占用内存
public  class Singleton1{
    private static Singleton1 singleton1 = new Singleton1();

    public static Singleton1 getSingleton1(){
        return singleton1;
    }
}

//懒汉式
public  class Singleton2{
    private static Singleton2 singleton1 = null;

    public static Singleton2 getSingleton1(){
        if(singleton1 ==null){
            singleton1 = new Singleton2();
        }
        return singleton1;
    }
}
```

「懒汉式」是在使用的时候才去创建，这样可以避免类在初始化时提前创建。但是如果在多线程的情况下，因为线程上下文切换导致两个线程都通过了 `if` 判断，这样就会 `new` 出两个实例，无法保证唯一性。可以采用如下方式，规避这个问题（参考 [Spring中Bean的单例及七种创建单例的方法](https://link.juejin.cn?target=https%3A%2F%2Fwww.modb.pro%2Fdb%2F84343)）

1. #### 懒汉式 + 同步

```java
//使用 synchronized 关键字进行加锁
//添加同步控制后，getSingleton1()方法是串行化的，获取时需要排队等候，效率较低
public class Singleton3 {

    private static Singleton3 singleton1 = null;

    public synchronized Singleton3 getSingleton1() {
        if (singleton1 == null) {
            singleton1 = new Singleton3();
        }
        return singleton1;
    }
}
```

2. #### 懒汉式 + 双重校验

```java
// 若有两个线程通过了第一个check，进入第二个check是串行化的，只能有一个线程进入，保证了唯一性
public class Singleton4 {

    private static Singleton4 singleton1 = null;

    public static Singleton4 getSingleton1() {
        if (singleton1 == null) {
            synchronized (Singleton4.class) {
                if (singleton1 == null) {
                    singleton1 = new Singleton4();
                }
            }
        }
        return singleton1;
    }
}
```

### 3. 如何注入Prototype Bean

如何注入 Prototype Bean，有两种方式

1. #### XML 配置中指定 scope

```xml
<bean id="xmlinstanceprototype" class="test.model.XMLInstance"
    scope="prototype">
    <property name="name" value="abc"/>
</bean>
```

2. #### 使用 `@Scope("prototype")` 注解

```java
@Component
@Scope("prototype")
```

**多例模式（Prototype）在进行注入时，不能使用 @Autowired，否则注入的还是单例模式。**

**实现多例模式（Prototype）的注入：**

**① 可以通过 ApplicationContext 的 getBean() 方法来获得 Bean；**

**② 通过 BeanFactory 的 getBean() 方法来获得 Bean。**

```java
@Component
@Scope("prototype") //prototype
@Data
class User {
    private String name;
    private int age;
}

//@Autowired 获得依旧是单例 Bean
@SpringBootApplication
class SpringbootAppApplicationTest1{
    @Autowired
    private User user; 

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanConfig/BeanConfig.xml");
		
        //虽然 Bean 中使用了 @Scope("prototype")，但使用@Autowire注入
        //此处获得的 Bean 仍然是单例的
    }
}


//方式1
//ApplicationContext # getBean
@SpringBootApplication
class SpringbootAppApplicationTest2{

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanConfig/BeanConfig.xml");
        //通过ApplicationContext的getBean方法获得Bean
        //user1 user2 user3 是三个不同的对象
        User user1 = (User)context.getBean("user");
        User user2 = (User)context.getBean("user");
        User user3 = (User)context.getBean("user");
    }
}


//方式2 
//BeanFactory # getBean
@SpringBootApplication
class SpringbootAppApplicationTest2{

    @Autowired
    private org.springframework.beans.factory.BeanFactory beanFactory;

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanConfig/BeanConfig.xml");
        //通过BeanFactory的getBean方法获得Bean
        //user1 user2 user3 是三个不同的对象
        User user1 = beanFactory.getBean(User.class);
        User user2 = beanFactory.getBean(User.class);
        User user3 = beanFactory.getBean(User.class);
    }
}
```

------

# 二、Spring Bean lifecycle（生命周期）

可能看到过的好多解释bean生命周期的文章，上来就直接开搞，先甩一张图（图片源自网络）：

![Spring的生命周期](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/15/170485f55560d36e~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

直接懵逼了，这啥玩意儿啊？怎么记啊？或者总算是记住了，但是转头就忘了。

下面我们结合Spring源码来进行分析，记忆，只需要记住四步，然后结合源码过程，理解其原理，自然而然就了如指掌。

我将从两方面来理解 Bean 的生命周期

1、 生命周期的概要流程：主要是对Bean生命周期进行概括，并且结合源码来理解记忆；

2、扩展点的作用：详尽的了解 Bean 生命周期过程中的扩展点及其作用。

## 1. 生命周期的概要流程

Bean 的生命周期，概括起来就是**4各阶段**：

① 实例化（Instantiation）

② 属性赋值（Populate）

③ 初始化（Initialization）

④ 销毁（Destruction）

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/2/15/1704860a4de235aa~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)

1. 实例化：第1步，实例化一个 Bean 对象；

2. 属性赋值：第2步，设置第1步中实例化Bean对象的属性和依赖；

3. 初始化：第3 ~ 7步，步骤较多，分开来看：

   a. 第3、4步为在初始化前执行；

   b. 第5、6步为初始化操作；

   c. 第7步在初始化后执行，该阶段结束，才能被用户所使用；

4. 销毁：第 8~10步，第8步不是真正意义上的销毁（还没使用呢），而是先在使用前注册了销毁的相关调用接口，为了后面第9、10步真正销毁 bean 时再执行相应的方法。

下面，我们来结合源码实际分析。会更加直观的看到这4个阶段的执行：

总体流程如下源码所示，源码位置：*<font color='#2196F3'>org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean</font>*

![1719558327048](assets/1719558327048.png)

简化上述代码后，我们提炼出主要流程，从doCreateBean() 方法中能看到依次执行了这 4 个阶段：

```java
// AbstractAutowireCapableBeanFactory.java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
		BeanWrapper instanceWrapper = null;
		// 1.实例化Bean
		if (instanceWrapper == null) {
			instanceWrapper = this.createBeanInstance(beanName, mbd, args);
		}
		Object exposedObject = bean;
		// 2.属性赋值
		this.populateBean(beanName, mbd, instanceWrapper);
        // 3.初始化
		exposedObject = this.initializeBean(beanName, exposedObject, mbd);
        // 4.销毁-注册回调接口
		this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
		return exposedObject;
	}
```

由于初始化包含了第 3~7步，较复杂，所以我们进到 initializeBean() 方法里具体看下其过程（注释的序号对应图中序号）：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    // 3. 检查 Aware 相关接口并设置相关依赖
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    // 4. BeanPostProcessor 前置处理
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // 5. 若实现 InitializingBean 接口，调用 afterPropertiesSet() 方法
    // 6. 若配置自定义的 init-method方法，则执行
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    // 7. BeanPostProceesor 后置处理
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

在 invokInitMethods() 方法中会检查 InitializingBean 接口和 init-method 方法，销毁的过程也与其类似：

```java
// DisposableBeanAdapter.java
public void destroy() {
    // 9. 若实现 DisposableBean 接口，则执行 destory()方法
    if (this.invokeDisposableBean) {
        try {
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((DisposableBean) this.bean).destroy();
                    return null;
                }, this.acc);
            }
            else {
                ((DisposableBean) this.bean).destroy();
            }
        }
    }
    
	// 10. 若配置自定义的 detory-method 方法，则执行
    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
    else if (this.destroyMethodName != null) {
        Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
        if (methodToInvoke != null) {
            invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
        }
    }
}
```

从 Spring 的源码我们可以直观的看到其执行过程，而我们记忆其过程便可以从这 4 个阶段出发，实例化、属性赋值、初始化、销毁。其中细节较多的便是初始化，涉及了 Aware、BeanPostProcessor、InitializingBean、init-method 的概念。这些都是 Spring 提供的扩展点，其具体作用将在下一节讲述。

------

## 2. 扩展点的作用

### 2.1 Aware 接口

若 Spring 检测到 bean 实现了 Aware 接口，则会为其注入相应的依赖。所以**通过让bean 实现 Aware 接口，则能在 bean 中获得相应的 Spring 容器资源**。

Spring 中提供的 Aware 接口有：

1. BeanNameAware：注入当前 bean 对应 beanName；
2. BeanClassLoaderAware：注入加载当前 bean 的 ClassLoader；
3. BeanFactoryAware：注入 当前BeanFactory容器 的引用。

其代码实现如下：

```java
// AbstractAutowireCapableBeanFactory.java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

以上是针对 BeanFactory 类型的容器，而对于 ApplicationContext 类型的容器，也提供了 Aware 接口，只不过这些 Aware 接口的注入实现，是通过 BeanPostProcessor 的方式注入的，但其作用仍是注入依赖。

1. EnvironmentAware：注入 Enviroment，一般用于获取配置属性；
2. EmbeddedValueResolverAware：注入 EmbeddedValueResolver（Spring EL解析器），一般用于参数解析；
3. ApplicationContextAware（ResourceLoader、ApplicationEventPublisherAware、MessageSourceAware）：注入 ApplicationContext 容器本身。

其代码实现如下：

```java
// ApplicationContextAwareProcessor.java
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware)bean).setEnvironment(this.applicationContext.getEnvironment());
    }

    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware)bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }

    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware)bean).setResourceLoader(this.applicationContext);
    }

    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware)bean).setApplicationEventPublisher(this.applicationContext);
    }

    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware)bean).setMessageSource(this.applicationContext);
    }

    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware)bean).setApplicationContext(this.applicationContext);
    }

}
```

### 2.2 BeanPostProcessor

BeanPostProcessor 是 Spring 为**修改 bean**提供的强大扩展点，其可作用于容器中所有 bean，其定义如下：

```java
public interface BeanPostProcessor {

	// 初始化前置处理
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	// 初始化后置处理
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

常用场景有：

1. 对于标记接口的实现类，进行自定义处理。例如3.1节中所说的ApplicationContextAwareProcessor，为其注入相应依赖；再举个例子，自定义对实现解密接口的类，将对其属性进行解密处理；
2. 为当前对象提供代理实现。例如 Spring AOP 功能，生成对象的代理类，然后返回。

```java
// AbstractAutoProxyCreator.java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        // 返回代理类
        return proxy;
    }

    return null;
}
```

### 2.3 InitializingBean 和 init-method

InitializingBean 和 init-method 是 Spring 为 **bean 初始化**提供的扩展点。

InitializingBean接口 的定义如下：

```java
复制代码public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

在 afterPropertiesSet() 方法写初始化逻辑。

指定 init-method 方法，指定初始化方法：

```java
复制代码<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="demo" class="com.chaycao.Demo" init-method="init()"/>
    
</beans>
```

DisposableBean 和 destory-method 与上述类似，就不描述了。

## 3. 总结

最后总结下如何记忆 Spring Bean 的生命周期：

- 首先是实例化、属性赋值、初始化、销毁这 4 个大阶段；
- 再是初始化的具体操作，有 Aware 接口的依赖注入、BeanPostProcessor 在初始化前后的处理以及 InitializingBean 和 init-method 的初始化操作；
- 销毁的具体操作，有注册相关销毁回调接口，最后通过DisposableBean 和 destory-method 进行销毁。
