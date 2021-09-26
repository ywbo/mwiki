[TOC]

# Mybatis源码解析（三）- 执行Mapper请求

Mybatis执行Mapper的过程，以查询user信息为例，测试代码如下:

```java
SqlSession session = sqlSessionFactory.openSession();
UserMapper userMapper = session.getMapper(UserMapper.class);
User user = userMapper.getUser(1L);
log.info(user.toString());
session.close();
```

包含三个动作：

1. 获取SqlSession
2. 创建Mapper接口的代理对象
3. 执行Mapper的方法请求

## 1. 获取SqlSession

SqlSessionFactory接口的默认实现类是DefaultSqlSessionFactory。当SqlSessionFactoryBuilder构建完成后，就得到了封装了Configuration配置的DefaultSqlSessionFactory对象。而SqlSession对象，通过openSession方法获得。

```java
SqlSession session = sqlSessionFactory.openSession();
```

实际通过openSessionFromDataSource方法，创建Mybatis的事务对象和执行器对象。

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 1.获取环境配置
      final Environment environment = configuration.getEnvironment();
      // 2.从环境配置查询事务工厂类
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 3.根据环境中的DataSource配置创建事务对象
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 4.根据执行器类型构建执行器实例
      final Executor executor = configuration.newExecutor(tx, execType);
      // 5.封装配置、执行器，返回SqlSession对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

以一个简单的mybatis的环境配置为例，其中定义了事务管理器的类型，以及配置数据库的连接。

```xml
<environments default="development">
		<environment id="development">
			<transactionManager type="JDBC"></transactionManager>
			<dataSource type="POOLED">
				<property name="driver" value="com.mysql.jdbc.Driver" />
				<property name="url" value="jdbc:mysql://ip:3306/user" />
				<property name="username" value="admin" />
				<property name="password" value="admin" />
			</dataSource>
		</environment>
</environments>
```

openSessionFromDataSource方法，从环境配置中，查询事务工厂类，对应mybatis的配置是transactionManager的标签，根据type=JDBC，获取到的TransactionFactory的实现是JdbcTransactionFactory。

TransactionFactory的工厂方法newTransaction，传入DataSouce对象，得到实例JdbcTransaction。DataSouce对象也是环境配置的type=POOLED，获取DataSouceFactory的实现为PooledDataSourceFactory，执行工厂方法getDataSouce获得实例PooledDataSource。

其中TransactionFactory和DataSouce的type和Class对象的对应关系，在Configuration构造函数中初始化。

```java
typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
```

拿到Transaction对象后，再来创建Mybatis核心的执行器，调用Configuration的newExecutor，以命令模式实例化不同的执行器。执行器的类型由settings配置的defaultExecutorType指定，默认为SIMPLE。另外cacheEnabled配置为Mybatis的二级缓存开关，默认开启，在原始执行器上封装一个CachingExecutor处理二级缓存。在执行器生成最后，给予插件修改的节点。

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

DefaultSqlSession的构造方法封装了Configuration对象和Executor对象，实例化出SqlSession对象。此时的SqlSession持有执行器和待执行的数据，长弓在手，只待拔箭。

```java
  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
```

## 2.创建Mapper接口的代理对象

有了SqlSession，要执行Mapper接口的方法，必须要创建Mapper接口的代理对象，这也是Mybatis核心处理的逻辑。

```java
UserMapper userMapper = session.getMapper(UserMapper.class);
```

Mapper的信息都存在Configuration对象中，因而SqlSession的getMapper实际调用的是Configuration的getMapper方法，实际操作的是MapperRegistry。

```java
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
  
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

MapperRegistry内部维护了一个Mapper接口对象和MapperProxyFactory对象关系的knownMappers，存储了mapper xml文件绑定的所有namespace类型。在解析mapper xml文件后，将namespace的类型添加到knownMappers中。而在生成Mapper接口对象时，再根据Mapper接口类型从knownMappers获取。

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

获取到MapperProxyFactory，调用工厂方法newInstance创建代理对象。MapperProxy对象即Mapper接口的通用代理对象执行器，实现Java动态代理的执行器接口InvocationHandler。使用Java的Proxy.newProxyInstance方法创建Mapper接口的代理对象。

```java
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```

执行Mapper接口时，调用的是MapperProxy的invoke方法。

## 3. 执行Mapper的方法请求

终于要执行Mapper方法了，我们调用user的查询方法。

```java
User user = userMapper.getUser(1L);
```

上面提到Mapper的代理对象执行器是MapperProxy，实现了InvocationHandler接口。因此调用接口方法时，根据java的动态代理逻辑，由InvocationHandler.invoke方法执行。在MapperProxy的invoke方法中，将Method对象封装成MapperMethod。

```java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

MapperMethod对象，根据Mapper接口+方法名，从Configuration对象中获取对应的MappedStatement，组装SqlCommand。sqlCommandType在解析mapper xml时划分，比如SELECT/UPDATE/DELETE/INSERT/FLUSH。MethodSignature构造方法，解析Mapper方法的返回值类型，为执行数据请求后返回值的处理做准备。

```java
mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());

public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
  this.command = new SqlCommand(config, mapperInterface, method);
  this.method = new MethodSignature(config, mapperInterface, method);
}

public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      final String methodName = method.getName();
      final Class<?> declaringClass = method.getDeclaringClass();
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      name = ms.getId();
      type = ms.getSqlCommandType();
 }
```

拿到MapperMethod后，执行execute方法。根据不同的sqlCommandType，调用SqlSession的对应方法。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
  }
```

对于select方法，针对不同的返回值，返回单条还是多条，执行不同的SqlSession的select方法。测试类中获取user对象返回单条数据。

```java
public interface UserMapper {
	User getUser(Long id);
}
```

根据返回值，执行selectOne方法，实际调用的还是selectList，只是处理执行结果时做了单条处理。

selectList方法，先获取MappedStatement，交给执行器Executor执行。

```java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

Executor接口的query方法，调用BaseExecutor实现类。根据Mapper接口方法的参数获取执行sql对象BoundSql。另外创建了一个cacheKey，这个cache是Mybatis的二级缓存的实现，在后续的文章中会单独讨论缓存的问题，这里就先略过。

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

抛开各种缓存的处理，来到BaseExecutor的抽象方法doQuery，以默认的SimpleExecutor来看。

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
}
```

根据Configuration创建Statement的处理器，实例化RoutingStatementHandler对象。

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

RoutingStatementHandler只是一个装饰器，构造函数中使用命令模式，根据不同的statementType，创建实际的Statement处理器。默认为PreparedStatementHandler。

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
}
```

在PreparedStatementHandler的构造函数中，又创建了方法参数的处理器ParameterHandler和结果集处理器ResultSetHandler，默认的实现分别是DefaultParameterHandler和DefaultResultSetHandler。

```java
this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
```

PreparedStatementHandler实例化完成后，先做一步预处理，即prepareStatement方法

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
```

PreparedStatementHandler的预处理操作，包括Connection的prepareStatement操作，设置statement的查询超时时间和每次检索返回值的条数fetchSize。

```java
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
	  // PreparedStatementHandler执行的就是Connection的prepareStatement操作
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
}
```

做完预处理后，对替换参数进行赋值，通过ParameterHandler接口的默认实现DefaultParameterHandler执行，判断字段的JdbcType，根据对应的TypeHandler赋值。

```java
  public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```

最后调用PreparedStatementHandler的query方法，执行PreparedStatement的execute方法，再对返回的结果集进行处理。

```java
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
  }
```

结果集的处理相对繁琐，这里简单说一下流程。

1. 根据MappedStatement存储的ResultMap实例化返回值对象，比如User对象。
2. 根据返回值对象的字段类型(以此为主)和表字段的数据库类型，获取对应的TypeHandler处理器，比如Long类型对应的LongTypeHandler。
3. TypeHandler处理器从Statement的ResultSet结果集里获取字段对应的值，比如LongTypeHandler获取字段值执行ResultSet的getLong(columnName)方法
4. 反射的方式赋值字段值到返回值对象中。

至此，从数据库返回的数据解析完成后返回给了调用者，完成了Mapper调用的全部流程。

## 4.执行流程图

![执行流程图](F:\PersonalWork\mWiki\mNote\library\008-images\mybatis3.png)

下一篇：整合Spring
