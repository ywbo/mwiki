[TOC]

# Mybatis源码解析(五)-一级缓存和二级缓存

数据库的使用，方便了数据的存储和查询。但同时对查询的性能优化，也一直是ORM框架的重要部分之一。Mybatis建立了两级缓存机制：一级缓存和二级缓存，分别针对session内的查询优化，以及跨session的查询优化。下面就对这两级缓存机制的实现原理进行介绍。

## 1.一级缓存

session级缓存，在同一SqlSession下，执行相同的查询sql，第一次从数据库查询，并写到缓存中；第二次直接取缓存中取。当每次执行增删改的操作后，都会清空SqlSession的缓存。

Mybatis默认开启一级缓存，默认在setting配置种设置。

```xml
<setting name="localCacheScope" value="SESSION"/>
```

如果不想使用一级缓存，可以设置value为STATEMENT,这样每执行一个Mapper语句都会清空一级缓存。

一级缓存由PerpetualCache实现，内部维护一个Map结构。

```java
public class PerpetualCache implements Cache {

  private Map<Object, Object> cache = new HashMap<Object, Object>();java
  
}
```

Executor接口实现类创建时，PerpetualCache实例化被抽象类BaseExecutor引用。

```java
this.localCache = new PerpetualCache("LocalCache");
```

### 1.1 查询时一级缓存实现

Executor的抽象类BaseExecutor执行query方法，创建一级缓存的key，创建维度为[xml文件的查询方法id+查询sql+偏移量+参数值+环境配置id]。

```java
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();

    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        value = metaObject.getValue(propertyName);
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

在查询数据库之前，先从PerpetualCache获取缓存，如果存在则直接返回。

```java
// 获取缓存
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
if (list != null) {
  handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
} else {
  list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

如果没有缓存，则从数据库获取数据后，按上面的cacheKey存放到缓存中。

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
  	// 从数据库获取数据
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  // 放入缓存
  localCache.putObject(key, list);
  return list;
}
```

配置中设置localCacheScope为STATEMENT，执行完查询后再清空缓存。所以开关并不是不执行缓存操作，而是执行后又清除了。

```java
if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
  // issue #482
  clearLocalCache();
}
```

### 1.2 更新清除一级缓存

SqlSession内执行更新操作，包括insert/update/delete，都通过PerpetualCache的clear方法清空localCache。

```java
public int update(MappedStatement ms, Object parameter) throws SQLException {
  clearLocalCache();
  return doUpdate(ms, parameter);
}

  public void clearLocalCache() {
    if (!closed) {
      localCache.clear();
      localOutputParameterCache.clear();
    }
  }
```

或者执行commit/rollback时也会清空缓存

```java
public void commit(boolean required) throws SQLException {
  clearLocalCache();
  flushStatements();
  if (required) {
    transaction.commit();
  }
}

public void rollback(boolean required) throws SQLException {
    if (!closed) {
      try {
        clearLocalCache();
        flushStatements(true);
      } finally {
        if (required) {
          transaction.rollback();
        }
      }
    }
}
```

因为一级缓存的PerpetualCache对象是被Executor对象引用，而Executor对象和SqlSession同生命周期，因此一级缓存的生命周期最大就是Session会话维度。

## 2.二级缓存

mapper映射文件的缓存，同一namespace下的mapper映射文件内容，在不同SqlSession下，可以共享。

二级缓存的配置是默认启动的

```xml
<setting name="cacheEnabled" value="true" />
```

但每个mapper映射文件需要手动启用配置，在mapper中配置cache标签。

```xml
<mapper namespace="...UserMapper">
    <cache/>java
</mapper>
```

启用二级缓存后，默认所有select语句都配置cache，如果不想启用，可以指定select语句的useCache属性为false。

### 2.1 二级缓存配置解析

cache的配置解析包括在mapper文件的解析中(XMLMapperBuilder.parse方法)

```java
configurationElement(parser.evalNode("/mapper"));
=>
cacheElement(context.evalNode("cache"));
```

cache配置可以指定缓存实现Class，缓存剔除策略，刷新周期等属性，具体属性如下：

- type：指定底层缓存实现类，默认为PerpetualCache。也可以自定义缓存实现，type指定自定义Cache全路径名称。
- eviction：缓存元素的剔除策略，默认为LRU，对应LruCache，最多保存1024个Key，超出按最近最少使用进行剔除。系统总共提供四种剔除策略。
  - `LRU` – 最近最少使用：移除最长时间不被使用的对象。
  - `FIFO` – 先进先出：按对象进入缓存的顺序来移除它们。
  - `SOFT` – 软引用：基于垃圾回收器状态和软引用规则移除对象。
  - `WEAK` – 弱引用：更积极地基于垃圾收集器状态和弱引用规则移除对象。
- flushInterval：清空缓存的时间间隔，单位是毫秒。每次对缓存操作时判断距离上一次清空缓存的时间超过了flushInterval，则清空当前缓存，原理见ScheduleCache类。
- size：缓存中最多保存的Key数量。LruCache默认存储最多1024个元素。
- readOnly：是否只读，默认为false。这里的只读指的是缓存返回的数据能否修改，当指定为false时，要求缓存对象实现序列号接口Serializable，写缓存时将对象序列化，读缓存时进行反序列号，返回一个新的对象。此时的缓存对象就可以进行修改，不会影响原有缓存对象。原理见SerializedCache类。

默认<cache/>标签效果如下：

- 映射语句文件中的所有 select 语句的结果将会被缓存。
- 映射语句文件中的所有 insert、update 和 delete 语句会刷新缓存。
- 缓存会使用最近最少使用算法（LRU, Least Recently Used）算法来清除不需要的缓存。
- 缓存不会定时进行刷新（flushInterval为空）。
- 缓存会保存列表或对象（无论查询方法返回哪种）的 1024 个引用（size=1024）。
- 缓存会被视为读/写缓存（readOnly=false），这意味着获取到的对象并不是共享的，可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改。

```java
private void cacheElement(XNode context) throws Exception {
  if (context != null) {
  	// 缓存实现类
    String type = context.getStringAttribute("type", "PERPETUAL");
    Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
    // 缓存剔除策略
    String eviction = context.getStringAttribute("eviction", "LRU");
    Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
    // 刷新周期
    Long flushInterval = context.getLongAttribute("flushInterval");
    // 缓存元素数量
    Integer size = context.getIntAttribute("size");
    boolean readWrite = !context.getBooleanAttribute("readOnly", false);
    boolean blocking = context.getBooleanAttribute("blocking", false);
    Properties props = context.getChildrenAsProperties();
    builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
  }
}
```

根据cache配置，构建cache对象

```java
public Cache useNewCache(Class<? extends Cache> typeClass,
    Class<? extends Cache> evictionClass,
    Long flushInterval,
    Integer size,
    boolean readWrite,
    boolean blocking,
    Properties props) {
  Cache cache = new CacheBuilder(currentNamespace)
      .implementation(valueOrDefault(typeClass, PerpetualCache.class))
      .addDecorator(valueOrDefault(evictionClass, LruCache.class))
      .clearInterval(flushInterval)
      .size(size)
      .readWrite(readWrite)
      .blocking(blocking)
      .properties(props)
      .build();
  configuration.addCache(cache);
  currentCache = cache;
  return cache;
}
```

build方法使用装饰模式不断封装缓存实现类，返回实际操作的Cache实现。

```java
public Cache build() {
  setDefaultImplementations();
  Cache cache = newBaseCacheInstance(implementation, id);
  setCacheProperties(cache);
  // issue #352, do not apply decorators to custom caches
  if (PerpetualCache.class.equals(cache.getClass())) {
    for (Class<? extends Cache> decorator : decorators) {
      cache = newCacheDecoratorInstance(decorator, cache);
      setCacheProperties(cache);
    }
    cache = setStandardDecorators(cache);
  } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
    cache = new LoggingCache(cache);
  }
  return cache;
}
```

setDefaultImplementations方法指定底层缓存实现类为PerpetualCache，装饰器为LruCache。

```java
private void setDefaultImplementations() {
  if (implementation == null) {
    implementation = PerpetualCache.class;
    if (decorators.isEmpty()) {
      decorators.add(LruCache.class);
    }
  }
}
```

setStandardDecorators设置标准装饰器流程，根据cache的配置增加不同的装饰器，最终返回查询使用的缓存实现。此时的缓存绑定在mapper的namespace维度上。

```java
private Cache setStandardDecorators(Cache cache) {
  try {
    MetaObject metaCache = SystemMetaObject.forObject(cache);
    if (size != null && metaCache.hasSetter("size")) {
      metaCache.setValue("size", size);
    }
    // 清除周期配置
    if (clearInterval != null) {
      cache = new ScheduledCache(cache);
      ((ScheduledCache) cache).setClearInterval(clearInterval);
    }
    // readOnly配置
    if (readWrite) {
      cache = new SerializedCache(cache);
    }
    // 默认日志输出和方法级加锁
    cache = new LoggingCache(cache);
    cache = new SynchronizedCache(cache);
    if (blocking) {
      cache = new BlockingCache(cache);
    }
    return cache;
  } catch (Exception e) {
    throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
  }
}
```

### 2.2 二级缓存实现

Executor的实现类由setting配置的executorType决定，默认为SimpleExecutor。创建完Executor的实现类后，对cacheEnabled进行判断，如果为true，CachingExecutor封装原始执行器实现类，开启二级缓存的全局开关。

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

SqlSession操作查询方法，由Executor执行query实际操作，当开启二级缓存后，调用的就是CachingExecutor的query方法。

从MappedStatement获取缓存实现对象，如果不为空，意味着配置了cache标签，然后再判断MappedStatement是否启用缓存，如果启用，使用TransactionalCacheManager管理缓存。

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
  // cache标签
  Cache cache = ms.getCache();
  if (cache != null) {
    flushCacheIfRequired(ms);
    // 默认select语句useCache="true"
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

**TransactionalCacheManager维护了一个SqlSession操作过程中临时的缓存，等到SqlSession执行commit或close时才会将临时缓存刷新到MappedStatement的缓存实现中，从而达到跨SqlSession的缓存共享。**

TransactionalCacheManager持有一个Map<Cache, TransactionalCache>结构，key为MappedStatement的缓存实现对象，value为临时缓存对象TransactionalCache类。TransactionalCache持有实际缓存对象的引用，entriesToAddOnCommit的map结构存储临时缓存。

```java
public TransactionalCache(Cache delegate) {
  this.delegate = delegate;
  this.clearOnCommit = false;
  this.entriesToAddOnCommit = new HashMap<Object, Object>();
  this.entriesMissedInCache = new HashSet<Object>();
}
```

TransactionalCache的putObject将缓存添加到entriesToAddOnCommit中，当SqlSession执行commit或close时，调用TransactionalCache的commit方法，刷新缓存到实际缓存实现中。

```java
TransactionalCache.java
public void putObject(Object key, Object object) {
  entriesToAddOnCommit.put(key, object);
}

public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
}

private void flushPendingEntries() {
	// 刷新临时缓存到实际缓存实现中
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    // 如果数据库也未查到，缓存设置为null
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
}
```

TransactionalCache的getObject方法，从实际缓存实现中获取，如果未查到，先存储到entriesMissedInCache中。

```java
public Object getObject(Object key) {
  // issue #116
  Object object = delegate.getObject(key);
  if (object == null) {
    entriesMissedInCache.add(key);
  }
}
```

等到执行完数据库查询后，赋值到缓存中。

### 2.3 LruCache实现原理

Lru算法，就是超量时移除最长时间不被使用的对象。LruCache使用有序集合LinkedHashMap来实现。在setSize方法时，将内部的keyMap创建为LinkedHashMap，同时重写了removeEldestEntry方法，判断如果超过size，则返回最早的元素。

```java
public void setSize(final int size) {
  keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
    private static final long serialVersionUID = 4267176411845948333L;

    @Override
    protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
      boolean tooBig = size() > size;
      // 超量返回最早的元素
      if (tooBig) {
        eldestKey = eldest.getKey();
      }
      return tooBig;
    }
  };
}
```

另外每次执行putObject方法，校验是否超量，如果超量，从实际缓存中移除最早的元素。

```java
private void cycleKeyList(Object key) {
  keyMap.put(key, key);
  // eldestKey不为空即超量
  if (eldestKey != null) {
    delegate.removeObject(eldestKey);
    eldestKey = null;
  }
}
```

由于二级缓存是Sql维度的细粒度缓存方式，任何更新操作会刷新二级缓存，实际工作中缓存的命中率并不高，权衡成本与收益，并不建议开启。缓存还是交给专门的缓存框架或中间件来操作更好。

参考文档：

1. [https://yq.aliyun.com/articles/608941](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fyq.aliyun.com%2Farticles%2F608941)