# Mybatis源码解析(四)-整合Spring

在实际使用时，一般将Mybatis整合到Spring中，也是将Mapper接口的生命周期交给Spring来管理。

## 1.单个Mapper的配置

```xml
	<!-- 数据源 -->
	<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
		<property name="url" value="jdbc:mysql://ip:3306/user"></property>
		<property name="username" value="admin"></property>
		<property name="password" value="admin"></property>
	</bean>
	
	<!-- mybatis的Session工厂 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="configLocation" value="mybatis-config.xml"></property>
		<property name="dataSource" ref="dataSource"></property>
	</bean>
	
	<!-- UserMapper代理 -->
	<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
		<property name="mapperInterface" value="com.lcifn.orm.mybatis.mapper.UserMapper"/>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
	</bean>
```

这里的配置有三个部分：

1. DataSouce的bean配置：jdbc的配置
2. SqlSessionFactory的bean配置：使用工厂类SqlSessionFactoryBean构建
3. Mapper接口的bean配置：使用工厂类MapperFactoryBean构建

### 1.1 SqlSessionFactoryBean构建SqlSessionFactory

SqlSessionFactoryBean实现了Spring的FactoryBean接口，getObject方法判断SqlSessionFactory对象是不是存在，不存在则执行afterPropertiesSet方法，实现调用的是buildSqlSessionFactory。

```java
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }

    return this.sqlSessionFactory;
  }
  
 public void afterPropertiesSet() throws Exception {
    this.sqlSessionFactory = buildSqlSessionFactory();
 }
```

buildSqlSessionFactory方法，沿用Mybatis的XMLConfigBuilder处理mybatis配置文件，并做解析的方式。只是扩展了很多属性可以在Spring配置。

```java
xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
xmlConfigBuilder.parse();
```

详细的解析可以参考[Mybatis源码解析(二)-配置解析](https://my.oschina.net/u/2377110/blog/4283388)。

### 1.2 MapperFactoryBean创建Mapper接口代理对象

MapperFactoryBean.getObject方法执行两步，一是获取SqlSession，二是获取Mapper代理对象。

```java
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

此时获取的SqlSession也是动态代理对象，因为MapperFactoryBean设置sqlSessionFactory属性时，执行了SqlSession的创建。

```java
  SqlSessionDaoSupport.java
  
  public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```

创建的SqlSessionTemplate对象，是被Spring管理的SqlSession封装对象，内部的SqlSession对象是jdk的动态代理类Proxy生成的，实际执行的是SqlSessionInterceptor。

```java
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {
      ...
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
```

当触发SqlSession的方法时，SqlSessionInterceptor执行invoke方法时，构建实际的SqlSession对象。

```java
SqlSession sqlSession = getSqlSession(
  SqlSessionTemplate.this.sqlSessionFactory,
  SqlSessionTemplate.this.executorType,
  SqlSessionTemplate.this.exceptionTranslator);
```

但并不是每次都创建的新的SqlSession，新创建的SqlSession交给Spring事务管理器进行管理。

```java
SqlSessionUtils.java

public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    // 1.从事务同步管理器中获取SqlSessionHolder
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    // 2.从SqlSessionHolder中获取SqlSession，存在即返回
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }
    // 3.SqlSession不存在，创建新的SqlSession
    session = sessionFactory.openSession(executorType);
    // 4.注册SqlSession到事务同步管理器
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
    return session;
  }
```

TransactionSynchronizationManager类统一管理连接资源，内部维护了一个ThreadLocal级的Map结构，创建的SqlSession存储在Map中，也意味着Spring环境下Mapper接口执行方法时SqlSession是线程共享的。

```java
private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
```

SessionFactory获取SqlSession的逻辑，具体见[Mybatis源码解析(三)-执行Mapper请求](https://my.oschina.net/u/2377110/blog/4285611)，拿到真正的SqlSession，后续的操作就是Mybatis内部执行的逻辑，这里就不赘述了。

## 2.批量Mapper配置

大部分情况下，业务需要对一个目录下的所有Mapper都进行配置，此时使用MapperScannerConfigurer扫描配置类。

```xml
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="com.lcifn.orm.mybatis.mapper"/>
		<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
	</bean>
```

MapperScannerConfigurer实现spring的BeanDefinitionRegistryPostProcessor接口，在注册BeanDefinition时启动Mapper扫描器。postProcessBeanDefinitionRegistry方法调用ClassPathMapperScanner的scan方法。

```java
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```

ClassPathMapperScanner继承自ClassPathBeanDefinitionScanner，重写了doScan方法，将Mapper接口信息维护成BeanDefinitionHolder类，其中beanClass属性为MapperFactoryBean字节码对象。

```java
definition.setBeanClass(this.mapperFactoryBean.getClass());
```

初始化Mapper接口对应的对象时，通过MapperFactoryBean工厂Bean创建一个Mapper接口的动态代理对象，实现执行的是MapperProxy类。调用Mapper接口的方法真正执行的是MapperProxy的invoke方法。这里就回到了单条Mapper配置的逻辑了，参考上面1.2小节。

总体上说，Mybatis整合Spring时，主要是兼容Spring的IOC和事务管理机制，实际执行基本沿用Mybatis的内部逻辑。