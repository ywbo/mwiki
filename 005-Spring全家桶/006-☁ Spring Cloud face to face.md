# ☁ Spring Cloud face to face

[TOC]

# 1. <font color='#C5564B'>什么是微服务</font>

<font color='red'>`微服务架构是一种架构模式或者说是一种架构风格`</font>，它<font color='red'>`提倡将单一应用程序划分为一组小的服务`</font>，每个服务运行在其独立的自己的进程中，<font color='red'>`服务之间相互协调、互相配合，为用户提供最终价值`</font>。<font color='red'>`服务之间采用轻量级的通信机制互相沟通（通常是基于HTTP的RESTful API）`</font>，每个服务都围绕着具体的业务进行构建，并且能够被独立的构建在生产环境、类生产环境等。

对具体的一个服务而言，应根据业务上下文，选择合适的语言、工具对其进行构建，可以有一个非常轻量级的集中式管理来协调这些服务，可以使用不同的语言来编写服务，也可以使用不同的数据存储。

# 2. <font color='#C5564B'>微服务相较于传统的web项目有什么优势</font>

传统的Web项目尽管也是遵循模块化开发，但最终它们会打包并部署为单体式应用。例如Java应用程序会被打包成WAR，部署在Tomcat或者Jetty上。

这种单体应用比较适合于小项目，优点是：

- 开发简单直接，集中式管理
- 基本不会重复开发
- 功能都在本地，没有分布式的管理开销和调用开销

它的缺点也十分明显，特别对于互联网公司来说：

- 开发效率低：所有的开发在一个项目改代码，递交代码相互等待，代码冲突不断
- 代码维护难：代码功能耦合在一起，新人不知道何从下手
- 部署不灵活：构建时间长，任何小修改必须重新构建整个项目，这个过程往往很长
- 稳定性不高：一个微不足道的小问题，可以导致整个应用挂掉
- 扩展性不够：无法满足高并发情况下的业务需求

主流的设计一般会采用**微服务架构**。其思路不是开发一个巨大的单体式应用，而是将应用分解为小的、互相连接的微服务。一个微服务完成某个特定功能，每个微服务都有自己的业务逻辑。一些微服务还会提供API接口给其他微服务和应用客户端使用

微服务架构的优点：

- 解决了复杂性问题：**开发的速度要快很多**，更容易理解和维护。
- **单独开发每个服务，与其他服务互不干扰**
- 可以独立部署每个微服务

微服务的缺点：

- 多服务运维难度
- **系统部署依赖**
- **服务间通信成本**
- **数据一致性**
- 系统集成测试
- 性能监控

# 3. <font color='#C5564B'>实现微服务要解决的四个问题？</font>

## <font color='#B96AD9'>**客户端如何访问这些服务**</font>

原来的服务都是可以进行单独调用，现在按功能拆分成独立的服务，变成了一个独立的Java进程了。客户端如何访问后台N个服务呢？

一般在后台N个服务和UI之间会一个代理或者叫**<font color='red'>API Gateway</font>**，它的作用包括：

- **<font color='red'>提供统一服务入口</font>**，让微服务对前台透明
- 聚合后台的服务，节省流量，提升性能
- **<font color='red'>提供安全，过滤，流控</font>**等API管理功能

## **<font color='#B96AD9'>服务之间如何通信</font>**

因为所有的微服务都是独立的Java进程跑在独立的虚拟机上，所以服务间的通信就是IPC（inter process communication，进程间通信），已经有很多成熟的方案。现在基本最通用的有两种方式：

- 同步调用
  - **<font color='red'>REST</font>**
  - **<font color='red'>RPC</font>**
- 异步**<font color='red'>消息调用</font>**（Kafka, ActiveMQ、RocketMQ）

## **<font color='#B96AD9'>这么多服务，怎么找?</font>**

在微服务架构中，一般每一个服务都是有多个拷贝，来做负载均衡。一个服务随时可能下线，也可能应对临时访问压力增加新的服务节点。服务之间如何相互感知？服务如何管理？这就是**<font color='red'>服务发现</font>**的问题了。

一般都是通过zookeeper等类似技术做服务注册信息的分布式管理。当服务上线时，服务提供者将自己的服务信息注册到ZK（或类似框架），并通过心跳维持长链接，实时更新链接信息。服务调用者通过ZK寻址，根据可定制算法，找到一个服务，还可以将服务信息缓存在本地以提高性能。当服务下线时，ZK会发通知给服务客户端。

- 客户端做：优点是架构简单，扩展灵活，只对服务注册器依赖。缺点是客户端要维护所有调用服务的地址，有技术难度，一般大公司都有成熟的内部框架支持，比如Dubbo。
- 服务端做：优点是简单，所有服务对于前台调用方透明，一般在小公司在云服务上部署的应用采用的比较多。

## **<font color='#B96AD9'>服务挂了怎么办?</font>**

分布式最大的特性就是网络是不可靠的。通过微服务拆分能降低这个风险，不过如果没有特别的保障，结局肯定是噩梦。

当我们的系统是由一系列的服务调用链组成的时候，我们必须确保任一环节出问题都不至于影响整体链路。相应的手段有很多：

- **<font color='red'>重试机制</font>**
- **<font color='red'>限流</font>**
- **<font color='red'>熔断机制</font>**
- **<font color='red'>负载均衡</font>**
- 降级（本地缓存）

# 4. <font color='#C5564B'>分布式和微服务有什么区别？</font>

它们的本质的区别体现在“目标”上。

分布式架构的目标就是访问量很大一台机器承受不了，或者是成本问题，不得不使用多台机器来完成服务的部署；

微服务的目标只是让各个模块拆分开来，不会被互相影响，比如模块的升级或者出现BUG或者是重构等等都不要影响到其他模块，微服务它是可以在一台机器上部署；

**<font color='red'>分布式也是微服务的一种，微服务也属于分布式。</font>**

# 5. <font color='#C5564B'>什么是Spring Cloud?</font>

Spring Cloud 就是微服务系统架构的一站式解决方案，在平时我们构建微服务的过程中需要做如服务发现注册 、配置中心 、消息总线 、负载均衡 、断路器 、数据监控 等操作，而 **<font color='red'>Spring Cloud 为我们提供了一套简易的编程模型，使我们能在 Spring Boot 的基础上轻松地实现微服务项目的构建</font>**。

# 6. <font color='#C5564B'>Spring Cloud的优缺点有哪些？</font>

优点：

- **<font color='red'>服务拆分粒度更细，有利于资源重复利用</font>**，有利于提高开发效率
- 可以**<font color='red'>更精准的制定优化服务方案</font>**，提高系统的可维护性
- 微服务架构采用去中心化思想，服务之间采用Restful等轻量级通讯，比ESB更轻量
- 适于互联网时代，**<font color='red'>产品迭代周期更短</font>**

缺点：

- 微服务过多，`治理成本高`，不利于维护系统
- 分布式系统`开发的成本高`（容错，分布式事务等），对团队挑战大

总的来说优点大过于缺点。

# 7. <font color='#C5564B'>SpringBoot和SpringCloud的区别？</font>

- SpringBoot专注于快速方便的开发单个个体微服务；SpringCloud是关注全局的微服务协调整体治理框架，它**<font color='red'>将SpringBoot开发的一个个单体微服务整合并管理起来</font>**，为各个微服务之间提供，配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等集成服务
- SpringBoot可以离开SpringCloud独立使用开发项目， 但是**<font color='red'>SpringCloud离不开SpringBoot</font>** ，属于依赖的关系
- SpringBoot专注于快速、方便的开发单个微服务个体；**<font color='red'>SpringCloud关注全局的服务治理框架</font>**。

# 8. <font color='#C5564B'>Spring Cloud有哪些组件？</font>

- **<font color='red'>Eureka</font>**：服务注册与发现。
- **<font color='red'>Feign</font>**：基于动态代理机制，根据注解和选择的机器，拼接请求 url 地址，发起请求。
- **<font color='red'>Ribbon</font>**：客户端负载均衡。实现负载均衡，从一个服务的多台机器中选择一台。
- **<font color='red'>Hystrix</font>**：断路器。提供线程池，不同的服务走不同的线程池，实现了不同服务调用的隔离，避免了服务雪崩的问题。
- **<font color='red'>Zuul</font>**：服务网关。网关管理，由 Zuul 网关转发请求给对应的服务。
- **<font color='red'>Spring Cloud Config</font>**：分布式配置

# 9. <font color='#C5564B'>组件 face to face</font>

## <font color='#B96AD9'>Eureka</font>

### Eureka的服务治理是什么？

Spring Cloud 封装了 Netflix 公司开发的 Eureka 模块来实现服务治理。

在传统的rpc远程调用框架中，管理每个服务与服务之间依赖关系比较复杂，管理比较复杂，所以**<font color='red'>需要使用服务治理，管理服务与服务之间依赖关系，可以实现服务调用、负载均衡、容错等，实现服务发现与注册</font>**。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175009075-1730858796.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175009075-1730858796.png)

### Eureka的服务注册是什么？

Eureka采用了CS的设计架构，**<font color='red'>Eureka Server 作为服务注册功能的服务器，它是服务注册中心</font>**。而系统中的其他微服务，使用 Eureka的客户端连接到 Eureka Server并维持心跳连接。这样系统的维护人员就可以通过 Eureka Server 来监控系统中各个微服务是否正常运行。

在服务注册与发现中，有一个注册中心。**<font color='red'>当服务器启动的时候，会把当前自己服务器的信息 比如 服务地址、通讯地址等以别名方式注册到注册中心上</font>**。另一方（消费者|服务提供者），以该别名的方式去注册中心上获取到实际的服务通讯地址，然后再实现本地RPC调用。RPC远程调用框架核心设计思想：在于注册中心，因为使用注册中心管理每个服务与服务之间的一个依赖关系(服务治理概念)。在任何rpc远程框架中，都会有一个注册中心(存放服务地址相关信息(接口地址))

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404083314240-1462998196.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404083314240-1462998196.png)

### Eureka如何实现高可用？

**<font color='red'>搭建集群环境，服务之间相互注册</font>**。

### Eureka如何获取服务信息？

通过服务发现的方式获取，自动装配DiscoveryClient，调用里面方法即可。

```java
//获取所有服务
 List<String> services = discoveryClient.getServices();
 //指定服务名获取对应的信息
 List<ServiceInstance> instances = discoveryClient.getInstances("XIAOBEAR-CLOUD-PAYMENT-SERVICE");
```

### Eureka的自我保护模式是什么？

**<font color='red'>默认情况下，如果Eureka Server在一定时间内没有接收到某个微服务实例的心跳，Eureka Server将会注销该实例（默认90秒）</font>**。但是当网络分区故障发生(延时、卡顿、拥挤)时，微服务与Eureka Server之间无法正常通信，以上行为可能变得非常危险了—— 因为微服务本身其实是健康的，此时本不应该注销这个微服务。Eureka通过“自我保护模式”来解决这个问题——**<font color='red'>当Eureka Server节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式</font>**。

在自我保护模式中，**<font color='red'>Eureka Server会保护服务注册表中的信息，不再注销任何服务实例</font>**。

它的设计哲学就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。

综上，自我保护模式是一种应对网络异常的安全保护措施。它的架构哲学是宁可同时保留所有微服务（健康的微服务和不健康的微服务都会保留）也不盲目注销任何健康的微服务。使用自我保护模式，可以让Eureka集群更加的健壮、稳定。

### 为什么会产生Eureka的自我保护呢？

为了防止Eureka Client可以正常运行，但是 与 Eureka Server网络不通情况下，Eureka Server不会立刻将Eureka Client服务剔除。

### 如何关闭Eureka的自我保护机制？

**<font color='red'>自我保护机制默认是开启的</font>**。

关闭可以通过配置文件进行关闭：

```yaml
eureka:
  instance:
    hostname: eureka7001.com  #eureka服务端的实例名称
  client:
    register-with-eureka: false   #false表示不向注册中心注册自己。
    fetch-registry: false      #false表示自己端就是注册中心，我的职责就是维护服务实例，并不需要去检索服务
    service-url:
      defaultZone: http://eureka7002.com:7002/eureka/
       #设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址。
  server:
     #关闭自我保护机制，保证不可用服务被及时踢除
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 1000
```

### Spring Cloud如何实现服务的注册？

服务发布时，指定对应的服务名，将服务注册到 注册中心(Eureka 、Zookeeper)。

**<font color='red'>如果是使用Eureka作为注册中心，则Eureka Server需要在启动类上添加@EnableEurekaServer注解，其他服务（即Eureka Client）在启动类上添加@EnableDiscoveryClient注解，然后用ribbon或feign进行服务直接的调用</font>**。

### 请说说Eureka和zookeeper 的区别？

**<font color='red'>Zookeeper保证了CP，Eureka保证了AP</font>**。【A：高可用 C：一致性 P：分区容错性】

当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的信息，但不能容忍直接down掉不可用。也就是说，服务注册功能对高可用性要求比较高，但**<font color='red'>zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新选leader。问题在于，选取leader时间过长，30 ~ 120s，且选取期间zk集群都不可用，这样就会导致选取期间注册服务瘫痪</font>**。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够恢复，但是漫长的选取时间导致的注册长期不可用是不能容忍的。

Eureka保证了可用性，Eureka各个节点是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点仍然可以提供注册和查询服务。而**<font color='red'>Eureka的客户端向某个Eureka注册或发现时发生连接失败，则会自动切换到其他节点，只要有一台Eureka还在，就能保证注册服务可用，只是查到的信息可能不是最新的</font>**。除此之外，Eureka还有自我保护机制，如果在15分钟内超过85%的节点没有正常的心跳，那么Eureka就认为客户端与注册中心发生了网络故障，此时会出现以下几种情况：

- ① Eureka不再从注册列表中移除因为长时间没有收到心跳而应该过期的服务。
- ② Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其他节点上（即保证当前节点仍然可用）
- ③ 当网络稳定时，当前实例新的注册信息会被同步到其他节点。

因此，**<font color='red'>Eureka可以很好地应对因网络故障导致部分节点失去联系的情况，而不会像Zookeeper那样使整个微服务瘫痪</font>**。

## <font color='#B96AD9'>Consul</font>

Consul 是<font color='red'>一套开源的分布式服务发现和配置管理系统</font>。

提供了微服务系统中的<font color='red'>服务治理、配置中心、控制总线等功能</font>。这些功能中的每一个都可以根据需要单独使用，也可以一起使用以构建全方位的服务网格，总之Consul提供了<font color='red'>一种完整的服务网格解决方案</font>。

## <font color='#B96AD9'>Ribbon</font>

### 什么是Spring Cloud Ribbon？

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套**<font color='red'>客户端负载均衡的工具</font>**。

简单的说，Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175100588-1846210789.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175100588-1846210789.png)

### Ribbon的本质是什么？

Ribbon就是负载均衡+RestTemplate调用

### Ribbon负载均衡策略有哪些？

- **<font color='red'>RoundRobinRule（轮询策略）</font>**：轮询，按照顺序依次选择
- **<font color='red'>RandomRule（随机策略）</font>**：随机，随机选择一个服务
- **<font color='red'>RetryRule（重试策略）</font>**：先按照RoundRobinRule的 策略获取服务，如果服务获取失败，则在指定的时间内重试，获取可用的服务
- **<font color='red'>RestAvailableRule（最小连接策略）</font>**：先过滤调由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
- **<font color='red'>AvailabilityFulteringRule（可用性敏感策略）</font>**：先过滤调故障实例，再选择并发量最小的实例
- **<font color='red'>WeightedResponseTimeRule（权重策略）</font>**：对RoundRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择。它的实现原理是，刚开始使用轮询策略并开启一个计时器，每一段时间收集一次所有服务提供者的平均响应时间，然后再给每个服务提供者附上一个权重，权重越高被选中的概率也越大。
- **<font color='red'>ZoneAvoidanceRule（区域敏感策略）</font>**：默认规则，复合判断server所在区域的性能和server的可用性选择服务器

### Ribbon底层实现原理？

ribbon实现的关键点是为ribbon定制的RestTemplate，ribbon利用了RestTemplate的拦截器机制，在拦截器中实现ribbon的负载均衡。负载均衡的基本实现就是利用applicationName从服务注册中心获取可用的服务地址列表，然后通过一定算法负载，决定使用哪一个服务地址来进行http调用。

**<font color='red'>@LoadBalanced底层其实就是个拦截器，拦截了所有的RestTemplate调用的接口，在通过调用的是哪个服务来判断对应的ip，拼接好正确的url之后，在通过HTTP请求再去请求对应的接口就可以了</font>**。

### Ribbon本地负载均衡客户端 VS Nginx服务端负载均衡区别？

- **<font color='red'>Nginx是服务器负载均衡</font>**，客户端所有请求都会交给nginx，然后由nginx实现转发请求。即负载均衡是由服务端实现的。
- **<font color='red'>Ribbon本地负载均衡</font>**，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用技术。

## <font color='#B96AD9'>Feign</font>

### Spring Cloud Feign是什么？

**<font color='red'>Feign是一个声明式WebService客户端。使用Feign能让编写Web Service客户端更加简单</font>**。

它的使用方法是定义一个服务接口然后在上面添加注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175125405-1252713306.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175125405-1252713306.png)

### Feign与OpenFeign的区别？

Feign是Netflix公司写的，是Spring Cloud组件中的**<font color='red'>一个轻量级RESTful的HTTP服务客户端</font>**；**<font color='red'>Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务</font>**。Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以调用服务注册中心的服务。

```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

**<font color='red'>OpenFeign是SpringCloud自己研发的，在Feign的基础上支持了SpringMVC的注解</font>**，如@RequesMapping等等。OpenFeign的@FeignClient可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。

```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### OpenFeign的超时控制你了解？

**<font color='red'>默认Feign客户端只等待一秒钟</font>**，但是服务端处理需要超过1秒钟，导致Feign客户端不想等待了，直接返回报错。

为了避免这样的情况，有时候我们需要设置Feign客户端的超时控制。

```yaml
#设置feign客户端超时时间(OpenFeign默认支持ribbon)
ribbon:
   #指的是建立连接所用的时间，适用于网络状况正常的情况下,两端连接所用的时间
  ReadTimeout: 5000
   #指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
```

## <font color='#B96AD9'>Hystrix</font>

### 什么是Hystrix断路器？

<font color='red'>Hystrix是一个用于处理分布式系统的**延迟和容错**的开源库</font>，在分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，<font color='red'>Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障，以提高分布式系统的弹性</font>。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175143586-1788053619.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175143586-1788053619.png)

Hystrix在SpringCloud的使用一般有两种形式：

- #### <font color='red'>Hystrix + Ribbon</font>

```xml
<!--eureka客户端，自带ribbon-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>1.4.6.RELEASE</version>
</dependency>
<!--熔断器-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
    <version>1.4.4.RELEASE</version>
</dependency>
```

- #### <font color='red'>Hystrix + OpenFeign</font>（默认是支持hystrix的，它没有默认打开，需要在配置文件打开）

```xml
<dependency> <!-- hystrix -->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<dependency> <!-- openfeign -->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```



```plaintext
# 开启feign下面的hystrix功能
feign.hystrix.enabled: true
```

### Hystrix实现延迟和容错的方法有哪些？

- 包裹请求：使用HystrixCommand包裹对依赖的调用逻辑，每个命令在独立线程中执行。这使用了设计模式中的“命令模式”。
- 跳闸机制：当某服务的错误率超过一定的阈值时，Hystrix可以自动或手动跳闸，停止请求该服务一段时间。
- <font color='red'>资源隔离</font>：Hystrix为每个依赖都维护了一个小型的线程池（或者信号量）。如果该线程池已满，发往该依赖的请求就被立即拒绝，而不是排队等待，从而加速失败判定。
- 监控：Hystrix可以近乎实时地监控运行指标和配置的变化，例如成功、失败、超时、以及被拒绝的请求等。
- <font color='red'>降级机制</font>：当请求失败、超时、被拒绝，或当断路器打开时，执行降级逻辑。降级逻辑由开发人员自行提供，例如返回一个缺省值。
- 自我修复：断路器打开一段时间后，会自动进入“半开”状态。

```java
@HystrixCommand(fallbackMethod = "str_fallbackMethod",
	groupKey = "strGroupCommand",  // groupKey的默认值是使用@HystrixCommand标注的方法所在的类名
	commandKey = "strCommand",  //commandKey的默认值是@HystrixCommand标注的方法名，即每个方法会被当做一个HystrixCommand
	threadPoolKey = "strThreadPool",  //threadPoolKey没有默认值，其实是和groupKey保持一致
	commandProperties = {                // 设置隔离策略，THREAD 表示线程池 SEMAPHORE：信号池隔离                
		@HystrixProperty(name = "execution.isolation.strategy", value = "THREAD"), // 当隔离策略选择信号池隔离的时候，用来设置信号池的大小（最大并发数）                
		@HystrixProperty(name = "execution.isolation.semaphore.maxConcurrentRequests", value = "10"), // 配置命令执行的超时时间	
		@HystrixProperty(name = "execution.isolation.thread.timeoutinMilliseconds", value = "10"), // 是否启用超时时间
		@HystrixProperty(name = "execution.timeout.enabled", value = "true"),                // 执行超时的时候是否中断
		@HystrixProperty(name = "execution.isolation.thread.interruptOnTimeout", value = "true"),                // 执行被取消的时候是否中断               
		@HystrixProperty(name = "execution.isolation.thread.interruptOnCancel", value = "true"),                // 允许回调方法执行的最大并发数               
		@HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "10"),                // 服务降级是否启用，是否执行回调函数                
		@HystrixProperty(name = "fallback.enabled", value = "true"),                // 是否启用断路器        
		@HystrixProperty(name = "circuitBreaker.enabled", value = "true"),                // 该属性用来设置在滚动时间窗中，断路器熔断的最小请求数。例如，默认该值为 20 的时候，                // 如果滚动时间窗（默认10秒）内仅收到了19个请求， 即使这19个请求都失败了，断路器也不会打开。
		@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"),                // 该属性用来设置在滚动时间窗中，表示在滚动时间窗中，在请求数量超过                // circuitBreaker.requestVolumeThreshold 的情况下，如果错误请求数的百分比超过50,                // 就把断路器设置为 "打开" 状态，否则就设置为 "关闭" 状态。
		@HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),                // 该属性用来设置当断路器打开之后的休眠时间窗。 休眠时间窗结束之后，                // 会将断路器置为 "半开" 状态，尝试熔断的请求命令，如果依然失败就将断路器继续设置为 "打开" 状态，                // 如果成功就设置为 "关闭" 状态。   
		@HystrixProperty(name = "circuitBreaker.sleepWindowinMilliseconds", value = "5000"),                // 断路器强制打开     
	    @HystrixProperty(name = "circuitBreaker.forceOpen", value = "false"),                // 断路器强制关闭          
		@HystrixProperty(name = "circuitBreaker.forceClosed", value = "false"),                // 滚动时间窗设置，该时间用于断路器判断健康度时需要收集信息的持续时间               
		@HystrixProperty(name = "metrics.rollingStats.timeinMilliseconds", value = "10000"),                // 该属性用来设置滚动时间窗统计指标信息时划分"桶"的数量，断路器在收集指标信息的时候会根据                // 设置的时间窗长度拆分成多个 "桶" 来累计各度量值，每个"桶"记录了一段时间内的采集指标。                // 比如 10 秒内拆分成 10 个"桶"收集这样，所以 timeinMilliseconds 必须能被 numBuckets 整除。否则会抛异常    
		@HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),                // 该属性用来设置对命令执行的延迟是否使用百分位数来跟踪和计算。如果设置为 false, 那么所有的概要统计都将返回 -1。  
		@HystrixProperty(name = "metrics.rollingPercentile.enabled", value = "false"),                // 该属性用来设置百分位统计的滚动窗口的持续时间，单位为毫秒。                @HystrixProperty(name = "metrics.rollingPercentile.timeInMilliseconds", value = "60000"),                // 该属性用来设置百分位统计滚动窗口中使用 “ 桶 ”的数量。 
		@HystrixProperty(name = "metrics.rollingPercentile.numBuckets", value = "60000"),                // 该属性用来设置在执行过程中每个 “桶” 中保留的最大执行次数。如果在滚动时间窗内发生超过该设定值的执行次数，                // 就从最初的位置开始重写。例如，将该值设置为100, 滚动窗口为10秒，若在10秒内一个 “桶 ”中发生了500次执行，                // 那么该 “桶” 中只保留 最后的100次执行的统计。另外，增加该值的大小将会增加内存量的消耗，并增加排序百分位数所需的计算时间。
		@HystrixProperty(name = "metrics.rollingPercentile.bucketSize", value = "100"),                // 该属性用来设置采集影响断路器状态的健康快照（请求的成功、 错误百分比）的间隔等待时间。                
		@HystrixProperty(name = "metrics.healthSnapshot.intervalinMilliseconds", value = "500"),                // 是否开启请求缓存   
		@HystrixProperty(name = "requestCache.enabled", value = "true"),                // HystrixCommand的执行和事件是否打印日志到 HystrixRequestLog 中    
		@HystrixProperty(name = "requestLog.enabled", value = "true"),        },        threadPoolProperties = {                // 该参数用来设置执行命令线程池的核心线程数，该值也就是命令执行的最大并发量         
		@HystrixProperty(name = "coreSize", value = "10"),                // 该参数用来设置线程池的最大队列大小。当设置为 -1 时，线程池将使用 SynchronousQueue 实现的队列，                // 否则将使用 LinkedBlockingQueue 实现的队列。 
		@HystrixProperty(name = "maxQueueSize", value = "-1"),                // 该参数用来为队列设置拒绝阈值。 通过该参数， 即使队列没有达到最大值也能拒绝请求。                // 该参数主要是对 LinkedBlockingQueue 队列的补充,因为 LinkedBlockingQueue                // 队列不能动态修改它的对象大小，而通过该属性就可以调整拒绝请求的队列大小了。               
		@HystrixProperty(name = "queueSizeRejectionThreshold", value = "5"),        })
public String strConsumer() {    
	return "hello 2020";
}
public String str_fallbackMethod(){    
	return "*****fall back str_fallbackMethod";
}
```

### 雪崩效应，你了解吗？

在微服务架构中，一个请求需要调用多个服务是非常常见的。如客户端访问A服务，而A服务需要调用B服务，B服务需要调用C服务，由于网络原因或者自身的原因，如果B服务或者C服务不能及时响应，A服务将处于阻塞状态，直到B服务C服务响应。此时<font color='red'>若有大量的请求涌入，容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，造成连锁反应，会对整个微服务系统造成灾难性的严重后果</font>，这就是服务故障的“雪崩”效应。

### 造成雪崩效应的原因：

- 单个服务的代码存在bug
- 请求访问量激增导致服务发生崩溃（如大型商城的枪红包，秒杀功能）
- 服务器的硬件故障也会导致部分服务不可用

### 服务降级，你了解吗？

所谓降级，就是当某个服务熔断之后，服务器将不再被调用，此时客户端可以自己准备一个本地的fallback回调，<font color='red'>返回一个缺省值</font>。也可以理解为<font color='red'>兜底方法</font>。

会发生降级的情况：

- 程序运行异常
- 超时
- 服务熔断触发服务降级
- 线程池、信号量打满也会导致服务降级

### 服务熔断，你了解吗？

在复杂的分布式系统中，微服务之间的相互调用，有可能出现各种各样的原因导致服务的阻塞，在高并发场景下，服务的阻塞意味着线程的阻塞，导致当前线程不可用，服务器的线程全部阻塞，导致服务器崩溃，由于服务之间的调用关系是同步的，会对整个微服务系统造成服务雪崩。

为了解决某个微服务的调用响应时间过长或者不可用进而占用越来越多的系统资源引起雪崩效应就需要进行服务熔断和服务降级处理。

所谓的服务熔断指的是某个服务故障或异常一起类似显示世界中的“保险丝"当某个异常条件被触发就直接熔断整个服务，而不是一直等到此服务超时。

<font color='red'>服务熔断就是相当于我们电闸的保险丝，一旦发生服务雪崩的，就会熔断整个服务，通过维护一个自己的线程池，当线程达到阈值的时候就启动服务降级，如果其他请求继续访问就直接返回fallback的默认值</font>。

### 服务限流，你了解吗？

<font color='red'>限流可以认为是服务降级的一种，限流就是限制系统的输入和输出流量以达到保护系统的目的</font>。一般来说系统的吞吐量是可以被测算的，为了保证系统的稳固运行，一旦达到的需要限制的阈值，就需要限制流量并采取少量措施以完成限制流量的目的。比方：推迟解决，拒绝解决，或者部分拒绝解决等等。秒杀高并发等操作，严禁一窝蜂的一样拥挤，排队有序进行，一秒钟N个，有序进行。

## <font color='#B96AD9'>Zuul</font>

### 什么是Spring Cloud Zuul？

Zuul是SpringCloud提供的成熟的路由方案，根据请求的路径不同，网关会定位到指定的微服务，并代理请求到不同的微服务接口，它对外隐蔽了微服务的真正接口地址。三个重要概念：动态路由表，路由定位，反向代理

- <font color='red'>动态路由表</font>：Zuul支持Eureka路由，手动配置路由，这两种都支持自动更新
- <font color='red'>路由定位</font>：根据请求路径，Zuul有自己的一套定位服务规则以及路由表达式匹配
- 反向代理：客户端请求到路由网关，网关受理之后，在对目标发送请求，拿到响应之后再给客户端

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175201407-2047595084.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175201407-2047595084.png)

### Zuul的应用场景有哪些？

对外暴露，**权限校验**，**服务聚合**，**日志审计**。

### 网关与过滤器有什么区别？

网关是对所有服务的请求进行分析过滤，过滤器是对单个服务而言。

### Zuul与Nginx有什么区别？

Zuul是java语言实现的，主要为java服务提供网关服务，尤其在微服务架构中可以更加灵活的对网关进行操作。

Nginx是使用C语言实现，性能高于Zuul，但是实现自定义操作需要熟悉lua语言，对程序员要求较高，可以使用Nginx做Zuul集群。

### ZuulFilter常用有那些方法？

- <font color='red'>run()</font>：过滤器的具体业务逻辑
- <font color='red'>shouldFilter()</font>：判断过滤器是否有效
- <font color='red'>fifilterOrder()</font>：过滤器执行顺序
- <font color='red'>fifilterType()</font>：过滤器拦截位置

## <font color='#B96AD9'>Gateway</font>

### 说说你对Spring Cloud Gateway的理解

Spring Cloud Gateway是<font color='red'>Spring Cloud官方推出的第二代网关框架，取代Zuul网关</font>。网关作为流量的，在微服务系统中有着非常作用，<font color='red'>网关常见的功能有路由转发、权限校验、限流控制等作用</font>。

使用了一个RouteLocatorBuilder的bean去创建路由，除了创建路由RouteLocatorBuilder可以让你添加各种predicates和filters，predicates断言的意思，顾名思义就是根据具体的请求的规则，由具体的route去处理，filters是各种过滤器，用来对请求做各种判断和修改。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175218558-1497381244.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175218558-1497381244.png)

### 为什么我们选择GateWay？

#### 1）因为Zuul1.0已经进入了维护阶段，而且<font color='red'>Gateway是SpringCloud团队研发的，值得信赖</font>

Gateway是基于异步非阻塞模型上进行开发的，性能方面不需要担心。虽然Netflix早就发布了最新的 Zuul 2.x，但 Spring Cloud 貌似没有整合计划。而且Netflix相关组件都宣布进入维护期，不知道前景如何。

#### 2）Spring Cloud GateWay有很多特性

- 基于Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建；
- 动态路由：能够匹配任何请求属性；
- 可以对路由指定 Predicate（断言）和 Filter（过滤器）；
- 集成Hystrix的断路器功能；
- 集成 Spring Cloud 服务发现功能；
- 易于编写的 Predicate（断言）和 Filter（过滤器）；
- 请求限流功能；
- 支持路径重写。

### Spring Cloud Gateway 与 Zuul的区别？

1）Zuul 1.x是一个基于阻塞 I/O 的 API Gateway

2）Zuul 1.x 基于Servlet 2.5使用阻塞架构，它不支持任何长连接(如 WebSocket)。Zuul的设计模式和Nginx较像，每次 I/O 操作都是从工作线程中选择一个执行，请求线程被阻塞到工作线程完成，但是差别是Nginx 用C++ 实现，Zuul 用 Java 实现，而 JVM 本身会有第一次加载较慢的情况，使得Zuul 的性能相对较差。

3）Zuul 2.x理念更先进，是基于Netty非阻塞和支持长连接的，但SpringCloud目前还没有整合。Zuul 2.x的性能较Zuul 1.x有较大提升。

4）Spring Cloud Gateway 建立在Spring Framework 5、Project Reactor和Spring Boot 2之上，使用非阻塞API。

5）Spring Cloud Gateway 还支持 WebSocket， 并且与Spring紧密集成拥有更好的开发体验。

Spring Cloud GateWay工作流程？

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404164850089-549813837.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404164850089-549813837.png)

客户端向 Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。

Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。

过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑。

Filter在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等。

核心逻辑：**路由转发+执行过滤链**

## <font color='#B96AD9'>Config</font>

### 了解Spring Cloud Config 吗?

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在Spring Cloud中，有分布式配置中心组件Spring Cloud Config，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库中。

在Spring Cloud Config 组件中，分两个角色，一是config server，二是config client。

使用方式：

- 添加pom依赖
- 配置文件添加相关配置
- 启动类添加注解<font color='red'>@EnableConfigServer</font>

### Spring Cloud Config修改配置文件，如何动态刷新？

配合Spring Cloud Bus实现配置的动态的刷新。

## <font color='#B96AD9'>Bus</font>

### 熟悉 Spring Cloud Bus 吗?

spring cloud bus 将分布式的节点用轻量的消息代理连接起来，它可以用于广播配置文件的更改或者服务直接的通讯，也可用于监控。如果修改了配置文件，发送一次请求，所有的客户端便会重新读取配置文件。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175237741-2006940299.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175237741-2006940299.png)

### Spring Cloud Bus如何动态刷新全局广播？

- 利用消息总线触发一个客户端/bus/refresh，从而刷新所有客户端配置
- 利用消息总线触发一个服务端ConfigServer的/bus/refresh端点，而刷新所有客户端的配置

第二个思想更合适一点，第一个不适合原因如下：

- 打破了微服务的单一职责性，因为微服务本身是业务模块，它本不应该承担配置刷新的职责
- 破坏了微服务各节点的对等性
- 有一定的局限性。比如微服务在迁移时，它的网络地址时常发生变化，这时要想做到自动刷新，那就会增加更多的修改

## <font color='#B96AD9'>Stream</font>

### Spring Cloud Stream消息驱动是什么？

<font color='red'>屏蔽底层消息中间件的差异，降低切换成本，统一消息的编程模型</font>；轻量级事件驱动微服务框架，可以使用简单的声明式模型来发送及接收消息，主要实现为Apache Kafka及RabbitMQ。

### 为什么Spring Cloud Stream可以统一底层差异？

在没有绑定器这个概念的情况下，我们的SpringBoot应用要直接与消息中间件进行信息交互的时候，由于各消息中间件构建的初衷不同，它们的实现细节上会有较大的差异性通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统一的Channel通道，使得应用程序不需要再考虑各种不同的消息中间件实现。

### Spring Cloud Stream如何解决重复消费和持久化？

加入分组属性Group，微服务应用放置于同一个Group中，就能够保证消息只会被其中一个应用消费一次。不同的组是可以消费的，同一个组内会发生竞争关系，只有其中一个可以消费。

## <font color='#B96AD9'>Sleuth</font>

### 什么是Spring Cloud Sleuth？

Spring Cloud Sleuth提供了<font color='red'>一套完整的服务跟踪的解决方案</font>，在分布式系统中提供追踪解决方案并且兼容支持了zipkin。

[![img](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175258169-545846520.png)](https://img2023.cnblogs.com/blog/2443180/202304/2443180-20230404175258169-545846520.png)



参考：[Spring Cloud最全面试题整理，全是干货](https://mp.weixin.qq.com/s?__biz=MzI0NDA5MjczNA==&mid=2247485168&idx=1&sn=3fe27eb5f0b7939ee073790ac3d13e9c&chksm=e9625ca1de15d5b79f9d9520cdc72256a771b0073c44087eac600904e88f7fabf3057f0fe1d2&mpshare=1&scene=23&srcid=0403RBFQZVVruuZWqusjyLcX&sharer_sharetime=1680530495897&sharer_shareid=5b6ee1f9492304ef222851839dbed57b#rd)[面试必备！SpringCloud体系重点知识](https://mp.weixin.qq.com/s?__biz=MzU2MDc5NjEzMA==&mid=2247484355&idx=1&sn=4f38355d64b326b503ac031d207b3e6c&chksm=fc03db4ecb745258706c574420b42485ee007a652ea0e2024505d33d02ce5e4db1abd457c5ee&mpshare=1&scene=23&srcid=0403cdSsPrqhhMyuqGftZNdT&sharer_sharetime=1680530687942&sharer_shareid=5b6ee1f9492304ef222851839dbed57b#rd)

 
