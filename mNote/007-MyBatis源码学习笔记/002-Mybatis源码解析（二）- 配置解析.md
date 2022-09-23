[TOC]

# Mybatis源码解析（二）- 配置解析

> Mybatis加载配置文件，通过解析xml，构建了一个Configuration对象，包含所有的配置项及Mapper接口的映射关系。可以说，配置文件的解析，为之后Mapper代理对象的创建做好了所有的准备。而配置的解析就在SqlSessionFactoryBuilder的build方法中。

```java
InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
```

## 一、获取xml文件Document对象

build方法是一个工厂方法，用来创建SqlSessionFactory对象，主要操作就是解析xml文件，加载配置。

```java
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
return build(parser.parse()); 
```

XMLConfigBuilder是Mybatis的配置构建器，够早喊出里生成XPathParser对象来解析xml文件。

```java
public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}
```

XPathParser是对xpath解析xml的封装，它持有xml的Document对象引用，在构造方法中使用DOM方式解析xml，然后获得Document对象。

```java
public XPathParser(InputStream inputStream,...) {
    this.document = createDocument(new InputSource(inputStream));
}

private Document createDocument(InputSource inputSource) {
	  // DocumentBuilderFactory是DOM方式解析xml的工厂类
      DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
      factory.setValidating(validation);
      factory.setNamespaceAware(false);
      factory.setIgnoringComments(true);
      factory.setIgnoringElementContentWhitespace(false);
      factory.setCoalescing(false);
      factory.setExpandEntityReferences(true);
      DocumentBuilder builder = factory.newDocumentBuilder();
	  ...
      return builder.parse(inputSource);
}
```

## 二、构建Configuration

SqlSessionFactoryBuilder的build方法拿到Document对象后，执行XMLConfigBuilder的parse方法，解析<configuration>标签。

```java
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```

parseConfiguration是解析mybatis配置的核心方法，不同的子标签使用不同的方法进行解析。

```java
 private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      // 类型昵称解析
      typeAliasesElement(root.evalNode("typeAliases"));
      // 插件解析
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // 设置项解析
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      // mapper文件解析
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

mybatis的配置我关注的就是settings、mapper两个标签，settings标签决定mybatis的运行方式，mapper标签决定mybatis要处理哪些mapper文件。

settingsElement方法初始化通用配置项，包括DefaultExecutorType（默认执行器类型为SIMPLE）等，这个方法比较简单，就不详细介绍了。

mapperElement方法读取mapper xml文件，缓存sql执行配置，以及Mapper接口和mapper xml的映射关系，存储在Configuration对象的mappedStatements的Map中。

mapper的配置有两种方式，一种是package标签，解析指定目录下的所有文件；另一种是resource/url/class三个标签，直接指定mapper xml文件的位置进行解析。

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
      	// 1.解析package标签配置
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
        // 2.解析resource/url/class标签配置
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

### 1. resource配置

resource配置指定xml文件解析，通过XMLConfigBuilder的mapperElement方法中，解析resource配置。

```java
InputStream inputStream = Resources.getResourceAsStream(resource);
XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
mapperParser.parse();
```

获取到xml文件路径后，通过XMLMapperBuilder.parse执行

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }
  }
```

交给XMLMapperBuilder.parse方法解析mapper xml文件

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      // 1.解析mapper根标签
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      // 2.绑定xml文件内容到namespace上，namespace即Mapper接口
      bindMapperForNamespace();
    }
  }
```

configurationElement解析参数、返回类型及sql配置

```java
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

至此，可以看到select|insert|update|delete标签的处理了，往下看一下

```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

实际由XMLStatementBuilder.parseStatementNode来解析

```java
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

	...
    
    // 解析动态sql
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    ...

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

解析完sql后，由MapperStatement的构建助手MapperBuilderAssistant调用addMappedStatement创建MappedStatement对象。此方法的参数实在太多。

```java
 public MappedStatement addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {

    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }

	// 构建MappedStatement
    MappedStatement statement = statementBuilder.build();
    // 添加到Configuration配置中
    configuration.addMappedStatement(statement);
    return statement;
  }
```

configuration.addMappedStatement将构建好的MappedStatement添加到mappedStatements的Map中，key为namespace+sql语句的id，从而调用Mapper接口的方法时，通过拼接接口和方法的字符串，就可以在Configuration对象中找到对应的MappedStatement，即获取到sql执行的所有信息。

```java
  public void addMappedStatement(MappedStatement ms) {
    mappedStatements.put(ms.getId(), ms);
  }
```



### 2. package标签解析

XMLConfigBuilder的mapperElement方法中，将package的name直接交给Configuration的addMappers方法进行解析和注册，实际由MapperRegistry处理。

```java
mapperRegistry.addMappers(packageName);
```

```java
public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }
```

ResolverUtil.find方法，寻找packageName包路径下的class文件，认定为Mapper接口对象。后续从Mapper接口类型出发，解析同路径下的xml文件。

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
      	// Mapper接口和代理对象工厂类Map
        knownMappers.put(type, new MapperProxyFactory<T>(type));
		// Mapper注解解析器，同时解析注解和xml两种方式的配置
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        // 实际解析方法
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
MapperAnnotationBuilder.parse方法

  public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      // 1.解析xml配置
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      
      // 2.解析注解配置
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          // issue #237
          if (!method.isBridge()) {
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }
```

这里只看xml配置的解析，将Mapper接口的.class替换成.xml读取

```java
private void loadXmlResource() {
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
      // 替换xml文件路径
      String xmlResource = type.getName().replace('.', '/') + ".xml";
      InputStream inputStream = null;
      try {
        inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
      } catch (IOException e) {
        // ignore, resource is not required
      }
      if (inputStream != null) {
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        // 解析xml
        xmlParser.parse();
      }
    }
```

这里就回到了resource配置解析指定xml文件的方式。

当所有的配置都解析完成，Configuration对象就创建完毕了。



## 三、创建SQLSessionFactory

SqlSessionFactoryBuilder的build方法，给定Configuration对象，构造一个DefaultSqlSessionFactory对象。

```java
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

在DefaultSqlSessionFactory工厂类中，我们可以调用openSession方法打开通向数据库的大门。

```java
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
```

究竟一条sql是怎么执行的，请看下一章。