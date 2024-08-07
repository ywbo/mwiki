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

- ### SpringCloud - Hytrix

  - Hystrix是隔离、熔断以及降级的一个`框架`。啥意思呢？说白了，**Hystrix会搞很多个小小的线程池**，比如订单服务请求库存服务是一个线程池，请求仓储服务是一个线程池，请求积分服务是一个线程池。每个线程池里的线程就仅仅用于请求那个服务。

    - 现在很不幸，积分服务挂了，会咋样？

      当然会导致订单服务里那个用来调用积分服务的线程都卡死不能工作了啊！但由于订单服务调用库存服务、仓储服务的这两个线程池都是正常工作的，所以这两个服务不会受到任何影响。

      这个时候如果别人请求订单服务，订单服务还是可以正常调用库存服务扣减库存，调用仓储服务通知发货。只不过调用积分服务的时候，每次都会报错。但是如果积分服务都挂了，每次调用都要去卡住几秒钟干啥呢？有意义吗？当然没有！所以我们直接对积分服务熔断不就得了，比如在5分钟内请求积分服务直接就返回了，不要去走网络请求卡住几秒钟，这个过程，就是所谓的熔断！

      那人家又说，兄弟，积分服务挂了你就熔断，好歹你干点儿什么啊！别啥都不干就直接返回啊？没问题，咱们就来个降级：每次调用积分服务，你就在数据库里记录一条消息，说给某某用户增加了多少积分，因为积分服务挂了，导致没增加成功！这样等积分服务恢复了，你可以根据这些记录手工加一下积分。这个过程，就是所谓的降级。

- ### SpringCloud - Zuul

  - Zuul，也就是微服务网关。**这个组件是负责网络路由的。**

    - 什么是网络路由？

      - 我们假设这么一个场景：假设你后台部署了几百个服务，现在有个前端兄弟，人家请求是直接从浏览器那儿发过来的。打个比方：人家要请求一下库存服务，你难道还让人家记着这服务的名字叫做inventory-service？部署在5台机器上？就算人家肯记住这一个，你后台可有几百个服务的名称和地址呢？难不成人家请求一个，就得记住一个？你要这样玩儿，那真是友谊的小船，说翻就翻！

        上面这种情况，压根儿是不现实的。所以一般微服务架构中都必然会设计一个网关在里面，像android、ios、pc前端、微信小程序、H5等等，不用去关心后端有几百个服务，就知道有一个网关，所有请求都往网关走，网关会根据请求中的一些特征，将请求转发给后端的各个服务。

        而且有一个网关之后，还有很多好处，比如可以做统一的降级、限流、认证授权、安全，等等。

- ### 总结

  最后再来总结一下，上述几个Spring Cloud核心组件，在微服务架构中，分别扮演的角色：

  - `Eureka`：各个服务启动时，Eureka Client都会将服务注册到Eureka Server，并且Eureka Client还可以反过来从Eureka Server拉取注册表，从而知道其他服务在哪里
  - `Ribbon`：服务间发起请求的时候，基于Ribbon做负载均衡，从一个服务的多台机器中选择一台
  - `Feign`：基于Feign的动态代理机制，根据注解和选择的机器，拼接请求URL地址，发起请求
  - `Hystrix`：发起请求是通过Hystrix的线程池来走的，不同的服务走不同的线程池，实现了不同服务调用的隔离，避免了服务雪崩的问题
  - `Zuul`：如果前端、移动端要调用后端系统，统一从Zuul网关进入，由Zuul网关转发请求给对应的服务
    - [REF]([SpringCould组件有哪些，他们的作用是什么？（面试常问框架没有之一）https://blog.csdn.net/tzydzj/article/details/113337606) 

----

## 3、分别说说平时都用到哪些？



## 4、使用过程中是怎么理解这些组件的？