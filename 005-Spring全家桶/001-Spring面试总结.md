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

Spring 的依赖注入分为**`接口注入（Interface Injection）`、`Setter 方法注入（Setter Injection）` 和`构造器注入（Constructor Injection）`** 三种方式。其中接口注入由于在灵活性和易用性比较差，现在从 Spring4 开始已被废弃。

- **构造器依赖注入**：构造器依赖注入通过容器触发一个类的构造器来实现的，该类有一系列参数，每个参数代表一个对其他类的依赖。

- **Setter方法注入**：Setter 方法注入是容器通过调用无参构造器或无参 static 工厂方法实例化 Bean 之后，调用该 Bean 的 setter 方法实现。

二者区别：

- 构造函数注入不支持部分注入，而 Setter 方法注入支持。
- 构造函数注入不会覆盖 setter 属性，而 Setter 方法会。
- 构造函数注入任意修改都会创建一个新实例，而 Setter 方法不会。
- 构造函数注入适用于设置大量属性，Setter 方法使用与设置少量属性。

-----

[一文整理总结常见Java后端面试题系列——Spring篇（2022最新版）_程序猿周周的博客-CSDN博客_前后端分离面试题java](https://blog.csdn.net/adminpd/article/details/123016872)

