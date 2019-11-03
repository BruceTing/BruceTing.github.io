---
title: MyBatis源码分析之SqlSessionFactory
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-11-03 15:30:10
comments: true
tags:
 - mybatis
keywords: mybatis
description: 源码分析SqlSessionFactory的构造过程，包括Environments、Plugins、TypeHandlers、ResultMaps、Mappers等的解析过程。
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/ssfactory.jpeg
---

## 构造SqlSessionFactory的主流程

学习过基本用法之后呢，接着来分析分析源码，首先来看看其中的```SqlSessionFactory```，我们把之前的```MyBatisTest```粘一下：

```MyBatisTest
public class MyBatisTest {

    public static void main(String[] args) throws Exception {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);

        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        try (SqlSession session = sqlSessionFactory.openSession()) {
            UserDAO mapper = session.getMapper(UserDAO.class);
            User user = mapper.selectUser(1L);
            System.out.println(user.toString());
        }

    }

}
```
先来看看这行代码干了啥：

```SqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```


org.apache.ibatis.session.SqlSessionFactoryBuilder#build(java.io.InputStream) 开始构造```SqlSessionFactory```：
> org.apache.ibatis.builder.xml.XMLConfigBuilder#XMLConfigBuilder(java.io.InputStream, java.lang.String, java.util.Properties)  配置文件、环境、解析器初始化
> > > org.apache.ibatis.builder.xml.XMLMapperEntityResolver  构造mybatis dtd文件解析器

> > org.apache.ibatis.parsing.XPathParser#XPathParser(java.io.InputStream, boolean, java.util.Properties, org.xml.sax.EntityResolver) 初始化文件解析器XPathParser，根据配置文件，构造文件的Document对象

> > org.apache.ibatis.session.Configuration 注册通用别名，比如：JDBC -> JdbcTransactionFactory.class，LOG4J -> Log4jImpl.class等，这个是非常重要非常重要非常重要的类，我们所有的配置以及操作都和它相关
> > 
> > ```Configuration()
> > public Configuration() {
    // JDBC
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);
    // JNDI
    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);
    // CACHE
    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class);
    // DB_VENDOR
    typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);
    // XML
    typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
    typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);
    // LOG
    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);
    // CGLIB
    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);
    // LANGUAGE
    languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
    languageRegistry.register(RawLanguageDriver.class);
  }
> > ```


> >org.apache.ibatis.builder.xml.XMLConfigBuilder#parse 配置文件解析标识，避免重复解析
> > > org.apache.ibatis.builder.xml.XMLConfigBuilder#parseConfiguration 解析配置文件的核心代码：
> > > 
> > > ```configuration
> > >  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      // 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有的不带
      // databaseId 或匹配当前 databaseId 的语句；如果带或者不带的语句都有，
      // 则不带的会被忽略。
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
> > > ``` 

> org.apache.ibatis.session.SqlSessionFactoryBuilder#build(org.apache.ibatis.session.Configuration) 根据解析出来的配置构造SqlSessionFactory：
> 
> ```SqlSessionFactory
>   public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
> ```


通过上面的代码分析知道，我们配置的```mybatis-config.xml```配置文件，正是在```new SqlSessionFactoryBuilder().build(inputStream)```方法下进行解析的，主要来看看```typeAliases、plugins、environments、typeHandlers、mappers```配置怎么解析的，解析到哪里去了。

## typeAliases解析
直接上源码：org.apache.ibatis.builder.xml.XMLConfigBuilder#typeAliasesElement：

```typeAliasesElement(XNode parent)
  private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      // 将配置的数据注册到TypeAliasRegistry的typeAliases属性中：
      // private final Map<String, Class<?>> typeAliases = new HashMap<>();
      // key: alise  value: type
      // 如果alise为null，则key为：type.getSimpleName()
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```

## plugins解析
直接上源码：org.apache.ibatis.builder.xml.XMLConfigBuilder#pluginElement：

```pluginElement(XNode parent)
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
        interceptorInstance.setProperties(properties);
        // 解析出每个plugin，反射实例化后，注册到configuration的interceptorChain属性中，
        // 其实是放到interceptorChain的list集合org.apache.ibatis.plugin.InterceptorChain#interceptors中：
        // private final List<Interceptor> interceptors = new ArrayList<>();
        // 每个插件处理器都实现了Interceptor接口，我们可以实现Interceptor接口，实现自定义的插件处理器
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

## environments解析
先看看源码org.apache.ibatis.builder.xml.XMLConfigBuilder#environmentsElement：

```environmentsElement(XNode context)
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      // 我们在调用new SqlSessionFactoryBuilder().build(inputStream)的时候，只是把配置文件传进去，没有传数据源ID或其他的属性配置，所以这个地方会去获取默认的数据源配置，刚好我们在配置的时候指定了default，通过传入数据源ID，可以达到使用多数据源的目的。注意：在使用默认环境数据源时，一定要保证默认的环境 ID 要匹配其中一个环境 ID。
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        // 这个地方就是环境ID校验了
        if (isSpecifiedEnvironment(id)) {
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }
```
#### 来看看环境ID是怎么校验的，很好理解：

```isSpecifiedEnvironment
  private boolean isSpecifiedEnvironment(String id) {
    if (environment == null) {
      throw new BuilderException("No environment specified.");
    } else if (id == null) {
      throw new BuilderException("Environment requires an id attribute.");
    } else if (environment.equals(id)) {
      return true;
    }
    return false;
  }
```
### 事物TransactionFactory解析

```TransactionFactory
  private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    if (context != null) {
      // 事物管理器类型，有两种[JDBC|MANAGED]：
      // JDBC：直接使用 JDBC 的提交和回滚设置，它依赖于从数据源得到的连接来管理事务的作用域。
      // MANAGED：这个配置几乎没做什么，既不提交也不会滚，让容器自己去管理，默认情况下，它只会关闭连接，然而并不是所有的容器允许这么做，通常需要将 closeConnection 属性设置为 false 来阻止它默认的关闭行为，列入：
      // <transactionManager type="MANAGED">
      //    <property name="closeConnection" value="false"/>
      // </transactionManager>
      // 这两种事务管理器类型都不需要设置任何属性。它们其实是类型别名，换句话说，你可以使用 TransactionFactory 接口的实现类的完全限定名或类型别名代替它们。
      // 属性配置，如果使用 Spring + MyBatis，则没有必要配置事务管理器， 因为 Spring 模块会使用自带的管理器来覆盖前面的配置。
      String type = context.getStringAttribute("type");
      // 属性配置
      Properties props = context.getChildrenAsProperties();
      // 反射获取TransactionFactory的实例
      TransactionFactory factory = (TransactionFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 属性设置
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }
```
#### 来看看resolveClass(type)里都干了啥
```resolveClass(type)
  protected <T> Class<? extends T> resolveClass(String alias) {
    if (alias == null) {
      return null;
    }
    try {
      return resolveAlias(alias);
    } catch (Exception e) {
      throw new BuilderException("Error resolving class. Cause: " + e, e);
    }
  }
  
  protected <T> Class<? extends T> resolveAlias(String alias) {
    return typeAliasRegistry.resolveAlias(alias);
  }
  
  org.apache.ibatis.type.TypeAliasRegistry#resolveAlias:
  public <T> Class<T> resolveAlias(String string) {
    try {
      if (string == null) {
        return null;
      }
      // issue #748
      String key = string.toLowerCase(Locale.ENGLISH);
      Class<T> value;
      if (typeAliases.containsKey(key)) {
        value = (Class<T>) typeAliases.get(key);
      } else {
        value = (Class<T>) Resources.classForName(string);
      }
      return value;
    } catch (ClassNotFoundException e) {
      throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
    }
  }
```
这个就是通过注册的别名去获取Class，通过```JDBC```获取到事物管理器Class：```JdbcTransactionFactory.class```，然后反射拿到它的实例，也就是```TransactionFactory```啦。

### DataSource解析
这个地方其实就是和```TransactionFactory```解析一样的啦：

```DataSourceFactory
  private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      DataSourceFactory factory = (DataSourceFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
  }
```
最后通过反射拿到```PooledDataSourceFactory```实例。来看看```getDeclaredConstructor().newInstance()```构造器里干了啥：

```PooledDataSourceFactory
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {

  public PooledDataSourceFactory() {
    this.dataSource = new PooledDataSource();
  }

}
```
```PooledDataSourceFactory```继承```UnpooledDataSourceFactory```，```UnpooledDataSourceFactory```实现```DataSourceFactory```。```new PooledDataSource()```里干了啥：

```PooledDataSource()
  public PooledDataSource() {
    dataSource = new UnpooledDataSource();
  }
```
```PooledDataSource```、```UnpooledDataSource```实现了```DataSource```，所以看到的是```PooledDataSource```，其实使用的是```UnpooledDataSource ```。看到的```PooledDataSourceFactory```，其实使用的```UnpooledDataSourceFactory```，也就是说干的事情都委托给了```UnpooledDataSource ```和```UnpooledDataSourceFactory```。所以最终获取到的```DataSource```是```UnpooledDataSource ```。

### 构建Environment，配置Configuration
```environmentsElement(XNode context)```方法中最后的几行代码就是把获取到的```TransactionFactory```、```DataSource```实例扔给```Environment.Builder```构造出```Environment```对象，然后赋值给```Configuration ```的```environment ```属性：

```org.apache.ibatis.mapping.Environment.Builder#build
    public Environment build() {
      return new Environment(this.id, this.transactionFactory, this.dataSource);
    }
```
至此，```Environment ```解析完毕。

## typeHandlers解析
先看看源码org.apache.ibatis.builder.xml.XMLConfigBuilder#typeHandlerElement：

```typeHandlerElement(XNode parent)
  private void typeHandlerElement(XNode parent) {
    if (parent != null) {
      // 解析出配置的typeHandler，并注册到configuration的typeHandlerRegistry中
      // 每个类型处理器均实现了TypeHandler接口，我们可以实现TypeHandler接口，自定义自己的类型解析器
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }
```

## mappers解析
先看看源码org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement：

```mapperElement(XNode parent)
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 解析配置了扫描 mappers 的package节点
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          /**
          * 解析配置了扫描 mappers 的 mapper 节点
          * Mapper的配置方式有三种，有且只能配置其中一种，每种都可以配置多个：
          * 1、使用相对于类路径的资源引用，例如：
          * <mappers>
          *     <mapper resource="org/mybatis/builder/UserMapper.xml"/>
          *     ....
          * </mappers>
          * 
          * 2、使用完全限定资源定位符（URL），例如：
          * <mappers>
          *     <mapper url="file:///var/mappers/UserMapper.xml"/>
          *     ....
          * </mappers>
          * 
          * 3、使用映射器接口实现类的完全限定类名，例如：
          * <mappers>
          *     <mapper class="org.mybatis.builder.UserMapper"/>
          *     ....
          * </mappers>
          */
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            // Mapper方式解析
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            // URL方式解析
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            // 完全限定类方式解析
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

### Package方式解析
看看```configuration.addMappers(mapperPackage)```这行代码里面干了啥：

```org.apache.ibatis.session.Configuration#addMappers(java.lang.String)
  public void addMappers(String packageName) {
    mapperRegistry.addMappers(packageName);
  }
```
原来是把扫描出的```Mapper接口```注册到```mapperRegistry```中，接着来看：

```org.apache.ibatis.binding.MapperRegistry#addMappers(java.lang.String)
  /**
   * @since 3.2.2
   */
  public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }

  /**
   * @since 3.2.2
   */
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }
```
后面调用```addMappers(String packageName, Class<?> superType)```方法并且指定检查的超类是```Object.class```，该超类对象通过```new ResolverUtil.IsA(superType)```赋值给```IsA implements Test```类中的```parent```属性，用作后面的类型检查。调用```ResolverUtil```实例的```find```方法，将匹配的类的Class对象放到```ResolverUtil ```实例的```matches```集合中：

```match类
  public ResolverUtil<T> find(Test test, String packageName) {
    String path = getPackagePath(packageName);

    try {
      List<String> children = VFS.getInstance().list(path);
      for (String child : children) {
        if (child.endsWith(".class")) {
          addIfMatching(test, child);
        }
      }
    } catch (IOException ioe) {
      log.error("Could not read package: " + packageName, ioe);
    }

    return this;
  }
  
    protected void addIfMatching(Test test, String fqn) {
    try {
      String externalName = fqn.substring(0, fqn.indexOf('.')).replace('/', '.');
      ClassLoader loader = getClassLoader();
      if (log.isDebugEnabled()) {
        log.debug("Checking to see if class " + externalName + " matches criteria [" + test + "]");
      }

      Class<?> type = loader.loadClass(externalName);
      if (test.matches(type)) {
        matches.add((Class<T>) type);
      }
    } catch (Throwable t) {
      log.warn("Could not examine class '" + fqn + "'" + " due to a " +
          t.getClass().getName() + " with message: " + t.getMessage());
    }
  }
```
回到```addMappers(String packageName, Class<?> superType)```中，最终把匹配的类注册到```org.apache.ibatis.binding.MapperRegistry#knownMappers```Map集合中：

```MapperRegistry#knownMappers
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

### Mapper方式解析
来看看```new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments())```里面干了啥：

```XMLMapperBuilder
  public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    // 这个地方和解析 Configuration 时是差不多的
    // 然后创建xxxMapper.xml的Document对象
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }

  private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }
```
因为```XMLMapperBuilder extends BaseBuilder```，所以看看```super(configuration)```里干了啥：

```org.apache.ibatis.builder.BaseBuilder#BaseBuilder
  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```
从```configuration```里把```typeAliasRegistry、typeHandlerRegistry```拿出来赋值给BaseBuilder中的```typeAliasRegistry、typeHandlerRegistry```。  
```MapperBuilderAssistant extends BaseBuilder```，看看构造方法：

```MapperBuilderAssistant(...)
  public MapperBuilderAssistant(Configuration configuration, String resource) {
    // 这里的super和上面是一样的
    super(configuration);
    ErrorContext.instance().resource(resource);
    this.resource = resource;
  }
```
最后拿到```XMLMapperBuilder```实例后，调用它的```parse()```方法：

```parse()
  public void parse() {
    // 检查configuration的Set集合loadedResources中是否包含当前的resource
    if (!configuration.isResourceLoaded(resource)) {
      // 开始去解析Mapper，将最终的结果集存入configuration中
      configurationElement(parser.evalNode("/mapper"));
      // 保存已解析过的mapper
      configuration.addLoadedResource(resource);
      // 将已加载过的Mapper的namespace注册到configuration的mapperRegistry属性中的
      // knownMappers集合中：
      // private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
      bindMapperForNamespace();
    }
    
    // 将未完成解析的ResultMaps解析到configuration的resultMaps集合中
    parsePendingResultMaps();
    /**
    * private void parsePendingResultMaps() {
    *    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    *    synchronized (incompleteResultMaps) {
    *        Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
    *        while (iter.hasNext()) {
    *           try {
    *               // org.apache.ibatis.builder.ResultMapResolver#resolve
    *               iter.next().resolve();
    *               iter.remove();
    *           } catch (IncompleteElementException e) {
    *               // ResultMap is still missing a resource...
    *           }
    *        }
    *    }
    * }
    */
    // 缓存解析
    parsePendingCacheRefs();
    // SQL解析：select | insert | update | delete
    // 将未完成解析的SQLStatements解析到configuration的mappedStatements集合中
    parsePendingStatements();
    /**
    * private void parsePendingStatements() {
    *     Collection<XMLStatementBuilder> incompleteStatements = configuration.getIncompleteStatements();
    *     synchronized (incompleteStatements) {
    *         Iterator<XMLStatementBuilder> iter = incompleteStatements.iterator();
    *         while (iter.hasNext()) {
    *             try {
    *                 // org.apache.ibatis.builder.xml.XMLStatementBuilder#parseStatementNode
    *                 iter.next().parseStatementNode();
    *                 iter.remove();
    *             } catch (IncompleteElementException e) {
    *                 // Statement is still missing a resource...
    *             }
    *         }
    *     }
    * }
    */
  }
```

#### configurationElement(parser.evalNode("/mapper"))

```org.apache.ibatis.builder.xml.XMLMapperBuilder#configurationElement```源码：

```configurationElement(...)
  private void configurationElement(XNode context) {
    try {
      // namespace检查与设置
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      // 二级缓存配置：可以在当前namespace中使用<cache/>开启二级缓存，
      // 当前空间的缓存只在当前空间有效，如果需要引用其他空间的缓存，则需要配置<cache-ref/>
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      // 解析引用外部的参数parameterMap，已经废弃
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // resultMap 节点数据解析，resultMap节点可以配置多个，id区分，将最终的解析结果已
      // resultMap的id为key，resultMap为value存入configuration的resultMaps集合中
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // sql节点解析
      sqlElement(context.evalNodes("/mapper/sql"));
      // select|insert|update|delete 节点解析
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```


##### 解析ResultMap
org.apache.ibatis.builder.xml.XMLMapperBuilder#resultMapElements：

```resultMapElements
  private void resultMapElements(List<XNode> list) throws Exception {
    for (XNode resultMapNode : list) {
      try {
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }
  
  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.emptyList(), null);
  }
  
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // 获取ResultMap的类型，我们通常配置的都是type，也即是：JAVA实体类的类型
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    // 通过别名或是全限定名获取type的Class对象
    Class<?> typeClass = resolveClass(type);
    if (typeClass == null) {
      // type不配或是配不正确，resolveClass都会报错，所以到不了这个地方，暂不研究
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        /** 解析在ResultMap中配置的java实体构造器节点，例如：
        * <constructor>
        *   <idArg column="id" javaType="int"/>
        *   <arg column="username" javaType="String"/>
        *   ......
        * </constructor>
        * 使用类似String column = context.getStringAttribute("column");方式获取节点数据
        * 将解析出来的节点数据放入ResultMapping中
        */
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        /**
        * 使用结果值来决定使用哪个 resultMap
        * 同样的将节点的数据放入ResultMapping中，并将ResultMapping结果集存入Discriminator的resultMapping属性中
        */
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        /**
        * 主要来看看这个buildResultMappingFromContext(resultChild, typeClass, flags)方法，从方法就可以看出，是用于构造ResultMapping，一个result对应就对应一个resultMapping
        */
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    // resultMap的id
    String id = resultMapNode.getStringAttribute("id",
            resultMapNode.getValueBasedIdentifier());
    // 继承于其他resultMap的id
    String extend = resultMapNode.getStringAttribute("extends");
    // 是否开启/关闭自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      // 获取ResultMap
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
  
  private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    // 获取配置的各种属性值
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
      property = context.getStringAttribute("name");
    } else {
      property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
        processNestedResultMappings(context, Collections.emptyList(), resultType));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    // 通过别名或全限定名解析java实体类的类型
    Class<?> javaTypeClass = resolveClass(javaType);
    // 通过别名或全限定名解析类型处理器的类型
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    // 通过别名获取jdbc的类型
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    // 根据解析出来的属性值构造ResultMapping
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
  }
  
  org.apache.ibatis.builder.MapperBuilderAssistant#buildResultMapping：
  public ResultMapping buildResultMapping(
      Class<?> resultType,
      String property,
      String column,
      Class<?> javaType,
      JdbcType jdbcType,
      String nestedSelect,
      String nestedResultMap,
      String notNullColumn,
      String columnPrefix,
      Class<? extends TypeHandler<?>> typeHandler,
      List<ResultFlag> flags,
      String resultSet,
      String foreignColumn,
      boolean lazy) {
    // 解析property对应的java类型
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    // 根据typeHandlerType或javaTypeClass和typeHandlerType从typeHandlerRegistry中获取TypeHandler
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    List<ResultMapping> composites;
    if ((nestedSelect == null || nestedSelect.isEmpty()) && (foreignColumn == null || foreignColumn.isEmpty())) {
      composites = Collections.emptyList();
    } else {
      composites = parseCompositeColumnName(column);
    }
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
  }
  
  private Class<?> resolveResultJavaType(Class<?> resultType, String property, Class<?> javaType) {
    if (javaType == null && property != null) {
      try {
        /** 
        * MetaClass.forClass(resultType,configuration.getReflectorFactory())
        * 就是通过resultType、reflectorFactory构造MetaClass，构造方法：
        * private MetaClass(Class<?> type, ReflectorFactory reflectorFactory) {
        *     this.reflectorFactory = reflectorFactory;
        *     this.reflector = reflectorFactory.findForClass(type);
        * }
        * 
        * org.apache.ibatis.reflection.DefaultReflectorFactory#findForClass方法
        * public Reflector findForClass(Class<?> type) {
        *     if (classCacheEnabled) {
        *          // synchronized (type) removed see issue #461
        *          // private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<>();
        *          return reflectorMap.computeIfAbsent(type,Reflector::new);
        *     } else {
        *          return new Reflector(type);
        *     }
        * }
        * 
        * public class Reflector {
        *
        *     private final Class<?> type;
        *     private final String[] readablePropertyNames;
        *     private final String[] writablePropertyNames;
        *     private final Map<String, Invoker> setMethods = new HashMap<>();
        *     private final Map<String, Invoker> getMethods = new HashMap<>();
        *     private final Map<String, Class<?>> setTypes = new HashMap<>();
        *     private final Map<String, Class<?>> getTypes = new HashMap<>();
        *     private Constructor<?> defaultConstructor;
        *
        *     private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();
        *
        *     public Reflector(Class<?> clazz) {
        *         type = clazz;
        *         // 添加默认构造器
        *         addDefaultConstructor(clazz);
        *         // clazz的所有Get方法
        *         addGetMethods(clazz);
        *         // clazz的所有Set方法
        *         addSetMethods(clazz);
        *         // clazz的所有属性
        *         addFields(clazz);
        *         readablePropertyNames = getMethods.keySet().toArray(new String[0]);
        *         writablePropertyNames = setMethods.keySet().toArray(new String[0]);
        *         for (String propName : readablePropertyNames) {
        *             caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        *         }
        *         for (String propName : writablePropertyNames) {
        *             caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        *         }
        *     }
        *     .......
        * }
        * 
        */
        MetaClass metaResultType = MetaClass.forClass(resultType, configuration.getReflectorFactory());
        // 通过Reflector获取属性Setter方法的类型，因为从数据库获取到值的时候，
        // 会通过typeHandler转成property对应的SetterType，将值set进去
        javaType = metaResultType.getSetterType(property);
      } catch (Exception e) {
        //ignore, following null check statement will deal with the situation
      }
    }
    if (javaType == null) {
      javaType = Object.class;
    }
    return javaType;
  }
  
  public ResultMapping build() {
      // lock down collections
      resultMapping.flags = Collections.unmodifiableList(resultMapping.flags);
      resultMapping.composites = Collections.unmodifiableList(resultMapping.composites);
      // TypeHandler解析
      resolveTypeHandler();
      // 安全性验证
      validate();
      return resultMapping;
  }
  
  private void resolveTypeHandler() {
      if (resultMapping.typeHandler == null && resultMapping.javaType != null) {
        Configuration configuration = resultMapping.configuration;
        TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
        resultMapping.typeHandler = typeHandlerRegistry.getTypeHandler(resultMapping.javaType, resultMapping.jdbcType);
      }
  }
  
  private void validate() {
      // Issue #697: cannot define both nestedQueryId and nestedResultMapId
      if (resultMapping.nestedQueryId != null && resultMapping.nestedResultMapId != null) {
        throw new IllegalStateException("Cannot define both nestedQueryId and nestedResultMapId in property " + resultMapping.property);
      }
      // Issue #5: there should be no mappings without typehandler
      if (resultMapping.nestedQueryId == null && resultMapping.nestedResultMapId == null && resultMapping.typeHandler == null) {
        throw new IllegalStateException("No typehandler found for property " + resultMapping.property);
      }
      // Issue #4 and GH #39: column is optional only in nested resultmaps but not in the rest
      if (resultMapping.nestedResultMapId == null && resultMapping.column == null && resultMapping.composites.isEmpty()) {
        throw new IllegalStateException("Mapping is missing column attribute for property " + resultMapping.property);
      }
      if (resultMapping.getResultSet() != null) {
        int numColumns = 0;
        if (resultMapping.column != null) {
          numColumns = resultMapping.column.split(",").length;
        }
        int numForeignColumns = 0;
        if (resultMapping.foreignColumn != null) {
          numForeignColumns = resultMapping.foreignColumn.split(",").length;
        }
        if (numColumns != numForeignColumns) {
          throw new IllegalStateException("There should be the same number of columns and foreignColumns in property " + resultMapping.property);
        }
      }
  }
  
  最后来看看org.apache.ibatis.builder.ResultMapResolver#resolve：
  public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
  }
  
  org.apache.ibatis.builder.MapperBuilderAssistant#addResultMap：
  public ResultMap addResultMap(
      String id,
      Class<?> type,
      String extend,
      Discriminator discriminator,
      List<ResultMapping> resultMappings,
      Boolean autoMapping) {
    // 命名空间设置
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);
    // 解析继承的result
    if (extend != null) {
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
      ResultMap resultMap = configuration.getResultMap(extend);
      List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
      extendedResultMappings.removeAll(resultMappings);
      // Remove parent constructor if this resultMap declares a constructor.
      boolean declaresConstructor = false;
      for (ResultMapping resultMapping : resultMappings) {
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          declaresConstructor = true;
          break;
        }
      }
      if (declaresConstructor) {
        extendedResultMappings.removeIf(resultMapping -> resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR));
      }
      resultMappings.addAll(extendedResultMappings);
    }
    // 通过解析出来的数据构造resultMap
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    // 将resultMap的id作为key，resultMap作为value保存到Configuration的resultMaps属性中
    configuration.addResultMap(resultMap);
    return resultMap;
  }
  
  public ResultMap build() {
      if (resultMap.id == null) {
        throw new IllegalArgumentException("ResultMaps must have an id");
      }
      // 存放column
      resultMap.mappedColumns = new HashSet<>();
      // 存放property
      resultMap.mappedProperties = new HashSet<>();
      // 存放id
      resultMap.idResultMappings = new ArrayList<>();
      // 存放构造器
      resultMap.constructorResultMappings = new ArrayList<>();
      // 存放整个result结果集
      resultMap.propertyResultMappings = new ArrayList<>();
      // 存放构造器参数
      final List<String> constructorArgNames = new ArrayList<>();
      // 解析获取到的resultMap.resultMappings
      for (ResultMapping resultMapping : resultMap.resultMappings) {
        resultMap.hasNestedQueries = resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
        resultMap.hasNestedResultMaps = resultMap.hasNestedResultMaps || (resultMapping.getNestedResultMapId() != null && resultMapping.getResultSet() == null);
        final String column = resultMapping.getColumn();
        if (column != null) {
          resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
        } else if (resultMapping.isCompositeResult()) {
          for (ResultMapping compositeResultMapping : resultMapping.getComposites()) {
            final String compositeColumn = compositeResultMapping.getColumn();
            if (compositeColumn != null) {
              resultMap.mappedColumns.add(compositeColumn.toUpperCase(Locale.ENGLISH));
            }
          }
        }
        final String property = resultMapping.getProperty();
        if (property != null) {
          resultMap.mappedProperties.add(property);
        }
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          resultMap.constructorResultMappings.add(resultMapping);
          if (resultMapping.getProperty() != null) {
            constructorArgNames.add(resultMapping.getProperty());
          }
        } else {
          // 没有构造器就把整个result结果集存起来
          resultMap.propertyResultMappings.add(resultMapping);
        }
        if (resultMapping.getFlags().contains(ResultFlag.ID)) {
          resultMap.idResultMappings.add(resultMapping);
        }
      }
      if (resultMap.idResultMappings.isEmpty()) {
        resultMap.idResultMappings.addAll(resultMap.resultMappings);
      }
      // 解析实际的构造参数
      if (!constructorArgNames.isEmpty()) {
        final List<String> actualArgNames = argNamesOfMatchingConstructor(constructorArgNames);
        if (actualArgNames == null) {
          throw new BuilderException("Error in result map '" + resultMap.id
              + "'. Failed to find a constructor in '"
              + resultMap.getType().getName() + "' by arg names " + constructorArgNames
              + ". There might be more info in debug log.");
        }
        resultMap.constructorResultMappings.sort((o1, o2) -> {
          int paramIdx1 = actualArgNames.indexOf(o1.getProperty());
          int paramIdx2 = actualArgNames.indexOf(o2.getProperty());
          return paramIdx1 - paramIdx2;
        });
      }
      
      // 已解析过的数据不可变
      // lock down collections
      resultMap.resultMappings = Collections.unmodifiableList(resultMap.resultMappings);
      resultMap.idResultMappings = Collections.unmodifiableList(resultMap.idResultMappings);
      resultMap.constructorResultMappings = Collections.unmodifiableList(resultMap.constructorResultMappings);
      resultMap.propertyResultMappings = Collections.unmodifiableList(resultMap.propertyResultMappings);
      resultMap.mappedColumns = Collections.unmodifiableSet(resultMap.mappedColumns);
      return resultMap;
  }
```

##### 解析sqlElement
org.apache.ibatis.builder.xml.XMLMapperBuilder#sqlElement：

```sqlElement
  private void sqlElement(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
  }

  private void sqlElement(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      String databaseId = context.getStringAttribute("databaseId");
      String id = context.getStringAttribute("id");
      id = builderAssistant.applyCurrentNamespace(id, false);
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
        // 将sqlElement保存到org.apache.ibatis.builder.xml.XMLMapperBuilder#sqlFragments 集合中
        // private final Map<String, XNode> sqlFragments
        sqlFragments.put(id, context);
      }
    }
  }
```

##### 解析select | insert | update | delete

```buildStatementFromContext
  private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      // org.apache.ibatis.builder.xml.XMLStatementBuilder
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
  
  // 开始解析
  public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    // 获取节点类型
    // SqlCommandType：UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    // 解析包含的<include/>节点
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());
	
	 // 解析参数类型
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
 
    // 解析数据库语言驱动LanguageDriver：XMLLanguageDriver
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // 解析selectKey，通常是配置的需要自动生成值的键
    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }
    
    /** 
    * 解析SQL
    * langDriver.createSqlSource(configuration, context, parameterTypeClass)
    * 来分析下：
    * org.apache.ibatis.scripting.xmltags.XMLLanguageDriver#createSqlSource
    * public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
    *     XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
    *     return builder.parseScriptNode();
    * }
    *
    * org.apache.ibatis.scripting.xmltags.XMLScriptBuilder#XMLScriptBuilder
    * public XMLScriptBuilder(Configuration configuration, XNode context, Class<?> parameterType) {
    *     /**
    *     * super(configuration)
    *     * 配置赋值 XMLScriptBuilder extends org.apache.ibatis.builder.BaseBuilder#BaseBuilder
    *     * public BaseBuilder(Configuration configuration) {
    *     *     this.configuration = configuration;
    *     *     this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    *     *     this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
    *     * }
    *     */
    *     super(configuration);
    *     this.context = context;
    *     this.parameterType = parameterType;
    *     initNodeHandlerMap();
    * }
    * 动态SQL处理初始化
    * private void initNodeHandlerMap() {
    *     nodeHandlerMap.put("trim", new TrimHandler());
    *     nodeHandlerMap.put("where", new WhereHandler());
    *     nodeHandlerMap.put("set", new SetHandler());
    *     nodeHandlerMap.put("foreach", new ForEachHandler());
    *     nodeHandlerMap.put("if", new IfHandler());
    *     nodeHandlerMap.put("choose", new ChooseHandler());
    *     nodeHandlerMap.put("when", new IfHandler());
    *     nodeHandlerMap.put("otherwise", new OtherwiseHandler());
    *     nodeHandlerMap.put("bind", new BindHandler());
    * }
    * 
    * org.apache.ibatis.scripting.xmltags.XMLScriptBuilder#parseScriptNode
    * public SqlSource parseScriptNode() {
    *     MixedSqlNode rootSqlNode = parseDynamicTags(context);
    *     SqlSource sqlSource;
    *     // DynamicSqlSource、RawSqlSourc、StaticSqlSource均实现了SqlSource接口
    *     if (isDynamic) {
    *         // 直接构造一个DynamicSqlSource对象，对于"${}"的动态SQL直接构造返回，
    *         // 不会对其进行解析，因此容易造成SQL注入攻击
    *         sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    *         /**
    *         * public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
    *         *    this.configuration = configuration;
    *         *    this.rootSqlNode = rootSqlNode;
    *         * }
    *         */
    *     } else {
    *         // 在RawSqlSource中对sql进行解析，对于"#{}"每次都会进行解析，可以预防SQL注入攻击
    *         sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    *         /**
    *         * org.apache.ibatis.scripting.defaults.RawSqlSource#RawSqlSource
    *         * public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
    *         *     this(configuration, getSql(configuration, rootSqlNode), parameterType);
    *         * }
    *         * public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    *         *     SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    *         *     Class<?> clazz = parameterType == null ? Object.class : parameterType;
    *         *     this.sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap());
    *         * }
    *         *
    *         * org.apache.ibatis.builder.SqlSourceBuilder#parse
    *         * public SqlSourceBuilder(Configuration configuration) {
    *         *     // 初始化configuration、typeAliasRegistry、typeHandlerRegistry
    *         *     super(configuration);
    *         * }
    *         *
    *         * public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    *         *     SqlSourceBuilder.ParameterMappingTokenHandler handler = new SqlSourceBuilder.ParameterMappingTokenHandler(this.configuration, parameterType, additionalParameters);
    *         *     GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    *         *     // 解析占位符"#{}"，构造SQL
    *         *     String sql = parser.parse(originalSql);
    *         *     // 返回解析出的静态SQL
    *         *     return new StaticSqlSource(this.configuration, sql, handler.getParameterMappings());
    *         * }
    *         */
    *     }
    *     return sqlSource;
    * }
    * 
    * parseDynamicTags(context) 
    * 解析SQL放入contents集合
    * SQL会被解析成3个部分：select、查询的属性、剩余的一段，例如：
    * select id,name,email from user where id = 1
    * 解析出来后就是：
    *   1、select
    *   2、id,name,email
    *   3、from user where id = 1
    * 
    * protected MixedSqlNode parseDynamicTags(XNode node) {
    *     List<SqlNode> contents = new ArrayList<>();
    *     NodeList children = node.getNode().getChildNodes();
    *     for (int i = 0; i < children.getLength(); i++) {
    *         XNode child = node.newXNode(children.item(i));
    *         // 节点类型：org.w3c.dom.Node
    *         if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
    *             String data = child.getStringBody("");
    *             TextSqlNode textSqlNode = new TextSqlNode(data);
    *             // 根据是否包含"${}"判断是否是动态SQL
    *             if (textSqlNode.isDynamic()) {
    *                 contents.add(textSqlNode);
    *                 isDynamic = true;
    *             } else {
    *                 contents.add(new StaticTextSqlNode(data));
    *             }
    *         } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
    *             String nodeName = child.getNode().getNodeName();
    *             // 根据节点名称从initNodeHandlerMap()中获取相应的SQL动态处理器
    *             NodeHandler handler = nodeHandlerMap.get(nodeName);
    *             if (handler == null) {
    *                 throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
    *             }
    *             // 使用获取到的处理器处理动态SQL
    *             handler.handleNode(child, contents);
    *             // 如果节点类型是：Node.ELEMENT_NODE，也会判定为动态SQL
    *             isDynamic = true;
    *         }
    *     }
    *     // 将contents放入MixedSqlNode的contents集合中
    *     /**
    *     * org.apache.ibatis.scripting.xmltags.MixedSqlNode#MixedSqlNode
    *     * public class MixedSqlNode implements SqlNode {
    *     *    private final List<SqlNode> contents;
    *     *    public MixedSqlNode(List<SqlNode> contents) {
    *     *        this.contents = contents;
    *     *    }
    *     *
    *     *    @Override
    *     *    public boolean apply(DynamicContext context) {
    *     *        contents.forEach(node -> node.apply(context));
    *     *        return true;
    *     *    }
    *     * }
    *     */
    *     return new MixedSqlNode(contents);
    * }
    *
    */
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    // 获取声明类型StatementType：STATEMENT, PREPARED, CALLABLE
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    // 获取设置的其他的属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");
    
    // 构造SQL的MappedStatement，MappedStatement是啥呢，往下看
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
  
  /**
  * org.apache.ibatis.builder.MapperBuilderAssistant#addMappedStatement
  * // MappedStatement就是用于存放解析出来的每个SQL的元数据信息，看看构造参数就知道了哈
  * public MappedStatement addMappedStatement(
  *    String id,
  *    SqlSource sqlSource,
  *    StatementType statementType,
  *    SqlCommandType sqlCommandType,
  *    Integer fetchSize,
  *    Integer timeout,
  *    String parameterMap,
  *    Class<?> parameterType,
  *    String resultMap,
  *    Class<?> resultType,
  *    ResultSetType resultSetType,
  *    boolean flushCache,
  *    boolean useCache,
  *    boolean resultOrdered,
  *    KeyGenerator keyGenerator,
  *    String keyProperty,
  *    String keyColumn,
  *    String databaseId,
  *    LanguageDriver lang,
  *    String resultSets) {
  *
  *  if (unresolvedCacheRef) {
  *    throw new IncompleteElementException("Cache-ref not yet resolved");
  *  }
  *
  *  id = applyCurrentNamespace(id, false);
  *  boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
  *
  *  MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
  *      .resource(resource)
  *      .fetchSize(fetchSize)
  *      .timeout(timeout)
  *      .statementType(statementType)
  *      .keyGenerator(keyGenerator)
  *      .keyProperty(keyProperty)
  *      .keyColumn(keyColumn)
  *      .databaseId(databaseId)
  *      .lang(lang)
  *      .resultOrdered(resultOrdered)
  *      .resultSets(resultSets)
  *      .resultMaps(getStatementResultMaps(resultMap, resultType, id))
  *      .resultSetType(resultSetType)
  *      .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
  *      .useCache(valueOrDefault(useCache, isSelect))
  *      .cache(currentCache);
  *
  *  ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
  *  /**
  *  * private ParameterMap getStatementParameterMap(
  *  *   String parameterMapName,
  *  *   Class<?> parameterTypeClass,
  *  *   String statementId) {
  *  *     parameterMapName = applyCurrentNamespace(parameterMapName, true);
  *  *     ParameterMap parameterMap = null;
  *  *       if (parameterMapName != null) {
  *  *          try {
  *  *              parameterMap = configuration.getParameterMap(parameterMapName);
  *  *          } catch (IllegalArgumentException e) {
  *  *              throw new IncompleteElementException("Could not find parameter map " + parameterMapName, e);
  *  *          }
  *  *       } else if (parameterTypeClass != null) {
  *  *          List<ParameterMapping> parameterMappings = new ArrayList<>();
  *  *          parameterMap = new ParameterMap.Builder(configuration, statementId + "-Inline", parameterTypeClass, parameterMappings).build();
  *  *       }
  *  *   return parameterMap;
  *  * }
  *  */
  *  if (statementParameterMap != null) {
  *    statementBuilder.parameterMap(statementParameterMap);
  *  }
  *
  *  MappedStatement statement = statementBuilder.build();
  *  /**
  *  * org.apache.ibatis.mapping.MappedStatement.Builder#build
  *  * public MappedStatement build() {
  *  *   assert mappedStatement.configuration != null;
  *  *   assert mappedStatement.id != null;
  *  *   assert mappedStatement.sqlSource != null;
  *  *   assert mappedStatement.lang != null;
  *  *   // 解析出来的结果就不允许改变了，否则会导致数据一致性问题
  *  *   mappedStatement.resultMaps = Collections.unmodifiableList(mappedStatement.resultMaps);
  *  *   return mappedStatement;
  *  * }
  *  */
  *  // 最终把数据放到configuration的mappedStatements属性中：
  *  // protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>(...)
  *  configuration.addMappedStatement(statement);
  *  return statement;
  * }
  *
  */
```

### 资源限定符（URL）方式解析
这种方式其实就是和Mapper方式解析是一样的啦，就是数据的来源不一样而已！

### 全限定类名方式解析
这种方式也很简单，直接把类加载出来注册到configuration的属性mapperRegistry中的org.apache.ibatis.binding.MapperRegistry#knownMappers属性中

## 小总结
构造```SqlSessionFactory```的过程就是对配置文件进行解析的过程，然后根据配置文件返回一个```DefaultSqlSessionFactory```。配置文件的解析主要涉及：

* typeAliases的解析  
  ```结果注册到：org.apache.ibatis.type.TypeAliasRegistry#typeAliases```  
  
* plugins的解析  
  ```结果注册到：org.apache.ibatis.plugin.InterceptorChain#interceptors```
  
* environments的解析：
  * 多数据源的解析
  
  ```结果注册到：org.apache.ibatis.session.Configuration#environment```  
  
* typeHandlers的解析  
  * 如果JavaType不为null，其中key为javatype
    ```注册到org.apache.ibatis.type.TypeHandlerRegistry#typeHandlerMap(private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>())```
    
  * 如果JavaType为null，其中key为handle的type
    ```注册到org.apache.ibatis.type.TypeHandlerRegistry#allTypeHandlersMap(private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>())```  
  
* mappers的解析
  * parameterMaps的解析
    ```结果注册到：org.apache.ibatis.session.Configuration#parameterMaps```
    
  * resultMap的解析  
    ```结果注册到：org.apache.ibatis.session.Configuration#resultMaps```  
    
  * sql节点的解析  
    ```结果注册到：org.apache.ibatis.builder.xml.XMLMapperBuilder#sqlFragments```
    
  * select | insert | update | delete的解析
     * 动态SQL与静态SQL的解析
     * 区别${} 和 #{}的解析 
 
    ```结果注册到：org.apache.ibatis.session.Configuration#mappedStatements```

以上数据解析出来之后，最终的结果都会汇总到configuration中，也就是说以上解析出来的结果在configuration中都有一个属性与之对应。



