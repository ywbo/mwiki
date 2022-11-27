# Spring 面试总结

## 一、基础

### 1. Spring 是什么？

Spring 是一个轻量级 Java 开发框架，最早有 Rod Johnson 创建，目的是为了解决企业级应用开发的业务逻辑层和其他各层的耦合问题。它是一个分层的 JavaSE/JavaEE full-stack（一站式）轻量级开源框架，为开发 Java 应用程序提供全面的基础架构支持。Spring 负责基础架构，因此 Java 开发者可以专注于应用程序的开发。

> **Spring最根本的使命是解决企业级应用开发的复杂性，即简化Java开发。**

-----

### 2. Spring带来了哪些好处？

- 基于 POJO 的轻量级和最小侵入性编程。
- DI 机制将对象之间的依赖关系交由框架处理，减低组件间的耦合性。
- 基于 AOP 技术支持将一些通用任务，如安全、事务、日志、权限等进行集中式管理，从而提供更好的复用。
- 对于主流的应用框架提供了集成支持。

----

### 3. Spring都有哪些模块？

![img](https://img-blog.csdnimg.cn/20201209012554205.png)

上图对应的是 Spring 4.x 版本的架构图，主要包括以下八个模块：

- **Spring Core**：基础，提供 IOC 和 DI 能力，可以说 Spring 其他所有的功能都依赖于该类库。

- **Spring Aspects**：该模块为集成 AspectJ 提供支持。

- **Spring AOP**：提供面向方面的编程实现。

- **Spring JDBC**：Java 数据库连接。

- **Spring JMS**：Java 消息服务。

- **Spring ORM**：用于支持 Hibernate、Mybatis 等 ORM 工具。

- **Spring Web**：为创建 Web 应用程序提供支持。

- **Spring Test**：提供了对 JUnit 和 TestNG 测试框架的支持。

------

### 4、Spring 中使用了哪些设计模式？

- **工厂模式**：包括简单工厂和工厂方法，如通过 BeanFactory 或 ApplicationContext 创建 Bean 对象。

- **单例模式**：Spring 中的 Bean 对象默认就是单例模式。

- **代理模式**：Spring AOP 就是基于代理实现的，包括 JDK 动态代理和 CGlib 技术。

- **模板方法模式**：Spring 中 jdbcTemplate 等以 Template 结尾对数据库操作的类就使用到模板模式。

- **观察者模式**：Spring 事件驱动模型就是观察者模式很经典的应用。

- **适配器模式**：Spring MVC 中，DispatcherServlet 根据请求解析到对应的Handler（也就是我们常说的 Controller）后，开始由 HandlerAdapter 适配器处理。

- **装饰者模式**：使用 DataSource 在不改动代码情况下切换数据源。

- **策略模式**：Spring 对资源的访问，如 Resource 接口。

-----

### 5、Spring 中有哪些不同类型事件？

Spring 提供了以下 5 种标准的事件：

- **上下文更新事件（ContextRefreshedEvent）**：在调用ConfigurableApplicationContext 接口中的 refresh() 方法时被触发。

- **上下文开始事件（ContextStartedEvent）**：当容器调用ConfigurableApplicationContext的Start()方法开始/重新开始容器时触发该事件。

- **上下文停止事件（ContextStoppedEvent）**：当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件。

- **上下文关闭事件（ContextClosedEvent）**：当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁。

- **请求处理事件（RequestHandledEvent）**：在Web应用中，当一个http请求（request）结束触发该事件。

  

  至于如果监听这些事件：

  一个 Bean 实现了 ApplicationListener 接口，当一个 ApplicationEvent 被发布以后，Bean 会自动被通知。

----

## 二、IOC

### 1. 什么是IOC？

Ioc 是 Inversion of Control 的缩写，即**控制反转**。Ioc 不是一项技术，而是一种**设计思想**。在 Java 开发中，Ioc 意味着你可以将设计好的对象交给 Ioc 容器，完成初始化和管理，当你需要时由容器提供控制。

Spring IOC 可谓是 Spring 的核心，对于 Spring 框架而言，所谓 Ioc 就是由 Spring 来负责控制对象的生命周期和对象间的关系。**正这个控制过程中，需要动态的向某个对象提供它所需要的其他对象，这一点是通过 DI（Dependency Injection，依赖注入）来实现的**。
----

### 2. IOC 作用或者好处？

实现对象间的解耦，同时降低**应用开发**的代码量和复杂度，使开发人员更专注业务。

----

### 3. IOC的实现原理？

Spring 的 IOC 是基于**工厂设计模式**再加上**反射**实现。

----

### 4. Spring有哪些类容器？

- **BeanFactory**：这是一个最简单的容器，它主要的功能是为依赖注入（DI）提供支持。
- **ApplicationContext**：Application Context 是 Spring 中的高级容器。和 BeanFactory 类似，它可以加载和管理配置文件中定义的 Bean。 另外，它还增加了企业所需要的功能，比如，从属性文件中解析文本信息和将事件传递给所指定的监听器。

一些常被使用的 ApplicationContext 实现类：

- **FileSystemXmlApplicationContext**：该容器从 XML 文件中加载已被定义的 Bean， 需要提供 XML 文件的完整路径。
- **ClassPathXmlApplicationContext**：同样从 XML 文件中加载已被定义的 Bean，但无需提供完整路径，因为它会从 CLASSPATH 中搜索配置文件。
- **WebXmlApplicationContext**：该容器会在一个 Web 应用程序的范围内加载在 XML 文件中已被定义的 Bean。

----

### 5. BeanFactory 和 ApplicationContext 的区别？

二者都是 Spring 框架的两大核心接口，都可以当做 Spring 的容器。其中 ApplicationContext 是 BeanFactory 的子接口。

- BeanFactory 是 Spring 里面最底层的接口，包含了各种 Bean 的定义，读取配置文档，管理 Bean 的加载、实例化，控制 Bean 的生命周期，维护对象之间的依赖关系等功能。

- ApplicationContext 接口作为 BeanFactory 的派生，除了提供 BeanFactory 所具有的功能外，还提供了更完整的框架功能：
  - 继承 MessageSource，支持国际化。
  - 统一的资源文件访问方式。
  - 提供在监听器中注册 Bean 的事件。
  - 支持同时加载多个配置文件。
  - 载入多个（有继承关系）上下文，使得每一个上下文都专注于一个特定的层次，如应用的 Web 层。

**具体区别体现在以下三个方面：**

- **加载方式不同**
  - BeanFactroy 采用的懒加载方式注入 Bean，即只有在使用到某个 Bean 时才对该 Bean 实例化。这样，我们就不能在程序启动时发现一些存在的 Spring 的配置问题。
  - ApplicationContext 是在启动时一次性创建了所有的 Bean。

- **创建方式不同**
  - BeanFactory 通常以编程的方式被创建，
  - ApplicationContext 还能以声明的方式创建，如使用 ContextLoader。

- **注册方式不同**
  二者都支持 BeanPostProcessor、BeanFactoryPostProcessor 的使用，但 BeanFactory 需要手动注册，而 ApplicationContext 则是自动注册。

----

### 6. 有那些注入方式，以及它们的区别是什么？

Spring 的依赖注入分为**`接口注入（Interface Injection）`、`Setter 方法注入（Setter Injection）` 和`构造器注入（Constructor Injection）`** 三种方式。其中**接口注入**由于在灵活性和易用性比较差，现在从 Spring4 开始**已被废弃**。

- **构造器依赖注入**：构造器依赖注入通过容器触发一个类的构造器来实现的，该类有一系列参数，每个参数代表一个对其他类的依赖。

- **Setter方法注入**：Setter 方法注入是容器通过调用无参构造器或无参 static 工厂方法实例化 Bean 之后，调用该 Bean 的 setter 方法实现。

二者区别：

- 构造函数注入不支持部分注入，而 Setter 方法注入支持。
- 构造函数注入不会覆盖 setter 属性，而 Setter 方法会。
- 构造函数注入任意修改都会创建一个新实例，而 Setter 方法不会。
- 构造函数注入适用于设置大量属性，Setter 方法使用与设置少量属性。

---

## 三、Bean

### 1. Spring Bean 的作用域有哪些？

Spring 提供以下五种 Bean 的作用域：

- **Singleton**: Bean 在每个 Spring Ioc 容器中只有一个实例，也是 Spring 的默认配置。
- **Prototype**：一个 Bean 的定义可以有多个实例。
- **Request**：每次 Http 请求都会创建一个 Bean，故该作用域仅在基于 Web 的 Spring ApplicationContext情形下有效。
- **Session**：在一个 Http Session 中，一个 Bean 对应一个实例。该作用域同样仅在基于 Web 的 Spring ApplicationContext 情形下有效。
- **Global-session**：在一个全局的 Http Session 中，一个 Bean 定义对应一个实例。

值的注意的是：使用 Prototype 作用域时需要慎重的思考，因为频繁创建和销毁 Bean 会带来很大的性能开销。

---

### 2. Spring 的单例是否线程安全？

可以肯定的是，**Spring 中的单例 Bean 并不是线程安全的**。

但我们日常使用时往往并未做多线程并发处理，那又是如何保证线程安全的呢？

实际上大部分时候我们定义的 Bean 是无状态的（如 dao 类），所以在某种程度上来说，Bean 也是安全的，但如果 Bean 有状态的话（比如 model 对象），那就要开发者自己去保证线程安全了。

【注】`其中有状态就是有数据存储功能，无状态就是不会。`
-----

### 3. Spring Bean 的生命周期？

![img](https://img-blog.csdnimg.cn/201911012343410.png)

Bean 在 Spring 容器中从创建到销毁经历了若干阶段，每一阶段都可以进行个性化定制。

1）Spring 对 Bean 进行实例化；

2）Spring 将配置和 Bean 的引用注入到对应的属性中；

3）如果 Bean 实现了 BeanNameAware 接口，Spring 将 Bean 的 ID 传递给 setBeanName() 方法；

4）如果 Bean 实现了 BeanFactoryAware 接口，Spring 将调用 setBeanFactory() 方法将 BeanFactory 容器实例传入；

5）如果 Bean 实现了 ApplicationContextAware 接口，Spring 将调用 setApplicationContext() 方法将 Bean 所在的应用上下文的引用传入进来；

6）如果 Bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的 postProcessBeforeInitialization() 方法；

7）如果 Bean 实现了 InitializingBean 接口，Spring 将调用它们的 afterPropertiesSet() 方法。类似地，如果 Bean 使用 initmethod 声明了初始化方法，该方法也会被调用；

8）如果 Bean 实现了 BeanPostProcessor 接口，Spring 将调用它们的postProcessAfterInitialization()方法；

9）此时，Bean 已经准备就绪，可以被应用程序使用了，它们将一直驻留在应用上下文中，直到该应用上下文被销毁；

10）如果 Bean 实现了 DisposableBean 接口，Spring 将调用它的 destroy() 接口方法。同样，如果使用 destroymethod 声明了销毁方法，该方法也会被调用。

----

## 四、AOP

### 1. 什么是 AOP，以及AOP的作用是什么？

AOP 全称（Aspect Oriented Programming），是**面向切片编程**的简称。

传统的 OOP 开发中代码逻辑是至上而下的过程中会长生一些横切性问题（大量与业务无关的重复代码），这些横切问题会散落在代码的各个地方且难以维护。AOP 的编程思想就是把业务逻辑和横切的问题进行分离，从而达到解耦的目的，使代码的重用性和开发效率高（目的是重用代码，把公共的代码抽取出来）。

即**AOP 的作用是对业务逻辑的各个部分进行隔离，降低业务逻辑的耦合性，提高程序的可重用型和开发效率。**
----

### 2. AOP有哪些应用场景？

- 日志记录
- 权限校验和管理
- 缓存
- 事务

----

### 3. 切面、切点、连接点、通知四者的关系是什么？

- 切面：拦截器类，其中会定义切点以及通知
- 切点：具体拦截的某个业务点（方法）
- 通知：声明通知方法在目标业务层的执行位置

---

### 4. AOP的实现原理是什么？

AOP 是基于代理实现的，Spring 提供了两种方式来生成代理对象：

- **JDK 动态代理**：利用拦截器加反射机制生成一个实现代理接口的匿名类。

- **CGlib**：利用 ASM 开源包修改字节码生成子类，且不支持 Final 修饰的方法。

**默认策略**是如果目标类是**接口**，则使用 JDK 动态代理技术，否则使用 Cglib 来生成代理。

CGlib 是否比 JDK 快？

CGlib 采用字节码技术生成代理，在 JDK 6 之前确实比使用 Java 反射效率要高。但随着 JDK 版本的每一次升级，JDK 代理效率都得到提升，在 JDK 8 时效率已经高于 CGlib，而 CGLIB 代理消息确有点跟不上步伐。

【REF】：[Spring AOP 实现原理](https://blog.csdn.net/moreevan/article/details/11977115/) 

----

## 五、事务

### 1. Spring 事务的开启方式？

- **编程式事务**：编码方式实现事务管理（PlatfromTransactionManager）
- **声明式事务**

可知编程式事务每次实现都要单独实现，但业务量大功能复杂时，使用编程式事务无疑是痛苦的，而声明式事务不同，声明式事务属于无侵入式，不会影响业务逻辑的实现。

【REF】：[spring事务的开启方式（编程式和声明式）](https://www.cnblogs.com/wangjing666/p/9655843.html) 

---

### 2. Spring 事务的隔离级别？

Spring 事务隔离级别比数据库事务隔离级别多一个 Default。

- DEFAULT （默认）
  这是一个 PlatfromTransactionManager 默认的隔离级别，**使用数据库默认的事务隔离级别**。另外四个与 JDBC 的隔离级别相对应。

- READ_UNCOMMITTED （读未提交）

- READ_COMMITTED （读已提交）

- REPEATABLE_READ （可重复读）

- SERIALIZABLE（串行化）

----

### 3. Spring 事务的传播级别？

**默认 Required 级别。**

|   **级别**    |                           **描述**                           |
| :-----------: | :----------------------------------------------------------: |
| **REQUIRED**  |         支持当前事务，如果没有事务会创建一个新的事务         |
|   SUPPORTS    |        支持当前事务，如果没有事务的话以非事务方式执行        |
|   MANDATORY   |              支持当前事务，如果没有事务抛出异常              |
| REQUIRES_NEW  |                创建一个新的事务并挂起当前事务                |
| NOT_SUPPORTED |      以非事务方式执行，如果当前存在事务则将当前事务挂起      |
|     NEVER     |           以非事务方式进行，如果存在事务则抛出异常           |
|    NESTED     | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作 |

NESTED 与 REQUIRES_NEW 的区别：

二者非常类似，都像一个嵌套事务，如果不存在一个活动的事务，都会开启一个新的事务。但

- 1）使用 REQUIRES_NEW 时，内层事务与外层事务就像两个独立的事务一样，一旦内层事务进行了提交后，外层事务不能对其进行回滚。两个事务互不影响。两个事务不是一个真正的嵌套事务。同时它需要 JTA 事务管理器的支持。

- 2）使用 NESTED 时，外层事务的回滚可以引起内层事务的回滚。而内层事务的异常并不会导致外层事务的回滚，它是一个真正的嵌套事务。

【REF】：[spring事务传播机制](https://blog.csdn.net/nhlbengbeng/article/details/87797781) 
---

### 4. Spring 事务失效的原因？

- **异常类型不对**：默认支持回滚的是 Runtime 异常，或**异常被业务捕获**；
- **数据源不支持事务**：如 MySQL 未开启事务或使用 MyISAM 存储引擎；
- **非 Public 方法不支持事务**；
- **传播类型不支持事务**；
- **事务未被 Spring 接管**；

-----

【REF】：[Spring篇（2022最新版）](https://blog.csdn.net/adminpd/article/details/123016872) 

