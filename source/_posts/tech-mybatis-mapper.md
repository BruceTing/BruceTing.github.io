---
title: MyBatis源码分析之Mapper
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-11-04 11:30:30
comments: true
tags:
 - mybatis
keywords: mybatis
description: 通过源码分析怎么获取Mapper接口实例
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/mapper.jpeg
---

## 获取Mapper的主流程
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

```xxxDAO或xxxMapper```定义的是一个接口，那么是怎么获取到它的实例的呢。上一章中获取到的```SqlSession ```实例是```DefaultSqlSession```，所以调用的是```DefaultSqlSession```的```getMapper(...)```方法，看看源码：

```session.getMapper(xxx.class)
  // 这是个泛型方法，给它什么类型，那么返回给你的就是什么类型
  public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
  }
```
继续看```Configuration```中的```getMapper(...)```方法：

```configuration.getMapper(type, this)
  // 这不就是从mapperRegistry中获取解析出来的mapper么
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
继续看看```mapperRegistry.getMapper(type, sqlSession)```：

```mapperRegistry.getMapper(type, sqlSession)
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 配置解析的时候，已将解析的Mapper注册到knownMappers中，所以直接从knownMappers中根据类型获取mapperProxyFactory实例
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 根据mapperProxyFactory获取xxxMapper实例，其实拿到的是一个代理类
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
来看看```mapperProxyFactory.newInstance(sqlSession)```：

```mapperProxyFactory.newInstance(sqlSession)
public class MapperProxyFactory<T> {
  // 这不就是定义的xxxMapper或xxxDAO接口吗
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    // Oh，动态代理，是不是很熟悉
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    // 构造一个MapperProxy实例：
    // MapperProxy<T> implements InvocationHandler
    // MapperProxy就是个代理类嘛
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

## 小总结

获取```xxxMapper```的流程：

* 根据```xxxMapper```类型从```configuration```中的mapperRegistry中获取已解析的```mapperProxyFactory```
* 在```mapperProxyFactory```中通过```MapperProxy```使用动态代理获取到```xxxMapper```的代理实例返回





