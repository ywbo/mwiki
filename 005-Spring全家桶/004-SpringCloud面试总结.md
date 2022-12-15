# SpringCloud 知识总结

## 1、什么是SpringCloud？

> SpringCloud是基于SpringBoot基础上开发的微服务框架，SpringCloud是一套目前非常完整的微服务解决方案框架，其内容包含：`服务治理`，`注册中心`，`配置管理`，`断路器`，`智能路由`，`微代理`，`控制总线`，`全局锁`，`分布式会话`等。

SpringCloud包含众多的子项目

SpringCloud config 分布式配置中心

SpringCloud Netflix 核心组件

​                      Eureka:服务治理 注册中心

​                      Hystrix:服务保护框架

​                      Ribbon:客户端负载均衡器

​                      Feign：基于ribbon和hystrix的声明式服务调用组件

​                      Zuul: 网关组件,提供智能路由、访问过滤等功能。

![image-20221210230517871](image-20221210230517871.png)



## 2、SpringCloud 组件都有哪些？

- ### SpringCloud - Eureka：服务治理，注册中心  

  - 服务治理：在传统rpc远程调用中，服务与服务依赖关系，管理比较复杂，所以需要使用服务治理，管理服务与服务之间依赖关系，可以实现服务调用、负载均衡、容错等，实现服务发现与注册。

  - 注册中心：在服务注册与发现中，有一个注册中心，当服务器启动的时候，会把当前自己服务器的信息 比如 服务地址通讯地址等以别名方式注册到注册中心上。

     另一方（消费者|服务提供者），以该别名的方式去注册中心上获取到实际的服务通讯地址，让后在实现rpc调用。

  - **Eureka Client**：负责将这个服务的信息注册到 Eureka Server中

  - **Eureka Server**：注册中心，里面有个注册表，保存了各个服务所在的机器和端口号

  

  - 这里是可以使用`ZooKeeper`来`替换`掉Eureka的。
    - Zookeeper是一个分布式协调工具，可以实现服务注册与发现、注册中心、消息中间件、分布式配置中心等。

- ### SpringCloud - Feign

  - **Feign的一个关键机制就是使用了动态代理**。
    - 首先，如果你对某个接口定义了@FeignClient注解，Feign就会针对这个接口创建一个动态代理；
    - 接着你要是调用那个接口，本质就是会调用 Feign创建的动态代理，这是核心中的核心；
    - Feign的动态代理会根据你在接口上的@RequestMapping等注解，来动态构造出你要请求的服务的地址；
    - 最后针对这个地址，发起请求、解析响应；

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC8xMS83LzE2NmViZmZmNTA1YjJhMjA?x-oss-process=image/format,png)

- ### SpringCloud - Ribbon

  服务部署在多台机器上，当请求过来，怎么知道具体该去请求哪台机器呢？

  - 这时Spring Cloud Ribbon就派上用场了。Ribbon就是专门解决这个问题的。它的作用是**负载均衡**，会帮你在每次请求时选择一台机器，均匀的把请求分发到各个机器上；

  - Ribbon的负载均衡默认使用的最经典的**Round Robin轮询算法**。这是啥？简单来说，就是如果订单服务对库存服务发起10次请求，那就先让你请求第1台机器、然后是第2台机器、第3台机器、第4台机器、第5台机器，接着再来—个循环，第1台机器、第2台机器。。。以此类推。

  - 此外，Ribbon是和Feign以及Eureka紧密协作，完成工作的，具体如下：

    首先Ribbon会从 Eureka Client里获取到对应的服务注册表，也就知道了所有的服务都部署在了哪些机器上，在监听哪些端口号。

  - 然后Ribbon就可以使用默认的Round Robin算法，从中选择一台机器

  - Feign就会针对这台机器，构造并发起请求。

  对于上述整个过程，再来一张图

  ![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC8xMS83LzE2NmVjMDAxZGMxNTVlOTg?x-oss-process=image/format,png)

  

## 3、分别说说平时都用到哪些？

