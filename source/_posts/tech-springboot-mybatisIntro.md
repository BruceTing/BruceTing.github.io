---
title: MyBatis基本用法及怎么与Springboot整合
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-10-29 15:30:10
comments: true
tags:
 - mybatis
keywords: mybatis
description: 通过对MyBatis的介绍，了解MyBatis通常的用法及怎么与Springboot进行整合
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/mybatis/mybatisIntro.jpeg
---

## 什么是MyBatis
MyBatis官网给出的定义如下：

```What is MyBatis
MyBatis is a first class persistence framework with support for custom SQL, stored procedures and advanced mappings. MyBatis eliminates almost all of the JDBC code and manual setting of parameters and retrieval of results. MyBatis can use simple XML or Annotations for configuration and map primitives, Map interfaces and Java POJOs (Plain Old Java Objects) to database records.
```
英文不好？没关系，MyBatis给出了多种语言，其中就包括了中文版官方翻译，来看看：

```MyBatis的定义
MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生类型、接口和 Java 的 POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。
```
## 怎么使用呢
我们就不按照官网那样，初始化一个maven项目，然后再手动引入jar包的方式来构建项目了。我们直接使用Idea来初始化一个Springboot项目(目前大家都已在用Springboot开发项目，所以直接使用```Spring Initializr```初始化项目就OK。实在不想用，那就没办法咯！)，熟悉一下Springboot整合中间件的流程。  
打开Idea，依次```File -> new -> Project...``` ， 选择```Spring Initializr```，配置好```JDK```，然后Next，配置好```Project Metadata```，然后next，可以选择```Web -> Spring Web```方便后续做测试，重要的是选择```SQL -> MyBatis Framework 、MySQL Driver```这两项，然后next配置项目名称及存储路径，Finish后呢，项目就建好了，顺带所需要的依赖包也引入到项目了。  
在开始之前呢，需要准备一个测试库（库名按你喜欢的方式命名就好）以及一个测试表（以下测试表为```USER```），准备好之后呢，就可以开始以下骚操作了。

### 使用官网方式
#### 1、添加配置  
在```resources```包下新建```mybatis-config.xml```，然后加入下面模板内容：

```mybatis-config
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  // 环境的名称，可以配置多个环境，区分开发、测试、生产
  <environments default="development">
    // 环境ID
    <environment id="development">
      // 启用事物
      <transactionManager type="JDBC"/>
      // 构建连接池
      <dataSource type="POOLED">
        // MySQL驱动
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        // MySQL数据库链接地址
        <property name="url" value="xxx"/>
        // 用户名
        <property name="username" value="test"/>
        // 密码
        <property name="password" value="test"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="mapper/UserMapper.xml"/>
  </mappers>
</configuration>
```
依次填上你MySQL的相关信息。  
MySQL链接的通常写法：```jdbc:mysql://ip或domain:port/库名?characterEncoding=utf-8&amp;autoReconnect=true&amp;failOverReadOnly=false&amp;serverTimezone=Asia/Shanghai```，在xml中配置的时候，需要把```&```进行转义为```&amp;```否则会报错：  

```error log
Exception in thread "main" org.apache.ibatis.exceptions.PersistenceException: 
### Error building SqlSession.
### Cause: org.apache.ibatis.builder.BuilderException: Error creating document instance.  Cause: org.xml.sax.SAXParseException; lineNumber: 11; columnNumber: 126; 对实体 "autoReconnect" 的引用必须以 ';' 分隔符结尾。
	at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:30)
	at org.apache.ibatis.session.SqlSessionFactoryBuilder.build(SqlSessionFactoryBuilder.java:80)
	at org.apache.ibatis.session.SqlSessionFactoryBuilder.build(SqlSessionFactoryBuilder.java:64)
	at com.springboot.mybatis.MyBatisTest.main(MyBatisTest.java:22)
```

#### 2、构建```User```实体类  
新建包```com.springboot.springbootmybatis.model```，编写实体类```User```：

```User
public class User {

    /**
     * 主键
     */
    private Long id;

    /**
     * 用户ID
     */
    private Long userId;

    /**
     * 用户名
     */
    private String name;

    /**
     * 用户的邮箱
     */
    private String email;

    /**
     * 创建时间
     */
    private Date createdTime;

    /**
     * 修改时间
     */
    private Date modifiedTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public Date getCreatedTime() {
        return createdTime;
    }

    public void setCreatedTime(Date createdTime) {
        this.createdTime = createdTime;
    }

    public Date getModifiedTime() {
        return modifiedTime;
    }

    public void setModifiedTime(Date modifiedTime) {
        this.modifiedTime = modifiedTime;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", userId=" + userId +
                ", name='" + name + '\'' +
                ", email='" + email + '\'' +
                ", createdTime=" + createdTime +
                ", modifiedTime=" + modifiedTime +
                '}';
    }
}
```

#### 3、构建```UserDAO```接口  
新建包```com.springboot.springbootmybatis.dao```，新建接口```UserDAO```：

```UserDAO
@Repository
public interface UserDAO {

    /**
     * 根据主键查询User信息
     * @param id
     * @return
     */
    User selectUser(Long id);

}
```

#### 4、构建```userMapper.xml```文件  
在```resources```下新建```mapper```文件夹，然后新建```userMapper.xml```：

```userUapper.xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.springboot.dao.UserDAO">
    <resultMap id="baseResultMap" type="com.springboot.model.User">
        <id column="ID" jdbcType="BIGINT" property="id"/>
        <result column="USER_ID" jdbcType="BIGINT" property="userId"/>
        <result column="NAME" jdbcType="VARCHAR" property="name"/>
        <result column="EMAIL" jdbcType="VARCHAR" property="email"/>
        <result column="CREATED_TIME" jdbcType="TIMESTAMP" property="createdTime"/>
        <result column="MODIFIED_TIME" jdbcType="TIMESTAMP" property="modifiedTime"/>
    </resultMap>
    
    <sql id="columns">
        ID, USER_ID, NAME, EMAIL, CREATED_TIME, MODIFIED_TIME
    </sql>
    
    <select id="selectUser" resultMap="baseResultMap">
    	select
    		<include refid="columns"/>
    	from USER where id = #{id}
    </select>
</mapper>
```
以上工作准备好之后，我们就可以开始测试了。  
新建包```com.springboot.springbootmybatis.mybatis```，然后新建类```MyBatisTest```，在```main方法```中，引入：

```MyBatis代码
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
运行代码后，就可以看到正常的日志输出啦：

```log
12:27:14.182 [main] DEBUG org.apache.ibatis.logging.LogFactory - Logging initialized using 'class org.apache.ibatis.logging.slf4j.Slf4jImpl' adapter.
12:27:14.216 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - PooledDataSource forcefully closed/removed all connections.
12:27:14.216 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - PooledDataSource forcefully closed/removed all connections.
12:27:14.216 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - PooledDataSource forcefully closed/removed all connections.
12:27:14.216 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - PooledDataSource forcefully closed/removed all connections.
12:27:14.326 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Opening JDBC Connection
12:27:15.277 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - Created connection 1709804316.
12:27:15.277 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65e98b1c]
12:27:15.287 [main] DEBUG com.springboot.dao.UserDAO.selectUser - ==>  Preparing: select ID, USER_ID, NAME, EMAIL, CREATED_TIME, MODIFIED_TIME from USER where id = ? 
12:27:15.321 [main] DEBUG com.springboot.dao.UserDAO.selectUser - ==> Parameters: 1(Long)
12:27:15.348 [main] DEBUG com.springboot.dao.UserDAO.selectUser - <==      Total: 1
User{id=1, userId=28250, name='测试人', email='test@danke.com', createdTime=Tue Oct 29 11:49:44 CST 2019, modifiedTime=Tue Oct 29 11:49:44 CST 2019}
12:27:15.349 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65e98b1c]
12:27:15.352 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65e98b1c]
12:27:15.352 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - Returned connection 1709804316 to pool.
```

到这里，我们可能会去想：上面的```USER实体类、UserDAO、UserMapper.xml```这几个文件，我们都要自己去写吗，一个两个还好，多的话那岂不是很麻烦？非也。官方的大佬已经为我们考虑好了，完全可以使用[MyBatis Generator(MBG)](http://mybatis.org/generator/index.html)来生成嘛，这不就很爽了吗！  

### Springboot自动配置
我们保留```dao、model、mapper```，自动配置的时候，还需要使用，可别删了，要不还得再来一遍。把```mybatis-config.xml```注释掉或改为```mybatis-config.xml.bak```，```MyBatisTest```可以暂时不用管。  

#### 1、数据源及Mapper配置
将```resources/application.properties```文件改为```resources/application.yml```，当然，不改也可以，个人喜欢```yml```方式配置而已。😎  
然后在```application.yml```文件中添加以下配置：

```application.yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: test
    password: test
    url: xxx

mybatis:
  mapper-locations: classpath:/mapper/*.xml

```
注意：在写yml文件的时候，数据库链接url中的```&```不需要转义成```&amp;```

#### 2、构建```Controller```
新建包```com.springboot.springbootmybatis.controller```，新建类```HelloUserController```：

```controller
@RestController
public class HelloUserController {

    @Autowired
    private UserDAO userDAO; //如果在UserDAO上没有@Repository，此处会报红色的下划线

    @GetMapping("/user/{userId}")
    public User getUser(@PathVariable("userId") Long userId) {
        User user = userDAO.selectUser(userId);
        return user;
    }

}
```
这里省略了```service```层，直接在```HelloUserController ```中调用DAO的接口。实际开发中，```Service```层是必不可少的。这里的```@GetMapping 等于 @RequestMapping(method = RequestMethod.GET)```，简单的写法，与之对应的还有```@PostMapping```。

#### 3、添加```Mapper```扫描
在主类```SpringbootMybatisApplication```上添加注解：```@MapperScan(basePackages = "com.springboot.springbootmybatis.dao")```，配置扫描DAO接口的包路径。

```SpringbootMybatisApplication
@SpringBootApplication
@MapperScan(basePackages = "com.springboot.springbootmybatis.dao")
public class SpringbootMybatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootMybatisApplication.class, args);
    }

}
```
注意：添加的包需要和主类```SpringbootMybatisApplication```在同一级目录，否则扫描不到其中的类，之前Springboot的自动配置源码已分析过：```会以主类SpringbootMybatisApplication所在的包为basePackage进行组件扫描```。既然如此，那这里不配置```@MapperScan```行不行呢，当然不行！想想啊，我们面向接口编程，需要使用DAO接口去数据库查询数据，然后返回给调用者，如果接口没有实现类，那怎么去数据库查询呢。而且如果没有实现类，Spring怎么去实例化，怎么去帮助我们管理Bean呢，那不用Spring行不行？行啊，那就使用上面的那种方式，也可以实现，但是不觉得麻烦吗，所有的事情都要自己去做了，比如：Bean的创建及管理，连接的打开关闭等。那这里加了```@MapperScan```后，为什么就可以了呢？预知后事如何，请听下回分解！

#### 4、访问测试
启动应用程序，在```SpringbootMybatisApplication```类中右键``` Run `SpringbootMybatisApplication ` ```，就可以启动啦！在控制台，可以看到以下输出：

```log
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
```
在浏览器中访问：[http://localhost:8080/user/1](http://localhost:8080/user/1)，即可看到结果：

```show
{"id":1,"userId":11111,"name":"测试人","email":"test@163.com","createdTime":"2019-10-29 15:49:44","modifiedTime":"2019-10-29 15:49:44"}
```


