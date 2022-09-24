[toc]

# <font color=' #FF6347'>Spring事务管理（一）- 配置方式</font>

​		当项目的数据需要持久化存储时，不可避免的要和数据进行交互。在交互的过程中，对象的支出则是尤为重要的。

​		JDBC规范了对事物的操作。在[深入浅出JDBC（一）- Connection与事务介绍]()一章中简要的介绍了JDBC事务相关的概念。JDBC将对不同数据的交互规范化，包括事务的操作，让开发者可以屏蔽不同数据库的差异使用接口编程。但事物的开启或关闭，以及事务的控制和配置还是需要手动编码控制，未免繁琐且容易出错。Spring基于此之上，开放出一套事务管理机制，将开发者从繁琐的事务控制中解脱出来，可以便捷的执行事务的控制。然而作为开发者，操作便捷，但原理更为重要，只有了解了原理，才能更好的把控程序。

​		那么接下来，我将从Spring事务管理的配置到原理，逐步介绍其运行机制。本篇先介绍三种从原始到简化的配置方式。

## :cactus: 以mybatis+mysql为基础，基本的xml配置方式

```xml
<!-- 数据源 -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
	<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
	<property name="url" value="jdbc:mysql://192.168.1.160:3306/zero2one"></property>
	<property name="username" value="root"></property>
	<property name="password" value="root"></property>
	<property name="connectionProperties" value="useUnicode=true;autoReconnect=true;failOverReadOnly=false;characterEncoding=utf8;zeroDateTimeBehavior=convertToNull;allowMultiQueries=true"></property>
</bean>

<!-- mybatis的Session工厂 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="configLocation" value="mybatis-config.xml"></property>
	<property name="dataSource" ref="dataSource"></property>
</bean>

<!-- UserMapper代理 -->
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
	<property name="mapperInterface" value="com.zero2one.spring.transaction.mapper.UserMapper"/>
	<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
</bean>

<!-- userManager实例 -->
<bean id="userManager" class="com.zero2one.spring.transaction.manager.UserManager">
	<property name="userMapper" ref="userMapper"></property>
</bean>
```

这里对mybatis的配置就不过多的介绍了，事务定义在UserManager层，UserManager中定义一个批量操作的额方法，来验证事务。

```java
@Slf4j
public class UserManager {
	@Getter
	@Setter
	private UserMapper userMapper;

	public void batchOperator(){
		User user = new User("lily", 25);
		// 插入一条user记录
		int insertRows = userMapper.insert(user);
		if(insertRows > 0){
			user = userMapper.getUser(user.getId());
			log.info(user.toString());
			user.setName("jack");
			user.setAge(28);
			// 更新user记录
			userMapper.update(user);
		}
	}
}
```



## :seedling: 1. TransactionProxyFactoryBean代理

> #### 使用TransactionProxyFactoryBean直接代理UserManager

```xml
<!-- userManager事务代理类 -->
<bean id="userManagerTransactionProxy" 
	class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
	<!-- 原始对象 -->
	<property name="target" ref="userManager"/>
	<!-- 事务属性 -->
	<property name="transactionAttributes">
		<props>
			<prop key="batchOperator">PROPAGATION_REQUIRED</prop>
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



TransactionProxyFactoryBean使用ProxyFactory生成代理类完成对事物的处理。transactionAttributes接收一个properties文件，key为方法名，value为方法对应的事物配置，配置规则的就解析类是TransactionAttributeEditor，配置如下：

| 配置名称                      | 前缀         | 配置选项                                                     | 默认值            |
| ----------------------------- | ------------ | ------------------------------------------------------------ | ----------------- |
| 传播属性                      | PROPAGATION_ | REQUIRE \| MANDATORY \| REQUIRE_NEW \| NESTED                | REQUIRED          |
| 隔离级别                      | ISOLATION_   | READ_UNCOMMITTED  \| READ_COMMITTED REPEATABLE_READ  \| SERIALIZABLE | DEFAULT           |
| 超时时间                      | timeout_     | 单位：秒                                                     | -1                |
| 只读                          | readOnly     | 不配置表示可读写，配置了即可阅读。                           | 可读可写          |
| 不回滚异常规则(noRollbackFor) | +            | 异常类包括路径                                               | 无                |
| 回滚异常规则(rollbackFor)     | -            | 异常类包路径                                                 | 运行时异常及Error |

Spring事务管理器则是PlatformTransactionManager的子类，实现其获取事务，提交事务和回滚事务三个方法，这里我结合自己的情况，使用JDBC的事务管理器：DataSourceTransactionManager。

 写一个测试类：

```java
public class TransactionProxyFactoryBeanTest {
	public static void main(String[] args) {
		ApplicationContext context = new ClassPathXmlApplicationContext("TransactionProxyFactoryBean.xml");
		UserManager userManager = context.getBean("userManagerTransactionProxy", UserManager.class);
		userManager.batchOperator();
	}
}
```

通过上下文获取userManager的代理对象，并执行操作方法，可以在日志中看到事物的运行。

## :palm_tree: 2. XML配置

​		上面的配置方式只能对一个类代理，用来研究Spring事务管理的机制很好，但在项目中，需要批量地配置事务管理。使用Spring AOP中的Advice和Pointcut的方式就能完成，而且Spring定义了[tx:advice](tx:advice)标签来支持配置。

```xml
<!-- 事务管理器 -->
<bean id="transactionManager"
	class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"></property>
</bean>

<!-- 事务Advice增强配置 -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
	<tx:attributes>
		<tx:method name="get*" read-only="true" />
		<tx:method name="*" propagation="REQUIRED"/>
	</tx:attributes>
</tx:advice>

<!-- 配置事务aop -->
<aop:config>
	<aop:pointcut id="userPointcut"
		expression="execution(* com.lcifn.spring.transaction.manager.UserManager.*(..))" />
	<aop:advisor advice-ref="txAdvice" pointcut-ref="userPointcut" />
</aop:config>
```

在tx:advice中，需要配置事务管理器，在其子标签中tx:attributes中配置方法和事务配置的关系。方法配置支持通配符匹配多个方法，事务的配置同上面的基本一致，只是Spring提供了属性标签来方便配置。配置aop:pointcut切入点，决定哪些类哪些方法需要被代理。

写一个测试类，这里直接获取userManager，因为aop:config自动代理的原因，返回的已经是代理对象了。

```java
public class TransactionTxAdvice {
	public static void main(String[] args) {
		ApplicationContext context = new ClassPathXmlApplicationContext("transaction-tx-advice.xml");
		UserManager userManager = context.getBean("userManager", UserManager.class);
		userManager.batchOperator();
	}
}
```



## :watermelon: 3. 声明式事务配置[@Transactional]()

​		XML配置的方式已经能大大地节约配置操作，但如果要对每个方法细粒度地配置，xml也会变得很繁琐。怎么办？这时我们想到了注解。Spring提供了超级便捷的方式，通过注解在类或方法上完成事务的控制和配置。

```xml
<!-- 事务注解驱动 -->
<tx:annotation-driven transaction-manager="transactionManager"/>

<!-- 事务管理器 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"></property>
</bean>
```

​		xml中的就是上面简单的配置，而在类或方法上使用@Transactional注解，比如上面的batchOperator方法可以配置如下：

```java
// 事务传播属性为必须，超时时间为2秒
@Transactional(propagation = Propagation.REQUIRED, timeout = 2)
```



#### 【注】每个方法是否开启事务，以及事务配置什么属性都可以通过这种方式精细化地控制。

## :shamrock: 4. 事务隔离级别和传播方式

​		Spring的事务隔离级别支持sql标准定义的四个级别：**Read Uncommitted**(未授权读取)，**Read Committed**(授权读取)，**Repeatable Read**(可重复读取)，**Serializable**(序列化)。可在[深入浅出JDBC(一) - Connection与事务介绍]()中了解详细。

​		Spring定义了事务传播方式，指的是多个层次(非同一个类)的事务方法执行时的规则，即事务在方法中的传播方式。Spring定义了七种传播行为(包括**事务的新建和回滚行为**)：

REQUIRED：默认传播方式。如果当前有事务，支持当前事务；如果没有事务，创建一个事务。回滚时全部回滚。

- REQUIRES_NEW：如果当前有事务，将当前事务挂起，创建新的事务；如果没有事务，也创建新的事务。回滚时只回滚当前事务，不影响其他事务。
- NESTED：嵌套事务。如果当前有事务，设置savepoint保存点；如果没有事务，创建一个事务。回滚时只回滚到保存点。
- SUPPORTS：如果当前有事务，支持当前事务；如果没有事务，就以非事务方式执行。回滚时全部回滚。
- MANDATORY：强制事务。如果当前有事务，支持当前事务；如果没有事务，则抛出异常。回滚时全部回滚。
- NOT_SUPPORTED：如果当前有事务，将当前事务挂起，以非事务方式执行。如果没有事务，以非事务方式执行。不存在回滚。
- NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。不存在回滚。



对于事务的创建，有一点需要着重强调。**JDBC默认连接的提交方式为自动，如果开启事务，即将自动提交改为手动。因此开启一个新的事务，即是获取一个新的连接**， ，具体实现也会在之后的源码解析里提示。下一章我们来解析TransactionProxyFactoryBean的实现原理。