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







-----

[一文整理总结常见Java后端面试题系列——Spring篇（2022最新版）_程序猿周周的博客-CSDN博客_前后端分离面试题java](https://blog.csdn.net/adminpd/article/details/123016872)

