---
title: MyBatis源码分析之SqlSession
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-11-04 10:10:10
comments: true
tags:
 - mybatis
keywords: mybatis
description: 源码分析TransactionFactory、Executor以及SqlSession的构造过程
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/sqlsession.jpeg
---

## SqlSession主流程
先把```MyBatisTest```主方法源码粘以下：

```MyBatisTest
public class MyBatisTest {

    public static void main(String[] args) throws Exception {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);

        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        
        // 这里获得的sqlSessionFactory的实例是：DefaultSqlSessionFactory
        // 所以调用的是DefaultSqlSessionFactory的openSession方法
        try (SqlSession session = sqlSessionFactory.openSession()) {
            UserDAO mapper = session.getMapper(UserDAO.class);
            User user = mapper.selectUser(1L);
            System.out.println(user.toString());
        }

    }

}
```

把```sqlSessionFactory.openSession()```方法的源码粘出来：

```sqlSessionFactory.openSession()
  public SqlSession openSession() {
    // configuration.getDefaultExecutorType()获取到的是ExecutorType：SIMPLE, REUSE, BATCH 中的SIMPLE
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
  
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // environment在SqlSessionFactoryBuilder#build中已经解析好了
      final Environment environment = configuration.getEnvironment();
      // 从解析出来的environment中获取transactionFactory
      /**
      * private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
      *     if (environment == null || environment.getTransactionFactory() == null) {
      *         return new ManagedTransactionFactory();
      *     }
      *     // environment中已解析出transactionFactory，这里获取到的transactionFactory是JdbcTransactionFactory实例，因我们在配置文件中配置的就是JDBC
      *     return environment.getTransactionFactory();
      * }
      */
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 构造JdbcTransactionFactory的Transaction
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 获取执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 直接构造SqlSession返回
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

## 构造Executor
org.apache.ibatis.session.Configuration#newExecutor：

```newExecutor(...)
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // 这里的executorType就是SIMPLE
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
      /**
      * SimpleExecutor extends BaseExecutor：
      * public SimpleExecutor(Configuration configuration, Transaction transaction) {
      *    super(configuration, transaction);
      * }
      * // 没做什么事情，简单赋值
      * protected BaseExecutor(Configuration configuration, Transaction transaction) {
      *     this.transaction = transaction;
      *     this.deferredLoads = new ConcurrentLinkedQueue<>();
      *     this.localCache = new PerpetualCache("LocalCache");
      *     this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
      *     this.closed = false;
      *     this.configuration = configuration;
      *     this.wrapper = this;
      * }
      */
    }
    // cacheEnabled默认是true，也就是一级缓存开关啦，默认开启
    if (cacheEnabled) {
      // 包装executor，最终的执行会委托给被包装的executor
      executor = new CachingExecutor(executor);
      /**
      * CachingExecutor implements Executor：
      * public CachingExecutor(Executor delegate) {
      *     this.delegate = delegate;
      *     // 这个this就是CachingExecutor
      *     delegate.setExecutorWrapper(this);
      * }
      */
    }
    // 构造最终的Executor返回
    executor = (Executor) interceptorChain.pluginAll(executor);
    /**
    * org.apache.ibatis.plugin.InterceptorChain#pluginAll
    * 返回的是一个Object，所以需要强转以下
    * public Object pluginAll(Object target) {
    *     for (Interceptor interceptor : interceptors) {
    *         // 调用自定义的interceptor的plugin方法
    *         target = interceptor.plugin(target);
    *     }
    *     // 如果自定义了interceptor，则返回最后执行的interceptor的plugin方法返回的Object对象
    *     // 返回的对象需要被强转成Executor，如果不能被强转，则会报类转换异常
    *     // 如果没有自定义interceptor，则直接返回CachingExecutor
    *     return target;
    * }
    *
    */
    return executor;
  }
```

## 小总结
### ```SqlSession```构造过程：

* 从```configuration```中获取已解析出的```environment```
* 通过```environment```获取```transactionFactory```
* 通过```transactionFactory```获取```Transaction```实例
* 使用```configuration```的```newExecutor(tx, execType)```构造```Executor```实例
  * 根据```executorType```获取```Executor```实例
  * 如果开启```cacheEnabled```，则将```executor```包装为```CachingExecutor```的实例
  * 执行自定义的```plugin```，获取最终的```executor```
* 直接通过```DefaultSqlSession```的构造方法获取```SqlSession```实例

再来张```SqlSession```图，理解相关类之间的依赖关系：
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/sqlsession1.jpg)







