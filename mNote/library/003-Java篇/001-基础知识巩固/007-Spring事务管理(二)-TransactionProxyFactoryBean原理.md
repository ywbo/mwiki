[toc]

# <font color='#FF6347'>Spring事务管理(二)-TransactionProxyFactoryBean原理</font>

> 通常Spring事务管理的配置都是XML或者生命是注解的方式，然后想要学习其运行原理，从TransactionProxyFactoryBean深入更加合适。窥斑见豹，逐步介绍Spring事物的运行机制。

## 一、 Spring事务的核心类

Spring事务的构成，基本有三部分，**事务属性的定义**，**事务对象及状态信息的持有**，**事务管理器的处理**。

### 1. 事务属性的定义

- TransactionDefinition 事务定义（传播属性，隔离属性，timeout等）
- TransactionAttribute 事务属性接口（继承TransactionDefinition ），常用的实现是 RuleBaseTransactionAttribute
- TransactionAttributeSource 事务属性数据源，可以根据 method 和 Class 获取 TransactionAttribute（注解方式实现 AnnotationTransactionAttribute，编程方式的实现为：NameMatchTransactionAttributeSource）

### 2. 事务对象及状态信息的持有

- Transaction 事务对象，由具体的事务管理器实现返回
- TransactionStatus 事务状态，持有事务对象以及事物本身的属相和状态，每次事务方法执行前，生成一个新的 TransactionStatus，但如果不需要创建新事物，则持有的事务和上层一致
- TransactionInfo 事务信息，持有事务属性，事务状态，事务连接点（方法路径），同时持有上一事务信息对象、形成链式结构

### 3. 事务管理器的处理

- PlatformTransactionManager 事务管理接口，AbstractPlatformTransactionManager为其抽象实现（实现了 getTransaction， commit， rollback 模板方法），常用的实现，如：JDBC的事务管理实现 DataSourceTransactionManager

## 二、 TransactionProxyFactoryBean

TransactionProxyFactoryBean 实现了 FactoryBean 和 InitializingBean接口，FactoryBean  支持对需要事物的类代理，InitializingBean 初始化事务环境的准备工作，完成代理对象的创建。我们再来看下xml的配置“

```xml
<!-- userManager事务代理类 -->
<bean id="userManagerTransactionProxy"
   class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
   <!-- 原始对象 -->
   <property name="target" ref="userManager"/>
   <!-- 事务属性 -->
   <property name="transactionAttributes">
      <props>
         <prop key="batchOperator">PROPAGATION_REQUIRED,readOnly</prop>
      </props>
   </property>
   <!-- 事务管理器 -->
   <property name="transactionManager">
      <bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
         <property name="dataSource" ref="dataSource"/>
      </bean>
   </property>
</bean>
```

需要配置有三个，**原始对象，事务属性和事务管理器**。其中原始对象用来生成代理对象，而事务属性和事务管理器都设置到一个很重要的对象中，即TransactionInterceptor，事务拦截器，对应AOP中的增强类，完成事务管理和方法本身的结合。

回到TransactionProxyFactoryBean的构造，其主要实现定义在抽象父类AbstractSingletonProxyFactoryBean中。在初始化方法afterPropertiesSet中，使用最原始的ProxyFactory， 完成了代理对象的创建。

```java
AbstractSingletonProxyFactoryBean.java

public void afterPropertiesSet() {
   if (this.target == null) {
      throw new IllegalArgumentException("Property 'target' is required");
   }
   if (this.target instanceof String) {
      throw new IllegalArgumentException("'target' needs to be a bean reference, not a bean name as value");
   }
   if (this.proxyClassLoader == null) {
      this.proxyClassLoader = ClassUtils.getDefaultClassLoader();
   }

   // 创建AOP代理工厂
   ProxyFactory proxyFactory = new ProxyFactory();

   // 添加前置增强拦截器
   if (this.preInterceptors != null) {
      for (Object interceptor : this.preInterceptors) {
         proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
      }
   }

   // 核心增强拦截器，有子类实现，在TransactionProxyFactoryBean中即是TransactionInterceptor
   // Add the main interceptor (typically an Advisor).
   proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(createMainInterceptor()));

   // 添加后置增强拦截器
   if (this.postInterceptors != null) {
      for (Object interceptor : this.postInterceptors) {
         proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
      }
   }

   // 拷贝AOP的基本配置，如exposeProxy等
   proxyFactory.copyFrom(this);

   // 设置原始对象
   TargetSource targetSource = createTargetSource(this.target);
   proxyFactory.setTargetSource(targetSource);

   // 设置原始对象实现的接口
   if (this.proxyInterfaces != null) {
      proxyFactory.setInterfaces(this.proxyInterfaces);
   }
   else if (!isProxyTargetClass()) {
      // Rely on AOP infrastructure to tell us what interfaces to proxy.
      Class<?> targetClass = targetSource.getTargetClass();
      if (targetClass != null) {
         proxyFactory.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
      }
   }

   // 对proxyFactory的后置处理
   postProcessProxyFactory(proxyFactory);

   // 代理工厂生成代理对象
   this.proxy = proxyFactory.getProxy(this.proxyClassLoader);
}
其中核心增强拦截器有子类实现其创建方法createMainInterceptor

TransactionProxyFactoryBean.java

protected Object createMainInterceptor() {
   // 初始化TransactionInterceptor
   this.transactionInterceptor.afterPropertiesSet();
   if (this.pointcut != null) {
   	  // 有pointcut的生成DefaultPointcutAdvisor
      return new DefaultPointcutAdvisor(this.pointcut, this.transactionInterceptor);
   }
   else {
      // Rely on default pointcut.
      // 没有pointcut的依赖默认的pointcut
      return new TransactionAttributeSourceAdvisor(this.transactionInterceptor);
   }
}
TransactionAttributeSourceAdvisor的默认pointcut类是TransactionAttributeSourcePointcut，定义在其内部。

TransactionAttributeSourceAdvisor.java

private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
   @Override
   @Nullable
   protected TransactionAttributeSource getTransactionAttributeSource() {
      return (transactionInterceptor != null ? transactionInterceptor.getTransactionAttributeSource() : null);
   }
};
```

TransactionAttributeSourcePointcut是一个抽象类，在TransactionAttributeSourceAdvisor匿名实现的。我们来看下TransactionAttributeSourcePointcut的实现，它继承了StaticMethodMatcherPointcut，一个对方法进行匹配的基本Pointcut类，实现matches方法。

```java
public boolean matches(Method method, @Nullable Class<?> targetClass) {
   if (targetClass != null && TransactionalProxy.class.isAssignableFrom(targetClass)) {
      return false;
   }
   // 获取事务属性数据源
   TransactionAttributeSource tas = getTransactionAttributeSource();
   // 根据方法和Class对象获取是否有事务属性配置存在，来决定是否切入事务AOP
   return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```

而事务属性数据源从哪里设置的呢？我们在上面的TransactionAttributeSourceAdvisor匿名实现的TransactionAttributeSourcePointcut类中可以发现TransactionAttributeSource是从TransactionInterceptor中获取的。而TransactionInterceptor的TransactionAttributeSource是哪里设置的呢？来源于XML配置的properties对象transactionAttributes，在TransactionProxyFactoryBean的setTransactionAttributes方法中。

```java
public void setTransactionAttributes(Properties transactionAttributes) {
   this.transactionInterceptor.setTransactionAttributes(transactionAttributes);
}
```

transactionAttributes实际被设置到TransactionInterceptor中

```java
public void setTransactionAttributes(Properties transactionAttributes) {
   NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
   tas.setProperties(transactionAttributes);
   this.transactionAttributeSource = tas;
}
```

这里看到TransactionAttributeSource的实现是NameMatchTransactionAttributeSource。在其内部维护了一个方法名和事务属性的Map

```java
private Map<String, TransactionAttribute> nameMap = new HashMap<>();
```

因此对哪个方法进行事务AOP的切入的原理就很清楚了。接下来就是核心拦截器TransactionInterceptor的解析。

## 三、 TransactionInterceptor

TransactionInterceptor实现了Spring AOP的基本增加接口MethodInterceptor，实现invoke方法

```java
public Object invoke(final MethodInvocation invocation) throws Throwable {
   // Work out the target class: may be {@code null}.
   // The TransactionAttributeSource should be passed the target class
   // as well as the method, which may be from an interface.
   Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

   // Adapt to TransactionAspectSupport's invokeWithinTransaction...
   return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}
```

具体实现由TransactionInterceptor抽象子类TransactionAspectSupport的invokeWithinTransaction方法执行

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
      final InvocationCallback invocation) throws Throwable {

   // If the transaction attribute is null, the method is non-transactional.
   // 获取事务属性数据源，如果为null，则此方法为非事务环境
   TransactionAttributeSource tas = getTransactionAttributeSource();
   // 获取事务属性
   final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
   // 根据事务属性决定事务管理器对象
   final PlatformTransactionManager tm = determineTransactionManager(txAttr);
   // 连接点标识，一般就是方法名
   final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

   if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
      // Standard transaction demarcation with getTransaction and commit/rollback calls.
      // 创建事务对象，并返回事务信息
      TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
      Object retVal = null;
      // 环绕增强实现，try中为方法本身的执行
      try {
         // This is an around advice: Invoke the next interceptor in the chain.
         // This will normally result in a target object being invoked.
         retVal = invocation.proceedWithInvocation();
      }
      catch (Throwable ex) {
         // target invocation exception
         // 异常后的事务处理
         completeTransactionAfterThrowing(txInfo, ex);
         throw ex;
      }
      finally {
      	 // 解绑事务信息和当前线程
         cleanupTransactionInfo(txInfo);
      }
      // 方法执行成功后事务提交的处理
      commitTransactionAfterReturning(txInfo);
      return retVal;
   }
   // ...
```

这个方法就是事务执行的核心部分，通过环绕增加，完成方法不同执行结果(成功或异常)对应事务的处理。当然前提是要先创建事务，来看createTransactionIfNecessary方法

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
      @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

   // If no name specified, apply method identification as transaction name.
   if (txAttr != null && txAttr.getName() == null) {
      txAttr = new DelegatingTransactionAttribute(txAttr) {
         @Override
         public String getName() {
            return joinpointIdentification;
         }
      };
   }

   TransactionStatus status = null;
   if (txAttr != null) {
      if (tm != null) {
         // 由事务管理器创建事务状态对象
         status = tm.getTransaction(txAttr);
      }
      else {
         if (logger.isDebugEnabled()) {
            logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                  "] because no transaction manager has been configured");
         }
      }
   }
   // 填充事务状态对象的其他信息
   return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

事务对象及状态的维护由具体的事务管理器来管理，下一章我们具体讨论事务管理器，这里主要关注它的构建骨架。返回的事务状态对象TransactionStatus，需要再次被prepareTransactionInfo方法填充

```java
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
      @Nullable TransactionAttribute txAttr, String joinpointIdentification,
      @Nullable TransactionStatus status) {

   // 创建事务信息对象，记录事务管理器，事务属性，及事务切入点信息
   TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
   if (txAttr != null) {
      // 记录事务状态
      txInfo.newTransactionStatus(status);
   }
   else {
      if (logger.isTraceEnabled())
         logger.trace("Don't need to create transaction for [" + joinpointIdentification +
               "]: This method isn't transactional.");
   }

   // We always bind the TransactionInfo to the thread, even if we didn't create
   // a new transaction here. This guarantees that the TransactionInfo stack
   // will be managed correctly even if no transaction was created by this aspect.
   // 绑定事务信息对象和当前线程，并在当前事务信息对象中记录线程原来绑定的事务信息对象，从而保证了事务信息栈的管理
   txInfo.bindToThread();
   return txInfo;
}
```

这里非常重要的是维护了一个线程级别的事务信息的栈结构。

事务信息组建完成后，就是方法本身的执行。如果发生异常，则事务该如何处理，由completeTransactionAfterThrowing方法执行

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
      // 校验当前异常是否要回滚事务，如果是，则执行回滚操作，如果不是，则执行提交操作
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
         catch (Error err) {
            logger.error("Application exception overridden by rollback error", ex);
            throw err;
         }
      }
      else {
         // We don't roll back on this exception.
         // Will still roll back if TransactionStatus.isRollbackOnly() is true.
         try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
         catch (Error err) {
            logger.error("Application exception overridden by commit error", ex);
            throw err;
         }
      }
   }
}
```

在上面的方法中，核心就是判断原始方法抛出的异常是否要回滚事务，如果是，则调用事务管理器回滚事务，如果不是，则直接提交事务。

对异常是否回滚事务的判断是由事务属性中的rollbackFor和noRollbackFor共

同决定的，默认都没有配置时，执行DefaultTransactionAttribute基本事务属性类中的rollbackOn方法

```java
public boolean rollbackOn(Throwable ex) {
   return (ex instanceof RuntimeException || ex instanceof Error);
}
```

即当异常为运行时异常或Error时，就会回滚，否则直接提交。直接提交这一点也是需要格外注意的。

如果方法运行没有异常，执行完成后，就需要提交事务，由commitTransactionAfterReturning方法执行

```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
      }
      txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
   }
}
```

调用事务管理器执行commit来提交事务。

至此，对于TransactionProxyFactoryBean的AOP代理生成以及TransactionInterceptor核心增强中事务执行的原理都基本解析清楚了，下一章介绍事务管理器的运行机制以及DataSourceTransactionManager如何和JDBC事务接口的交互。
