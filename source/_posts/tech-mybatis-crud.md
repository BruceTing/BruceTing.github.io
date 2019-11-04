---
title: MyBatis源码分析之SQL执行流程
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-11-04 15:30:10
comments: true
tags:
 - mybatis
keywords: mybatis
description: 源码分析CRUD执行流程
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/crud.jpeg
---

## CRUD主流程
先把主类```MyBatisTest```代码粘以下：

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

以```select```为例来看看```mapper.selectUser(1L)```怎么执行的。  
在```session.getMapper(UserDAO.class)```中获取到的是一个```MapperProxy```的代理实例，那么就毫无疑问的会调用到```MapperProxy```的```invoke(...)```方法：

```org.apache.ibatis.binding.MapperProxy#invoke(...)
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (method.isDefault()) {
        if (privateLookupInMethod == null) {
          return invokeDefaultMethodJava8(proxy, method, args);
        } else {
          return invokeDefaultMethodJava9(proxy, method, args);
        }
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 构造MapperMethod实例，调用它的execute方法执行
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

看看```cachedMapperMethod(method)```方法：

```cachedMapperMethod(method)
  private MapperMethod cachedMapperMethod(Method method) {
    // 把method作为key，mapperMethod实例作为value缓存起来
    // 核心在MapperMethod构造中
    return methodCache.computeIfAbsent(method,
        k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }
```

继续看看```MapperMethod```源码：

```new MapperMethod(...)
  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    // 构造SqlCommand实例
    this.command = new SqlCommand(config, mapperInterface, method);
    /**
    * org.apache.ibatis.binding.MapperMethod.SqlCommand#SqlCommand：
    * public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
    *     // 获取方法名
    *     final String methodName = method.getName();
    *     // 获取方法所在类的类型
    *     final Class<?> declaringClass = method.getDeclaringClass();
    *     // 从Configuration中获取MappedStatement
    *     // Map<String, MappedStatement> mappedStatements
    *     // key：mapperInterface.getName() + "." + methodName
    *     MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass, configuration);
    *     if (ms == null) {
    *         if (method.getAnnotation(Flush.class) != null) {
    *             name = null;
    *             type = SqlCommandType.FLUSH;
    *         } else {
    *             throw new BindingException("Invalid bound statement (not found): " + mapperInterface.getName() + "." + methodName);
    *         }
    *      } else {
    *          // 方法全限定名，例：
    *          // com.springboot.springbootmybatis.dao.UserDAO.selectUser
    *          name = ms.getId();
    *          // SQL类型，例：SELECT
    *          type = ms.getSqlCommandType();
    *          if (type == SqlCommandType.UNKNOWN) {
    *              throw new BindingException("Unknown execution method for: " + name);
    *          }
    *     }
    * }
    */
    
    // 获取方法签名信息
    this.method = new MethodSignature(config, mapperInterface, method);
    /**
    * org.apache.ibatis.binding.MapperMethod.MethodSignature#MethodSignature
    * public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
    *     // 方法返回值类型解析
    *     Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
    *     if (resolvedReturnType instanceof Class<?>) {
    *         this.returnType = (Class<?>) resolvedReturnType;
    *     } else if (resolvedReturnType instanceof ParameterizedType) {
    *         this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
    *     } else {
    *         this.returnType = method.getReturnType();
    *     }
    *     this.returnsVoid = void.class.equals(this.returnType);
    *     this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
    *     this.returnsCursor = Cursor.class.equals(this.returnType);
    *     this.returnsOptional = Optional.class.equals(this.returnType);
    *     this.mapKey = getMapKey(method);
    *     this.returnsMap = this.mapKey != null;
    *     this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
    *     this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
    *     // Sql的参数解析，将参数根据索引位置以及参数名封装到
    *     // org.apache.ibatis.reflection.ParamNameResolver#names中，为后续占位符替换做准备
    *     // private final SortedMap<Integer, String> names
    *     // 例：select id, name from User where id = ?
    *     // 那么names中就是：names.put(0, "id")
    *     this.paramNameResolver = new ParamNameResolver(configuration, method);
    * }
    */
  }
```

获取到```MapperMethod```的实例之后，就调用它的```execute(...)```方法执行：

```execute(...)
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
        // 根据返回值类型，决定调用哪一个查询方法
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
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
根据```command.getType()```，调用相应的```case xxx```片段执行后，获取执行结果返回。

## INSERT、UPDATE、DELETE
```execute(...)
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
      ......
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
### method.convertArgsToSqlCommandParam(args)
构造占位符参数值，为后续的占位符替换做准备，参数的构造调用org.apache.ibatis.binding.MapperMethod.MethodSignature#convertArgsToSqlCommandParam方法：

```convertArgsToSqlCommandParam(Object[] args)
    // 很显然是从paramNameResolver中将占位符参数构造出来
  public Object convertArgsToSqlCommandParam(Object[] args) {
      return paramNameResolver.getNamedParams(args);
  }
  
  // 构造占位符参数
  // param.put(entry.getValue(), args[entry.getKey()])
  // key：参数名称，value：参数值
  public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
      return args[names.firstKey()];
    } else {
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
```

### update核心方法
不管```command.getType()```是```INSERT、UPDATE、DELETE```中的哪一种，最终都是调用```org.apache.ibatis.session.defaults.DefaultSqlSession#update```方法执行。

```
  public int insert(String statement) {
    return insert(statement, null);
  }

  @Override
  public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }

  @Override
  public int update(String statement) {
    return update(statement, null);
  }

  @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      // 根据statement获取MappedStatement
      // statement：namespace + "." + sql的id，例：
      // com.springboot.springbootmybatis.dao.UserDAO.insert
      MappedStatement ms = configuration.getMappedStatement(statement);
      
      // 调用org.apache.ibatis.executor.CachingExecutor#update方法，其实最终会委托给org.apache.ibatis.executor.BaseExecutor#update方法执行
      return executor.update(ms, wrapCollection(parameter));
      /**
      * org.apache.ibatis.executor.BaseExecutor#update
      * public int update(MappedStatement ms, Object parameter) throws SQLException {
      *     ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
      *     if (closed) {
      *         throw new ExecutorException("Executor was closed.");
      *     }
      *     clearLocalCache();
      *     return doUpdate(ms, parameter);
      * }
      * 
      * 先前的ExecutorType是SIMPLE，所以调用的是
      * org.apache.ibatis.executor.SimpleExecutor#doUpdate 方法
      * public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
      *     Statement stmt = null;
      *     try {
      *         Configuration configuration = ms.getConfiguration();
      *         
      *         // 获取执行的handle
      *         StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      *         /**
      *         * org.apache.ibatis.session.Configuration#newStatementHandler
      *         * public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
      *         *     StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
      *         *     // 执行自定义插件
      *         *     statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
      *         *     return statementHandler;
      *         * }
      *         */
      *
      *         // 这里返回的stmt是org.apache.ibatis.logging.jdbc.PreparedStatementLogger类型
      *         stmt = prepareStatement(handler, ms.getStatementLog());
      *         /**
      *         *
      *         * private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      *         *     Statement stmt;
      *         *     Connection connection = getConnection(statementLog);
      *         *     stmt = handler.prepare(connection, transaction.getTimeout());
      *         *     handler.parameterize(stmt);
      *         *     return stmt;
      *         * }
      *         *
      *         */
      *
      *         // 调用org.apache.ibatis.executor.statement.RoutingStatementHandler#update方法执行update
      *         // 最终会委托给org.apache.ibatis.executor.statement.PreparedStatementHandler#update执行
      *         return handler.update(stmt);
      *         /**
      *         * PreparedStatementHandler#update：
      *         * public int update(Statement statement) throws SQLException {
      *         *     // 是不是很熟悉，很熟悉就对了
      *         *     PreparedStatement ps = (PreparedStatement) statement;
      *         *     ps.execute();
      *         *     int rows = ps.getUpdateCount();
      *         *     Object parameterObject = boundSql.getParameterObject();
      *         *     KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
      *         *     keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
      *         *     return rows;
      *         * }
      *         *
      *         */
      *     } finally {
      *         closeStatement(stmt);
      *     }
      * }
      * 
      */
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  @Override
  public int delete(String statement) {
    return update(statement, null);
  }

  @Override
  public int delete(String statement, Object parameter) {
    return update(statement, parameter);
  }
```

### rowCountResult(...)
对执行的结果集进行封装org.apache.ibatis.binding.MapperMethod#rowCountResult：

```rowCountResult(...)
  // 根据返回值类型，封装结果
  private Object rowCountResult(int rowCount) {
    final Object result;
    if (method.returnsVoid()) {
      result = null;
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
      result = rowCount;
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
      result = (long)rowCount;
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
      result = rowCount > 0;
    } else {
      throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
    }
    return result;
  }
```

## SELECT

```execute(...)
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      ......
      case SELECT:
        // 根据返回值类型，决定调用哪一个查询方法
        // 无返回值时执行
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        // 返回多条数据时即List类型结果时执行
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        // 结果类型是一个Map时执行
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        // 暂时不讨论
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          // 其他情况时执行，例：返回的结果是User
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
        ......
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

### executeWithResultHandler(sqlSession, args)

```executeWithResultHandler(sqlSession, args)
  private void executeWithResultHandler(SqlSession sqlSession, Object[] args) {
    // 根据command.getName()即statementId获取MappedStatement
    MappedStatement ms = sqlSession.getConfiguration().getMappedStatement(command.getName());
    if (!StatementType.CALLABLE.equals(ms.getStatementType())
        && void.class.equals(ms.getResultMaps().get(0).getType())) {
      throw new BindingException("method " + command.getName()
          + " needs either a @ResultMap annotation, a @ResultType annotation,"
          + " or a resultType attribute in XML so a ResultHandler can be used as a parameter.");
    }
    // 占位符参数转换
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      sqlSession.select(command.getName(), param, rowBounds, method.extractResultHandler(args));
    } else {
      sqlSession.select(command.getName(), param, method.extractResultHandler(args));
    }
  }
  
  /**
  * org.apache.ibatis.session.defaults.DefaultSqlSession#select
  * @Override
  * public void select(String statement, Object parameter, ResultHandler handler) {
  *     select(statement, parameter, RowBounds.DEFAULT, handler);
  * }
  * @Override
  * public void select(String statement, ResultHandler handler) {
  *     select(statement, null, RowBounds.DEFAULT, handler);
  * }
  * 
  * // 最终都会执行到这个select方法
  * @Override
  * public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
  *     try {
  *         MappedStatement ms = configuration.getMappedStatement(statement);
  *         // 执行org.apache.ibatis.executor.BaseExecutor#query
  *         executor.query(ms, wrapCollection(parameter), rowBounds, handler);
  *     } catch (Exception e) {
  *         throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  *     } finally {
  *         ErrorContext.instance().reset();
  *     }
  * }
```

### executeForMany(sqlSession, args)

```executeForMany(sqlSession, args)
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.selectList(command.getName(), param);
    }
    // 结果集转换
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
  
  /**
  * org.apache.ibatis.session.defaults.DefaultSqlSession#selectList
  * @Override
  * public <E> List<E> selectList(String statement) {
  *     return this.selectList(statement, null);
  * }
  * @Override
  * public <E> List<E> selectList(String statement, Object parameter) {
  *     return this.selectList(statement, parameter, RowBounds.DEFAULT);
  * }
  * 
  * // 最终都会执行到这个selectList方法
  * @Override
  * public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  *     try {
  *         MappedStatement ms = configuration.getMappedStatement(statement);
  *         // 执行org.apache.ibatis.executor.BaseExecutor#query
  *         return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  *     } catch (Exception e) {
  *         throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  *     } finally {
  *         ErrorContext.instance().reset();
  *     }
  * }
  */
```

### executeForMap(sqlSession, args)

```executeForMap(sqlSession, args)
  private <K, V> Map<K, V> executeForMap(SqlSession sqlSession, Object[] args) {
    Map<K, V> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.selectMap(command.getName(), param, method.getMapKey(), rowBounds);
    } else {
      result = sqlSession.selectMap(command.getName(), param, method.getMapKey());
    }
    return result;
  }
  
  /**
  * org.apache.ibatis.session.defaults.DefaultSqlSession#selectMap
  * @Override
  * public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
  *     return this.selectMap(statement, null, mapKey, RowBounds.DEFAULT);
  * }
  * @Override
  * public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
  *     return this.selectMap(statement, parameter, mapKey, RowBounds.DEFAULT);
  * }
  * 
  * // 最终都会执行到这个selectMap方法
  * @Override
  * public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
  *     // 首先执行selectList查询
  *     final List<? extends V> list = selectList(statement, parameter, rowBounds);
  *     // 将list结果集转换成Map<K, V>
  *     final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<>(mapKey,
            configuration.getObjectFactory(), configuration.getObjectWrapperFactory(), configuration.getReflectorFactory());
  *     final DefaultResultContext<V> context = new DefaultResultContext<>();
  *     for (V o : list) {
  *         context.nextResultObject(o);
  *         // 将结果集以key，value的形式放到org.apache.ibatis.executor.result.DefaultMapResultHandler#mappedResults Map集合中
  *         mapResultHandler.handleResult(context);
  *     }
  *     return mapResultHandler.getMappedResults();
  * }
  *
  */
```

### selectOne(command.getName(), param)

```selectOne(command.getName(), param)
  @Override
  public <T> T selectOne(String statement) {
    return this.selectOne(statement, null);
  }

  @Override
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    // 执行selectList查询，获取返回结果集的一个元素
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }
```

### executor.query(...)
首先调用org.apache.ibatis.executor.CachingExecutor#query进行缓存管理：

```CachingExecutor#query
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  // 缓存管理
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 委托给org.apache.ibatis.executor.BaseExecutor#query
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    // 委托给org.apache.ibatis.executor.BaseExecutor#query
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

最终库查询调用委托给org.apache.ibatis.executor.BaseExecutor#query

```executor.query(...)
  // select以及selectList都会执行这个方法，只不过最后一个参数不一样
  // select：ResultHandler handler
  // selectList：Executor.NO_RESULT_HANDLER
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 构造缓存key，例：
    // -849388093:2205047678:com.springboot.springbootmybatis.dao.UserDAO.selectUser:0:2147483647:select ID, USER_ID, NAME, EMAIL, CREATED_TIME, MODIFIED_TIME from USER where id = ?:5:development
    // 第一个：hashCode，第二个：类似于一个验证码
    // 第三个：sql语句，第四个：sql语句参数
    // 第五个：环境id
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }
  
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 从缓存查询
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      // 缓存有数据直接返回
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 查询DB
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
  
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 查询数据
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    // 缓存数据
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
  
  // 以下操作就和update差不多啦
  // 调用org.apache.ibatis.executor.SimpleExecutor#doQuery
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      // 获取到一个RoutingStatementHandler实例
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      // 获取到一个org.apache.ibatis.logging.jdbc.PreparedStatementLogger实例
      stmt = prepareStatement(handler, ms.getStatementLog());
      // 调用RoutingStatementHandler的query方法，其实最终还是委托给了
      // org.apache.ibatis.executor.statement.PreparedStatementHandler#query
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
  
  // 调用org.apache.ibatis.executor.statement.PreparedStatementHandler#query
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    // 很熟悉的操作
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
  }
  
  // 调用org.apache.ibatis.executor.resultset.DefaultResultSetHandler#handleResultSets进行结果集处理
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      // 到这里处理结果集，将处理结果放到multipleResults集合中
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }

```
## 小总结

* ```Mapper```的执行实际上是调用了获取到的```MapperProxy```实例的```invoke```方法
* 在```invoke```方法中构造```MapperMethod```实例后，调用它的```execute```方法
* 在```execute```方法中根据```command.getType()```类型选择相应的方法调用，可以分为两类
  * INSERT、UPDATE、DETETE
     
     * 三个操作汇总调用```org.apache.ibatis.session.defaults.DefaultSqlSession#update```方法执行
     * 然后调用```org.apache.ibatis.executor.CachingExecutor#update```
     * 再将执行委托给```org.apache.ibatis.executor.BaseExecutor#update```执行
     * 再调用```org.apache.ibatis.executor.SimpleExecutor#doUpdate```
     * 然后调用```org.apache.ibatis.executor.statement.PreparedStatementHandler#update```执行获取结果
     * 最后调用```org.apache.ibatis.binding.MapperMethod#rowCountResult```处理结果返回

  * SELECT
     * ```executeWithResultHandler``` 后面调用```org.apache.ibatis.session.defaults.DefaultSqlSession#select```执行
     * ```executeForMany```、```executeForMap```、```selectOne``` 都是调用```org.apache.ibatis.session.defaults.DefaultSqlSession#selectList```执行
     * 以上两步最后都是调用```org.apache.ibatis.executor.CachingExecutor#query```
     * 然后委托给```org.apache.ibatis.executor.BaseExecutor#query```执行
     * 再调用```org.apache.ibatis.executor.SimpleExecutor#doQuery```方法
     * 然后调用```org.apache.ibatis.executor.statement.PreparedStatementHandler#query```
     * 最后调用```org.apache.ibatis.executor.resultset.DefaultResultSetHandler#handleResultSets```处理结果集返回

再来张```Executor```的图看看，对其中```xxxExecutor```的调用关系就更清楚了：
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/crud-executor.jpg)








