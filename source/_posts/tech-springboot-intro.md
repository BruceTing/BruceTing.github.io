---
title: Springboot开篇
author: 多多
avatar: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/base/avatar.jpg
authorLink:
authorAbout: 一个好奇的人
authorDesc: 一个好奇的人
categories: tech
date: 2019-10-19 15:00:00
comments: true
tags:
 - springboot
keywords: springboot
description: springboot的基本介绍及应用
photos: https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro.jpeg
---

我们在学习某一知识之前，总得清楚它是**干什么的（What）**，**怎么干（How）**，**为什么这么干（Why）**，对它发出来自灵魂的三拷问。  
来看看官网介绍：  
## 一、Introducing Spring Boot
```springboot
Spring Boot makes it easy to create stand-alone, production-grade Spring-based Applications that you can run. We take an opinionated view of the Spring platform and third-party libraries, so that you can get started with minimum fuss. Most Spring Boot applications need very little Spring configuration.  

You can use Spring Boot to create Java applications that can be started by using java -jar or more traditional war deployments. We also provide a command line tool that runs “spring scripts”.  

Our primary goals are:
  Provide a radically faster and widely accessible getting-started experience for all Spring development.
  Be opinionated out of the box but get out of the way quickly as requirements start to diverge from the defaults.
  Provide a range of non-functional features that are common to large classes of projects (such as embedded servers, security, metrics, health checks, and externalized configuration).
  Absolutely no code generation and no requirement for XML configuration.  
```

明白没？不明白？那我就用我这三级都没过的英语水平简单的翻译下：
```translate
Spring boot 使得创建以及运行基于单机版或生产级别得spring应用变得很简单。我们提出了spring平台和第三库的观点，使得你可以以最小得代价去开始干你想干得事情。而且对于大部分Spring Boot应用程序仅仅需要很少的配置就可以运行。  

你可以使用Spring Boot去创建Java引用，并且以java -jar或是传统部署war包的方式运行。我们也提供了在命令行以"spring scripts"的方式运行  

我们的重要目标是：  
  对于所有的Spring开发从根本上提供一个更快速且可广泛地容易访问的入门体验
  开箱即用
  给一系列大型项目提供通用的非功能性特性（例如嵌入式服务器、安全性、指标、运行状况检查和外部化配置）
  绝对的无需代码生成和XML配置
```

还没明白？自行Google去吧！！！下面可以不用翻译了。

## 二、System Requirements
Spring Boot 2.2.0.RELEASE requires [Java 8](https://www.java.com) and is compatible up to Java 13 (included). [Spring Framework 5.2.0.RELEASE](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/) or above is also required.

Explicit build support is provided for the following build tools:

|Build Tool    | Version      |
| :-------------: | :-------------: |
|Maven         | 3.3+         |
|Gradle        | 5.x (4.10 is also supported but in a deprecated form)|

Spring Boot supports the following embedded servlet containers:

|Name          | Servlet Version|
| :-------------: | :-------------: |
|Tomcat 9.0    | 4.0            |
|Jetty 9.4     | 3.1            |
|Undertow 2.0  | 4.0            |

You can also deploy Spring Boot applications to any Servlet 3.1+ compatible container.

## 三、Installing Spring Boot
### 1、Installation Instructions for the Java Developer
对于开发者来说的话，主要有下面两种方式：
**a、Maven Installation**
使用maven前，需要先配置好[maven](https://maven.apache.org)的环境。典型的说，你的**Maven POM**文件会继承```spring-boot-starter-parent``` project并且会声明一个或者多个starters。  
一个典型的```pom.xml```文件：

```pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>myproject</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <!-- Inherit defaults from Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.0.RELEASE</version>
    </parent>

    <!-- Add typical dependencies for a web application -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <!-- Package as an executable jar -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
**b、Gradle Installation**
[Gradle](https://gradle.org)没用过，自行看文档吧！

### 1、Installing the Spring Boot CLI
The Spring Boot CLI (Command Line Interface) is a command line tool that you can use to quickly prototype with Spring. It lets you run Groovy scripts, which means that you have a familiar Java-like syntax without so much boilerplate code.

You do not need to use the CLI to work with Spring Boot, but it is definitely the quickest way to get a Spring application off the ground.

`Spring Boot CLI 是一个命令行工具，在我们使用Spring Boot的工作中并不需要使用CLI。`

## 四、Developing Your First Spring Boot Application
This section describes how to develop a simple “Hello World!” web application that highlights some of Spring Boot’s key features. We use Maven to build this project, since most IDEs support it.
`这个部分演示怎么使用maven部署一个简单的"Hello World"web应用，大部分IDEs都支持使用maven`

Before we begin, open a terminal and run the following commands to ensure that you have valid versions of Java and Maven installed:
`开始之前要确认Java和Maven已安装好`

```Java
$ java -version
java version "1.8.0_102"
Java(TM) SE Runtime Environment (build 1.8.0_102-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.102-b14, mixed mode)
```
```maven
$ mvn -v
Apache Maven 3.5.4 (1edded0938998edf8bf061f1ceb3cfdeccf443fe; 2018-06-17T14:33:14-04:00)
Maven home: /usr/local/Cellar/maven/3.3.9/libexec
Java version: 1.8.0_102, vendor: Oracle Corporation
```
This sample needs to be created in its own folder. Subsequent instructions assume that you have created a suitable folder and that it is your current directory.
`也就是说，需要创建一个文件夹，并且当前的路径是在你创建的文件下`
	
### 1、Creating the POM
We need to start by creating a Maven pom.xml file. The pom.xml is the recipe that is used to build your project. Open your favorite text editor and add the following:
`使用喜爱的编辑工具创建一个pom.xml文件`

```pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>myproject</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.0.RELEASE</version>
    </parent>

    <!-- Additional lines to be added here... -->
</project>
```
### 2、Adding Classpath Dependencies
Spring Boot提供了很多的"Starters"，让你可以把jars添加到classpath下。我们在POM中的parent部分添加了```spring-boot-starter-parent```，```spring-boot-starter-parent```是一个特殊的starter，它提供了```dependency-management```，因此我们在依赖中可以踢出```version```标签。  
这个地方我们以添加```spring-boot-starter-web```为列：

```web dependency
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```
### 3、Writing the Code
To finish our application, we need to create a single Java file. By default, Maven compiles sources from src/main/java, so you need to create that folder structure and then add a file named src/main/java/Example.java to contain the following code:
`maven默认是在src/main/java/下编译源文件的，因此需要将Example.java类文件添加到src/main/java/目录下`

```Example.java
import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.web.bind.annotation.*;

@RestController
@EnableAutoConfiguration
public class Example {

    @RequestMapping("/")
    String home() {
        return "Hello World!";
    }

    public static void main(String[] args) {
        SpringApplication.run(Example.class, args);
    }
}

```
```@RestController```和```@RequestMapping```是SpringMVC的注解！
```@EnableAutoConfiguration```这是一个对于Spring Boot来说，很核心的一个注解，我们后面再说。

### 4、Running the Example
At this point, your application should work. Since you used the spring-boot-starter-parent POM, you have a useful run goal that you can use to start the application. Type ```mvn spring-boot:run``` from the root project directory to start the application. You should see output similar to the following:
`在项目根目录执行mvn spring-boot:run，就可以看到输出日志`

```log
$ mvn spring-boot:run

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::  (v2.2.0.RELEASE)
....... . . .
....... . . . (log output here)
....... . . .
........ Started Example in 2.222 seconds (JVM running for 6.514)
```
If you open a web browser to [localhost:8080](localhost:8080), you should see the following output:

```text
Hello World!
```
To gracefully exit the application, press ```ctrl-c```。

### 5、Creating an Executable Jar
To create an executable jar, we need to add the ```spring-boot-maven-plugin``` to our ```pom.xml```. To do so, insert the following lines just below the dependencies section:
`在pom文件中添加如下依赖`

```maven plugin
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
Notice：The ```spring-boot-starter-parent``` POM includes ```<executions>```configuration to bind the ```repackage``` goal. If you do not use the parent POM, you need to declare this configuration yourself. See the [plugin documentation](https://docs.spring.io/spring-boot/docs/2.2.0.RELEASE/maven-plugin//usage.html) for details.
`spring-boot-starter-parent包含了<executions>并且绑定了repackage构建，因此我们直接加上上面的依赖就可以了。如果你没有使用parent Pom的话呢，就需要自己声明构建配置了`

Save your ```pom.xml``` and run ```mvn package``` from the command line, as follows:

```plugin log
$ mvn package

[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building myproject 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] .... ..
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ myproject ---
[INFO] Building jar: /Users/developer/example/spring-boot-example/target/myproject-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.2.0.RELEASE:repackage (default) @ myproject ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
If you look in the ```target``` directory, you should see ```myproject-0.0.1-SNAPSHOT.jar```. The file should be around 10 MB in size. If you want to peek inside, you can use ```jar tvf```, as follows:
`在target目录中可以看到myproject-0.0.1-SNAPSHOT.jar文件，使用jar tvf可以查看文件内部的详细信息`

```detail
$ jar tvf target/myproject-0.0.1-SNAPSHOT.jar
```
You should also see a much smaller file named ```myproject-0.0.1-SNAPSHOT.jar.original``` in the ```target``` directory. This is the original jar file that Maven created before it was repackaged by Spring Boot.
`在target目录中还可以看到一个myproject-0.0.1-SNAPSHOT.jar.original文件，它是maven在使用Spring Boot打包之前创建的jar源文件`

To run that application, use the java -jar command, as follows:

```run log
$ java -jar target/myproject-0.0.1-SNAPSHOT.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::  (v2.2.0.RELEASE)
....... . . .
....... . . . (log output here)
....... . . .
........ Started Example in 2.536 seconds (JVM running for 2.864)
```
As before, to exit the application, press ctrl-c.

到此为止呢，Spring Boot的灵魂三问已经清楚了。但是发现一个问题，手动去写POM，然后创建类文件，这么简单工程还好，那么复杂的工程的呢，这个过程就很痛苦了啊。那么我们来看看使用工具怎么创建Spring Boot工程，这里使用Idea为例，其他的IDEs类似，就不用说了哈！

## 五、Develop Spring Boot Aplication With IDEA
### 1、初始化Spring Boot项目
打开Idea，左上角依次：file -> new -> project  
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro1.jpg)  
在弹出框中左侧菜单中选择Spring Initializr后，在右侧配置SDK版本，下面选择Default就行了，然后next  
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro2.jpg)  
配置project，然后next  
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro3.jpg)  
选择需要整合的web中间件，这里有很多的中间可选，可以都看看，后面将要介绍的一些中间件也在里面（如：eureka），然后next  
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro4.jpg)  
配置项目存储路径，Finish后，来看下目录结构：  
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro5.jpg)  
到这里，你就可以直接启动了：

```started log
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.0.RELEASE)

2019-10-19 17:27:49.228  INFO 33267 --- [           main] c.s.s.SpringbootWebApplication           : Starting SpringbootWebApplication on bogon with PID 33267 (/Users/dingzhongshen/SelfProjects/springblog/springboot-web/target/classes started by dingzhongshen in /Users/dingzhongshen/SelfProjects/springblog/springboot-web)
2019-10-19 17:27:49.232  INFO 33267 --- [           main] c.s.s.SpringbootWebApplication           : No active profile set, falling back to default profiles: default
2019-10-19 17:27:50.389  INFO 33267 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2019-10-19 17:27:50.398  INFO 33267 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2019-10-19 17:27:50.399  INFO 33267 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.27]
2019-10-19 17:27:50.469  INFO 33267 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2019-10-19 17:27:50.469  INFO 33267 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1150 ms
2019-10-19 17:27:50.631  INFO 33267 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2019-10-19 17:27:50.816  INFO 33267 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2019-10-19 17:27:50.819  INFO 33267 --- [           main] c.s.s.SpringbootWebApplication           : Started SpringbootWebApplication in 2.547 seconds (JVM running for 5.328)
```
看到上面日志```Tomcat started on port(s): 8080```，就表示Spring Boot项目已经初始化并启动成功了。是不是很简单，开箱即用嘛！！！  
当然啦，我们要做的事情，可不止这么多，接下来写个Controller试试。

### 2、配置Spring Boot
在```resources```目录下，有个```application.properties```文件，我喜欢改成```application.yml```。然后就可以在这个文件中配置需要的属性啦，比如将服务端口号改成8081：

```application.yml
server:
  port: 8081
```
启动应用之后，就可以在日志里看到端口被改成：

```port
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
```
```context path```也是可以进行配置的。在使用spring boot的时候，需要集成一些其他的中间件，它们的属性都配置在这里，那他们有什么样的属性，怎么看呢？别急，后面介绍！

### 3、编写逻辑，加入相应的启动注解
![](https://cdn.jsdelivr.net/gh/bruceting/blogcdn/img/article/tech/spring/springboot/springboot-intro6.jpg)  
启动之后，在浏览器流输入[http://localhost:8080/](http://localhost:8080/)，就可以看到效果啦，是不是很酷：

```text
Hello World!
```


Bingo！！！恭喜你，已经初步掌握Spring Boot啦！上面已经包含了springboot整合中间件的过程，不知道你发现没，没有？不急，后面再说！现在你就只需要知道，使用Spring Boot无非就这么 ***三板斧*** 就行了！





