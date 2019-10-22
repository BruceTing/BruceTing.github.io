---
title: Springboot核心注解
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-10-21 11:00:00
comments: true
tags:
 - springboot
keywords: springboot
description: 通过对springboot核心注解的了解，为学习自动配置原理做下铺垫
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/annotation.jpeg
---

在开始介绍Springboot自动配置之前，来看看Springboot有哪些最核心的注解。
继续[Springboot开胃菜](https://www.yeahme.cn/2019/10/19/tech-springboot-intro/#%E4%BA%94%E3%80%81Develop-Spring-Boot-Aplication-With-IDEA)，打开```SpringbootWebApplication```类后，发现该类上有且只有一个注解```@SpringBootApplication```，来看看这个是注解所包含的意思：
@SpringBootApplication 这是一个便捷的组合注解，使用该注解意味着：

  * 启用配置```@SpringBootConfiguration```
  * 触发自动配置```@EnableAutoConfiguration```
  * 组件扫描```@ComponentScan```
  * 配置属性扫描```@ConfigurationPropertiesScan```

一个个来看看都是啥意思：

  * @SpringBootConfiguration 表示提供Springboot应用所需要的配置，该注解在Springboot应用中可以使用来替代```@Configuration```，这样的话，在自动配置的时候，就可以被自动发现了。Springboot应用有且只能包含一个```@SpringBootConfiguration```注解。
  * @EnableAutoConfiguration 这个是核心中的核心注解啦！使用该注解后，spring应用上下文就会试着去猜测并且配置你所期望的bean。自动配置类是基于你的classpath和你定义了什么样的bean而被应用。比如说：在你的类路径下有```tomcat-embedded.jar```，那就意味着你希望有一个```TomcatServletWebServerFactory```，除非你自己定义了```TomcatServletWebServerFactory```bean。使用```@SpringBootApplication```意味着自动配置自动生效，如果有你不需要的配置，可以使用```#excludeName()```进行排除。自动配置的类通常都是Spring```@Configuration```的bean，这些bean通过使用```SpringFactoriesLoader```机制定位。通常自动配置类都是```@Conditional```的beans(大多数使用```@ConditionalOnClass```和```@ConditionalOnMissingBean```注解)。在注解上包含了两个注解：
   * AutoConfigurationPackage 存储自动配置的包，在该包下的bean都会被注册
     * @Import(AutoConfigurationPackages.Registrar.class) 通过这个注解去存储基础包及注册bean的定义
   * @Import(AutoConfigurationImportSelector.class) 这就是自动配置的核心啦，所有的自动配置类通过这个导入进来，那么它是在哪里被调用，以及自动配置bean怎么配置进来的，留待后面分析。
  * @ComponentScan 这个是组件扫描，可以配置组件扫描的基础包以及过滤器
  * @ConfigurationPropertiesScan 指定扫描```@ConfigurationProperties```类的基础包
    * @Import(ConfigurationPropertiesScanRegistrar.class) 通过扫描使用```ImportBeanDefinitionRegistrar```注册```@ConfigurationPropertie```bean定义
    * @EnableConfigurationProperties 对带有```@ConfigurationProperties```注解的bean都可以使用标准的方式被注册，比如说使用```@Bean```方式
      * @Import(EnableConfigurationPropertiesRegistrar.class) 对带有```@EnableConfigurationProperties```的类使用```ImportBeanDefinitionRegistrar```进行注册

这些注解在项目里都可以点进去看看，熟悉一下，对自动配置的解析有帮助，要不然会感到有点懵！最后，我们再用一个图来总结下，标红的是核心中的核心，需要重点关注的地方：
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/annotation1.png)


