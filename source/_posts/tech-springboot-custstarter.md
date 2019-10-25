---
title: 自定义Springboot的Starter
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-10-24 18:30:00
comments: true
tags:
 - springboot
keywords: springboot
description: 学习完Springboot的自动配置原理后，实战怎么自定义Springboot的Starter
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/custstarter.jpeg
---


理解了Springboot自动配置原理后呢，来看看怎么自定义一个starter。  
Starter的命名也是有一定的规范的，官方说的是：spring-boot-starter-xxx被官方保留使用啦，第三方自定义就使用xxx-spring-boot-starter。

### 自定义Springboot Starter
怎么初始化一个项目，请看[这里吧](https://www.yeahme.cn/2019/10/19/tech-springboot-introduction/#五、Develop-Spring-Boot-Aplication-With-IDEA)！   
1、使用Idea初始化一个没有整合任何中间件的项目，也就是说在使用工具中的```Spring Initializr```初始化项目的时候，在选择中间件依赖的时候，啥也不选直接next。这里就初始化一个叫```hello-spring-boot-starter```的项目  

2、在初始化出来的项目中，把生成的包及其中的类删掉，然后新建自己的包。  

3、新建```com.springboot.hello.marker```包，这个包用于条件注解用，也就是说，自定义的自动配置必须在有这个类的前提下，自动配置才能生效。

```Marker
public class Marker {
}
```

4、新建```com.springboot.hello.service```包，用于写业务逻辑：

```HelloStarterService
public class HelloStarterService {

    public String helloStarter() {
        return "WOW！Hello starter is working...";
    }

}
```

5、新建	```com.springboot.hello.autoconfig```包，在其中编写自动配置类：

```HelloStarterAutoConfiguration
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Marker.class)
public class HelloStarterAutoConfiguration {

    @Bean
    public HelloStarterService getHelloStarterService() {
        return new HelloStarterService();
    }

}
```
如果业务逻辑中需要属性配置，还可以在类上加上```@EnableConfigurationProperties({xxxProperties.class})```，也可以在类上使用```@EnableConfigurationProperties```启用属性配置后，在方法上使用```    @ConfigurationProperties(prefix = "xxx.xxx")```进行配置，也可以加上其他的条件注解```@ConditionalXXX```进行更复杂的配置。  

6、在```resources```包下新建```META-INF/spring.factories```文件，在其中写上自动配置类：

```auto configure
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.springboot.hello.autoconfig.HelloStarterAutoConfiguration
```

7、在打包之前删除```pom.xml```中的这段配置，否则回报```找不到main类的错误```：

```pom.xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

8、在我们之前的```springboot-web```项目的```pom.xml```中引入这个starter：

```starter
<dependency>
    <groupId>com.springboot</groupId>
    <artifactId>hello-spring-boot-starter</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```
然后我们在项目的```application.yml```文件中打开```debug```：

```debug
server:
  port: 8081
debug: true
```
上面的端口号，是之前改的，后面访问的时候就使用这个端口！  

9、在```Helloworld.java```中加入在自动配置类中写的```HelloStarterService```，这个地方不需要去实例化，就可以拿到接口```HelloStarterService```的实例，自动配置已经帮我们搞好了，直接用就行了。

```Helloworld
@RestController
public class Helloworld {

    @Autowired
    private HelloStarterService helloStarterService;

    @RequestMapping
    public String helloworld() {
        return "hello world";
    }

    @RequestMapping("/helloStarter")
    public String helloStarter() {
        return helloStarterService.helloStarter();
    }

}
```
启动一下，见证奇迹的时刻。  
在启动后的日志中发现，自定义的自动配置类已经生效了：

```log
============================
CONDITIONS EVALUATION REPORT
============================


Positive matches:
-----------------
	...
	...
	HelloStarterAutoConfiguration matched:
      - @ConditionalOnClass found required class 'com.springboot.hello.marker.Marker' (OnClassCondition)
   ...
   ...
   
Negative matches:
-----------------

   ActiveMQAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory' (OnClassCondition)

   ...
   ...
```
```Positive matches```是自动配置生效的，```Negative matches```是自动配置没有生效的。
如果我们的自动配置没有生效会不会出现在```Negative matches```中呢，把自动配置中的条件稍微改一下，来看看：

```HelloStarterAutoConfiguration
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(Marker.class)
public class HelloStarterAutoConfiguration {

    @Bean
    public HelloStarterService getHelloStarterService() {
        return new HelloStarterService();
    }

}
```
把条件改成了```@ConditionalOnBean(Marker.class)```，由于在Spring context环境中是不存在```Marker```类实例的，所以自动配置是不会生效的，重新打包，```reimport```一下maven依赖，然后启动```springboot-web```项目后，会看到这样的输出日志：

```log

***************************
APPLICATION FAILED TO START
***************************

Description:

Field helloStarterService in com.springboot.springbootweb.controller.Helloworld required a bean of type 'com.springboot.hello.service.HelloStarterService' that could not be found.

The injection point has the following annotations:
	- @org.springframework.beans.factory.annotation.Autowired(required=true)

The following candidates were found but could not be injected:
	- Bean method 'getHelloStarterService' in 'HelloStarterAutoConfiguration' not loaded because @ConditionalOnBean (types: com.springboot.hello.marker.Marker; SearchStrategy: all) did not find any beans of type com.springboot.hello.marker.Marker


Action:

Consider revisiting the entries above or defining a bean of type 'com.springboot.hello.service.HelloStarterService' in your configuration.
```
自动配置失效，属性注入失败，应用启动失败！把```helloStarterService```这个属性注释掉：

```Helloworld
@RestController
public class Helloworld {

//    @Autowired
//    private HelloStarterService helloStarterService;

    @RequestMapping
    public String helloworld() {
        return "hello world";
    }

    @RequestMapping("/helloStarter")
    public String helloStarter() {
//        return helloStarterService.helloStarter();
        return null;
    }

}
```
再启动下，服务正常启动后看看日志：

```log
============================
CONDITIONS EVALUATION REPORT
============================


Positive matches:
-----------------

   DispatcherServletAutoConfiguration matched:
      - @ConditionalOnClass found required class 'org.springframework.web.servlet.DispatcherServlet' (OnClassCondition)
      - found 'session' scope (OnWebApplicationCondition)
   ...
   ...
   
Negative matches:
-----------------

   ...
   ...
   HelloStarterAutoConfiguration:
      Did not match:
         - @ConditionalOnBean (types: com.springboot.hello.marker.Marker; SearchStrategy: all) did not find any beans of type com.springboot.hello.marker.Marker (OnBeanCondition)

   ...
   ...
```
预料之中，对吧！通个这个日志就可以知道自定义的自动配置类有没有生效啦！  
OK！我们去掉```helloStarterService```属性及方法中的注释，将```Starter```中的```@ConditionalOnBean```改回```@ConditionalOnClass```，重新打包后，在项目中```reimport```一下项目依赖，让项目重新正常启动起来。  

10、在浏览器中输入：```http://localhost:8081/helloStarter```后，可以看到：

```res
WOW！hello starter is working...
```
到此呢，自定义Springboot Starter就搞定了。  
再来总结下步骤：

* 初始化一个starter项目
* 编写业务逻辑
* 编写自动配置类及配置相应的配置
* 将自动配置类加入到```META-INF/spring.factories```文件中
* 打包starter
* 在项目中引用








