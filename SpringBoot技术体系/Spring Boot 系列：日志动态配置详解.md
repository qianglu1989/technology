- ## 一、简介

  Spring Boot 版本： 2.3.4.RELEASE

  不知道大家有没有过当线上出现问题的时候，需要某些DEBUG日志，但奈何当前使用时INFO。

  如果想启用DEBUG就需要重新打包发版，但某些场景下重启有可能问题就不会复现了，真是脑阔疼啊。

  今天我们就来说下Spring Boot 下的日志配置动态调整，让你的日志级别随心而动。

  

  #### Spring Boot的日志


  ![](https://imgkr2.cn-bj.ufileos.com/277c9024-b1dd-4039-ab29-b79a2fdd1040.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=U67F86JUsNuQL4%252FZmEKXi7%252FRi6I%253D&Expires=1602927530)


  在Spring Boot 内部使用的其实是Commons Logging, 而基于Spring Boot的配置加载机制为我们提供了Java Util Logging、Log4j2、Logback几种日志方式。

  Logback是其默认的日志框架，如果没有特殊的必要真不建议更换了。（不要说性能了）

  

  #### 日志格式

  不要小瞧格式这种东西，在实际应用的时候是贼拉重要的一件事。

  不知道大家的公司有没有统一的日志基础组件，当然没有也大概会有统一的日志配置文件吧。

  想想如果你的日志格式不统一的话，如果每个项目都有自己的风格的话，你叫你的运维小伙伴怎么帮你切分日志？帮你报警呢？那真是正则写到死，完全靠爱发电了。（比如我们使用的Loghub 不统一要被运维打死的）

  

  来看下我们的日志格式配置，这里只放PATTERN

  ```
  -|%d{yyyy-MM-dd HH:mm:ss.SSS}|%-5level|%X{tid}|%thread|%logger{36}.%M:%L-%msg%n
  ```

  先解释下各个位置：

  - `%d{yyyy-MM-dd HH:mm:ss.SSS} `：时间
  - `%-5level` ： 日志级别
  - `%X{tid} `： 我们自定义的分布式追踪ID
  - `%thread `： 线程
  - `%logger{36}.%M:%L` ：class的全名（36代表最长字符）.信息 行号
  - `%msg%n` ： 输出信息 换行

  

  **这里不知道大家能否理解前面有一个 `-|`是为了什么？ 其实是为了在正则切分的时候方便区分换行的日志，如异常堆栈的信息。**

  

  #### 几个知识点

  再说几个其他Spring Boot使用的小点，我们就来进入正题

  - 在这里Logback 没有FATAL 级别，被归到ERROR里面了

  - 可以在`application.properties `里面配置 `debug=true`来开启debug模式，你也可以配置`trace=true` 开启 trace模式
  - 可以再`application.properties`里使用`logging.level.<logger-name>=<level>`这种格式来配置各种日志级别，比如`org.hibernate`级别被设置为ERROR `logging.level.org.hibernate=error`
  - 日志组的概念，如果这么一个个配置烦死了，可以设定一个组给它整体配置。如：`logging.group.tomcat=org.apache.catalina, org.apache.coyote`配置级别为TRACE `logging.level.tomcat=TRACE`
  - 如果使用Logback配置文件，官方建议使用`logback-spring.xml`这样的名称，如果使用`logback.xml`这样的，Spring的日志初始化就该有问题了。
  - 如果你控制日志配置，但又不想用logback.xml作为Logback配置的名字，可以在application.properties配置文件里面通过logging.config属性指定自定义的名字：`logging.config=classpath:xxx-log.xml`

  

  ## 二、动态修改日志级别

  下面我们就来说说在运行状态下的Spring Boot应用是怎么进行动态日志级别变更的

  

  #### Spring Boot Actuator

  Actuator 想必了解过Spring Boot的都知道它的大名，监控、审计、度量信息等等。我们使用的优雅停机、健康检查、日志信息都是通过这玩意来实现的。

  

  在我们使用了Actuator 后，我们就可以使用其Logger的REST接口来操作我们的日志了，有如下三个

  - `GET http://127.0.0.1:6080/actuator/loggers`  返回当前应用全部的日志级别信息（想要啥有啥）
  - `GET  http://127.0.0.1:6080/actuator/loggers/{name}` 返回{name}的日志级别
  - `POST  http://127.0.0.1:6080/actuator/loggers/{name}`  配置参数`{"configuredLevel":"INFO","effectiveLevel":"INFO"}` 修改日志级别

  

  #### 使用Actuator 机制动态修改级别

  

  **1）、依赖必要的配置 Spring Boot Actuator (如果你继承了parent是不需要version的)**

  ```
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
      <version>2.3.4.RELEASE</version>
  </dependency>
  ```

  

  **2）、配置文件**

  ```
  #注意，这里我只开启了loggers,health两个Actuator
  management.endpoints.web.exposure.include=loggers,health
  management.endpoint.loggers.enabled=true
  
  ```

  

  **3）、创建一个Controller**

  ```
  @RestController
  @RequestMapping("/log")
  public class TestLogController {
  
      private Logger log = LoggerFactory.getLogger(TestLogController.class);
  
      @GetMapping
      public String log() {
          log.trace("This is a TRACE level message");
          log.debug("This is a DEBUG level message");
          log.info("This is an INFO level message");
          log.warn("This is a WARN level message");
          log.error("This is an ERROR level message");
          return "See the log for details";
      }
  }
  ```

  

  **4）、启动应用**

  此时在你启动应用后，如果你使用IDEA，就可以看到如下Mapping,有三个关于logger的REST接口

   

  ![](https://imgkr2.cn-bj.ufileos.com/9edbdf72-3ed7-459b-9338-1e13a0214743.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Ncxd8co1Rd6umqCt7jMXl3T8GYU%253D&Expires=1602927762)

  

  **5）、测试**

  此时，我先执行`GET http://127.0.0.1:6080/log`,控制台打印如下信息

  ```
  2020-10-15 23:14:51.204  INFO 52628 --- [nio-6080-exec-7] c.q.chapter1.failure.TestLogController   : This is an INFO level message
  2020-10-15 23:14:51.220  WARN 52628 --- [nio-6080-exec-7] c.q.chapter1.failure.TestLogController   : This is a WARN level message
  2020-10-15 23:14:51.221 ERROR 52628 --- [nio-6080-exec-7] c.q.chapter1.failure.TestLogController   : This is an ERROR level message
  
  ```

  此时我们执行`GET http://127.0.0.1:6080/actuator/loggers/ROOT`返回`{"configuredLevel":"INFO","effectiveLevel":"INFO"}`，代表此时我们应用的日志级别是INFO，ROOT表示的是根节点。

  

  **6)、修改级别**

  此时我们不想再看INFO信息，想将整个应用日志级别换位WARN的话，我们来执行`POST  http://127.0.0.1:6080/actuator/loggers/ROOT` 参数为`{"configuredLevel":"TRACE","effectiveLevel":"TRACE"}`

  

  此时我们再来执行`GET http://127.0.0.1:6080/log`,控制台打印如下信息

  ```
  2020-10-15 23:24:11.481 TRACE 53552 --- [nio-6080-exec-3] c.q.chapter1.failure.TestLogController   : This is a TRACE level message
  2020-10-15 23:24:11.481 DEBUG 53552 --- [nio-6080-exec-3] c.q.chapter1.failure.TestLogController   : This is a DEBUG level message
  2020-10-15 23:24:11.481  INFO 53552 --- [nio-6080-exec-3] c.q.chapter1.failure.TestLogController   : This is an INFO level message
  2020-10-15 23:24:11.481  WARN 53552 --- [nio-6080-exec-3] c.q.chapter1.failure.TestLogController   : This is a WARN level message
  2020-10-15 23:24:11.481 ERROR 53552 --- [nio-6080-exec-3] c.q.chapter1.failure.TestLogController   : This is an ERROR level message
  ```

  

  **另几种方法**

  - 配置文件扫描: 就是Logback 自动扫描的特性修改级别，配合`logback-spring.xml` 开启`<configuration scan="true" scanPeriod="15 seconds">`就可以实现动态修改`logback-spring.xml`内部日志级别的目的了，有兴趣可以自己试试。

  - arthas 动态修改
  - 结合远程配置中心，如Apollo实现级别动态修改

  

  ## 三、实现原理

  这里我们主要使用的是`Spring Boot Actuator Log` ,所以我们也就来说说它的原理。

  

  #### Endpoint的加载

  首先我们从依赖`spring-boot-actuator`中找到我们的`LoggersEndpoint`（所有的Actuator都是这一个路数），如图：


  ![](https://imgkr2.cn-bj.ufileos.com/862ffadd-6fab-4a01-a7c7-91612ff96680.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=xfxJ8hzx5dmafUcTraFcUt05oVM%253D&Expires=1602927794)

  

  熟悉Spring Boot加载机制的朋友都了解，在每个actuator Endpoint的背后，必然还会存在一个`xxxEndpointAutoConfiguration` 来为我们进行Endpoint 的加载。

  

  而这些加载机制就都存放在`spring-boot-actuator-autoconfigure`中，我们在其中可以找到`LoggersEndpointAutoConfiguration`用于加载`LoggersEndpoint`的配置类。

  

  来看下它的核心：

  ```java
  @Bean
  	@ConditionalOnBean(LoggingSystem.class)
  	@Conditional(OnEnabledLoggingSystemCondition.class)
  	@ConditionalOnMissingBean
  	public LoggersEndpoint loggersEndpoint(LoggingSystem loggingSystem,
  			ObjectProvider<LoggerGroups> springBootLoggerGroups) {
  		return new LoggersEndpoint(loggingSystem, springBootLoggerGroups.getIfAvailable(LoggerGroups::new));
  	}
  ```

  可以看到两个重要的参数：

  - LoggingSystem：一个抽象顶级类
  - springBootLoggerGroups：存储了当前日志分组数据

  

  **总结下：**

  1、我们依赖了`spring-boot-starter-actuator `包后，里面依赖了`spring-boot-actuator-autoconfigure`

  2、在启动扫描到`spring-boot-actuator-autoconfigure` 下的`META-INF/spring.factories` 时，`LoggersEndpointAutoConfiguration`会被加载到

  3、`LoggersEndpointAutoConfiguration` 内又声明了`LoggersEndpoin` 并赋值`LoggingSystem` 和`springBootLoggerGroups` 作为其参数

  4、项目启动后我们通过`LoggersEndpoint` 接口进行日志数据访问

  

  #### LoggingSystem

  

  **LoggingSystem的继承关系**

  通过上面可以了解到 `LoggingSystem`就是日志操作管理的核心了，所以我们先来看下他的Diagrams

  

  ![](https://imgkr2.cn-bj.ufileos.com/fe55169b-11e7-4fbe-952b-710a7f1295e3.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=ygdGx9sEB%252BbbEBCJi30D0%252FIi4fA%253D&Expires=1602927813)


  通过继承关系一眼可以看到，`LogbackLoggingSystem`就是我们的正主了,虽然说知道了正主那我们的LoggingSystem到底是怎么加载的呢？

  

  **LoggingSystem的加载**

  主要参与类说下：

  - `LoggingApplicationListener`
  - `ApplicationStartingEvent`
  - `LogbackLoggingSystem`
  - `LoggingSystem`

  

  不多说，先上图


  ![](https://imgkr2.cn-bj.ufileos.com/3da9d3fc-cd38-4259-a54a-f1ec6f33ae36.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=pbBoSG5oLOgcVGZfKdhPvq9OPUk%253D&Expires=1602927822)


  1、应用使用`SpringApplication.run(Chapter1Application.class, args);`启动

  2、发送启动事件`ApplicationStartingEvent`、`ApplicationEnvironmentPreparedEvent`等

  3、`LoggingApplicationListener`接收事件进行事件分发

  4、`LoggingApplicationListener`接收事件`ApplicationStartingEvent`

  5、`LoggingApplicationListener`调用内部`onApplicationStartingEvent(ApplicationStartingEvent event) `方法，使用LoggingSystem.get(classloader)初始化LoggingSystem。

  6、然后调用`LogbackLoggingSystem.beforeInitialize()`(因为这里我们用的是logback)

  7、`LoggingApplicationListener`接收事件`ApplicationEnvironmentPreparedEvent`

  8、`LoggingApplicationListener`调用内部`onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) `方法

  9、`onApplicationEnvironmentPreparedEvent`调用`initialize(ConfigurableEnvironment environment, ClassLoader classLoader)`方法，会根据环境变量配置进行logging system的初始化

  

  #### LoggersEndpoint

  

  最后，我们就来看看LoggersEndpoint就可以了,这里为了方便解读我就不全拷贝过来了，需要哪里选哪里

  

  首先看我们对日志操作的三个接口：

  - `GET /actuator/loggers`   对应  `public Map<String, Object> loggers() `方法

  - `GET /actuator/loggers/{name} `  对应 `public LoggerLevels loggerLevels(@Selector String name) `方法
  - `POST /actuator/loggers/{name}`   对应 `public void configureLogLevel(@Selector String name, @Nullable LogLevel configuredLevel) ` 方法

  这里拿一个`public Map<String, Object> loggers() `方法举例说明，其他的差不多

  

  ```java
  @ReadOperation
  	public LoggerLevels loggerLevels(@Selector String name) {
  		Assert.notNull(name, "Name must not be null");
  		LoggerGroup group = this.loggerGroups.get(name);
  		if (group != null) {
  			return new GroupLoggerLevels(group.getConfiguredLevel(), group.getMembers());
  		}
  		LoggerConfiguration configuration = this.loggingSystem.getLoggerConfiguration(name);
  		return (configuration != null) ? new SingleLoggerLevels(configuration) : null;
  	}
  ```

  

  代码其实很简单，就不做过多解读了。

  

  

  ## 四、絮叨

  

  其实日志在我们的系统应用中很重要，对于问题的排查也是重要的凭证。根据经验我们系统的日志最好能做到几个点：

  - 日志格式统一化

  - 最好提供统一的日志组件，比如我们就使用了公共的logback-spring.xml组件。如果我们需要修改日志某些特性，比如加APM日志等等，只需要改一个点，我们系统的所以日志状态都会发生改变。

  - 最好使用动态配置，调整日志配置，比如Apollo

    