> 爱生活，爱编码，微信搜一搜【**架构技术专栏**】关注这个喜欢分享的地方。本文 [架构技术专栏](http://47.104.79.116:4321/) 已收录，有各种视频、资料以及技术文章。



#### 一、概述

目前Spring Boot已经发展到2.3.4.RELEASE ,对于它的好处网上也是铺天盖地的，这里就不再重复了。直接说重点，从Spring Boot1.x一步步跟着迭代升级到现在的2.3.4也是遇到了很多的坑，了解其新版本的特性是非常重要的，可以帮助我们避免很多不必要的麻烦。

因为我也一直在搞基于Spring Boot技术栈的组件开发工作，最近准备针对基础组件进行部分重构，所以顺便把当前版本的特性从头在顺一遍，就当是回顾总结了，这个回顾只介绍目前版本的一些特性，不对特性展开来叙述，如果有兴趣可以@我，后面我也会根据某一块来进行详细的分析。喜欢的朋友可以跟着看一看，希望对你有所帮助。



## 二、从头开始Application

首先我们先得来初始化各项目用于测试，大家可以使用官方的https://start.spring.io/ 生产，或者使用IDEA的插件Spring Boot进行项目初始化，文末我也会放下我测试demo的地址。

#### 1、应用启动失败（Startup Failure）

如果应用启动失败，Spring Boot会帮我们把大概为什么会启动失败的信息打印在日志中，如下面我用6080端口第二次启动应用就会提示我如下

```java
***************************
APPLICATION FAILED TO START
***************************

Description:

Embedded servlet container failed to start. Port 6080 was already in use.

Action:

Identify and stop the process that's listening on port 8080 or configure this application to listen on another port.
```



有了这种友好的提示真是幸福感爆棚啊，而且Spring Boot 还给我们提供了更多的扩展接口FailureAnalyzer，并提供了响应得抽象类AbstractFailureAnalyzer。如果我们不满足他默认的启动异常信息，就可以通过FailureAnalyzer 来进行一些定制化开发（比如在异常发生的时候打印堆栈等）.FailureAnalyzer的扩展使用了SPI的方式，所以在我们使用的时候需要在应用内创建META-INF/spring.factories，来声明下我们的实现，下面上个小demo。

```java
/** 首先创建我们自己的类，并且可以根据自己的需要来进行异常拦截，这里我拦截的就是端口占用异常PortInUseException
 * @ClassName LearningFailureAnalyzer
 * @Author QIANGLU
 * @Date 2020/9/23 9:10 下午
 * @Version 1.0
 */
public class LearningFailureAnalyzer extends AbstractFailureAnalyzer<PortInUseException> {

    private static final Logger LOGGER = LoggerFactory.getLogger(LearningFailureAnalyzer.class);

    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, PortInUseException cause) {
        LOGGER.error("我出异常了，哇卡卡卡卡卡卡");
        return new FailureAnalysis("端口：" +cause.getPort()+"被占用",cause.getMessage(),rootFailure);
    }
}


//第二步就是创建个META-INF/spring.factories 了，如下
  
org.springframework.boot.diagnostics.FailureAnalyzer=\
  com.qianglu.chapter1.failure.LearningFailureAnalyzer
  
 
//第三布启动两次我们的应用，就会发现，打印的信息是我们需要的了
 ***************************
APPLICATION FAILED TO START
***************************

Description:

端口：6080被占用

Action:

Port 6080 is already in use
  

```



**这东西可用场景其实很多很多，大家想一想有没有点启发**



#### 2、延迟初始化（Lazy Initialization）

在Spring Boot刚出的时候，因为启动加载慢还被人吐槽过，这不，现在懒加载来了。允许你的应用开启懒加载，你的beans 不需要在项目启动的时候被创建了，啥时候用啥时候在创建。这样就能节省你很多启动时间，但有利就有弊，懒加载这玩意在web应用中会导致你很多web相关的bean也被延迟加载，知道有请求进来才会被初始化，所以在使用的时候一定要注意，否则就会有叫你很懵逼的异常了。

并且官方也说了，你都延迟初始化了，那有些问题可能也会延迟被发现。比如我们以前某些配置配错了，经常会在启动的时候就报XXXbean不能被找到之类的。嘿嘿，现在可就不了，一样的启动成功，只有在你用的时候给你掉链子，就问你怕不怕吧。

还有就是官方提示延迟初始化的，会导致初期jvm 内存表现比较小，但要注意配置足够的内存给未来对象创建使用（我觉得一般应用这都不是问题，不需要过多关注）。

下面我们就来看看两种配置方式：

- 使用SpringApplication调用setLazyInitialization 方法设置
- 使用配置spring.main.lazy-initialization=true 

如果你设置了延迟初始化，又有某些特殊的类想初始化，那可以配置@Lazy(false) 关闭其懒加载。



### 3、配置Banner



这玩意说实话我一直不知道有啥用，以前我们都是配置个大佛保平安，娱乐性大于实际吧，当然也许有没GET到的点。

配置方式也很简单，就是在你的classpath下放个banner.txt，通过spring.banner.location 配置来指定下文件位置。当然还有很多属性，什么编码、gif、version之类的我其实懒得说了，没啥兴趣。



####4、配置你的SpringApplication

咱们一般启动类都是直接调用SpringApplication.run就行了。但如果你觉得太简单没啥意思，那其实SpringApplication.run 里面有很多有意思的属性你可以去看看，比如我关闭banner

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(MySpringConfiguration.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
}
```



当然还有很多配置属性，你也可以使用application.yml来配置SpringApplication 的属性。



#### 5、流式构建API（Fluent Builder API）

官方提供了SpringApplicationBuilder 类来帮大家使用流式构建的方式来创建多级的ApplicationContext。SpringApplicationBuilder可以帮助我们构建一种层级关系，如下这种方式等于SpringApplication.run

```java
new SpringApplicationBuilder()
        .sources(Parent.class)
        .child(Application.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
```





#### 6、应用可用性（Application Availability）

这里说的其实就是k8s的Liveness 和 Readiness ，他们已经成为了Spring Boot的核心。

简单说下，Liveness 和 Readiness  在k8s中代表了应用程序状态的各个方面。

Liveness 状态来查看内部情况可以理解为health check，如果Liveness失败就就意味着应用处于故障状态并且目前无法恢复，这种情况就重启吧。

Readiness 状态用来告诉应用是否已经准备好接受客户端请求，如果Readiness未就绪那么k8s就不能路由流量过来。

我们可以用代码来监听Readiness状态，并进行我们需要的处理

```java
@Component
public class ReadinessStateExporter {

    @EventListener
    public void onStateChange(AvailabilityChangeEvent<ReadinessState> event) {
        switch (event.getState()) {
        case ACCEPTING_TRAFFIC:
            // xxxxx
        break;
        case REFUSING_TRAFFIC:
            // xxxxxx
        break;
        }
    }

}
```

我们也能在应用出现故障不能被恢复的时候改变此状态来进行动态的降级和隔离，这个真是太爽了，有机会建议大家试一试

```
@Component
public class LocalCacheVerifier {

    private final ApplicationEventPublisher eventPublisher;

    public LocalCacheVerifier(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void checkLocalCache() {
        try {
            //...
        }
        catch (CacheCompletelyBrokenException ex) {
            AvailabilityChangeEvent.publish(this.eventPublisher, ex, LivenessState.BROKEN);
        }
    }

}
```



#### 7、事件和监听（Application Events and Listeners）

Spring Boot的事件通知机制，简直是解耦神器，包括Spring Cloud的用的分布式远程通知机制其实核心也是这个，这是增加了一层中间件来进行消息的传递。

 有些事件因为实在ApplicationContext创建前就触发了，所以很多时候不能使用@Bean来声明这些事件。最好使用`SpringApplication.addListeners(…)` 和 `SpringApplicationBuilder.listeners(…)` 来注册监听器。

但如果你真是把握不了这些加载时机的话，那有个万能的办法就是配置SPI扩展，直接在META-INF/spring.factories 配置，加上你得listener就行了，如：org.springframework.context.ApplicationListener=com.example.project.MyListener



官方介绍了写在启动时候会发送的事件顺序：

1、ApplicationStartingEvent 	在运行开始的时候发送事件

2、ApplicationEnvironmentPreparedEvent  当Environment在上下文中被使用的时候发送事件

3、ApplicationContextInitializedEvent  在所有的bean定义前，ApplicationContext准备好并且ApplicationContextInitializers已经被调用的时候发送事件

4、ApplicationPreparedEvent 刷新配置前、bean的定义加载之后发送事件

5、ApplicationStartedEvent  刷新上下文后，在执行**CommandLineRunner**的实现之前

6、AvailabilityChangeEvent 发送LivenessState.CORRECT 表面应用是活跃状态

7、ApplicationReadyEvent 在执行**CommandLineRunner**接口之后发送 

8、AvailabilityChangeEvent 发送ReadinessState.ACCEPTING_TRAFFIC 后代表应用可以接入流量

9、ApplicationFailedEvent 发送应用启动失败事件

上面的只是`SpringApplicationEvent` 的事件，一般咱们也不需要对这些进行操作，带你得知道它的存在，以免出了问题都不知道怎么找，其实人家Spring Boot已经都发给你了。



#### 8、Web属性

一般使用SpringApplication就会为我们正确的创建ApplicationContext类型，用于确定WebApplicationType 也就是应用类型的方式其实很简单：

- 如果存在Spring MVC 就使用AnnotationConfigServletWebServerApplicationContext
- 如果Spring MVC不存 在，但是Spring WebFlux存在，就使用AnnotationConfigReactiveWebServerApplicationContext
- 都没有的话就用AnnotationConfigApplicationContext
- 如果你既用了Spring MVC 又用了Spring WebFlux  WebClient，Spring MVC 这一套是默认使用的，除非你设置SpringApplication.setWebApplicationType(WebApplicationType)来强制改变。





#### 9、访问应用参数（Accessing Application Arguments）

如果你想访问SpringApplication.run(…) 的参数，你其实可以注入一个org.springframework.boot.ApplicationArguments 对象，ApplicationArguments这个接口提供对原始String[] 参数以及已解析的选项和非选项参数的访问，上demo:

```java
import org.springframework.boot.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.stereotype.*;

@Component
public class MyBean {

    @Autowired
    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        // if run with "--debug logfile.txt" debug=true, files=["logfile.txt"]
    }

}
```



Spring Boot 中的环境变量也注册了一个CommandLinePropertySource，我们可以使用@Value来获取某个环境变量。





#### 10、使用ApplicationRunner or CommandLineRunner

这俩货也是我们经常会用到的东西，如果你需要在项目启动时加载一些东西，那它俩简直就是神器了，这俩接口都提供了一个run方法，这个方法会在SpringApplication.run(…) 执行完成前被调用。这俩接口适合哪种在应用接收请求前来处理一些东西。



举个使用例子

```java
import org.springframework.boot.*;
import org.springframework.stereotype.*;

@Component
public class MyBean implements CommandLineRunner {

    public void run(String... args) {
        // Do something...
    }

}
```



如果咱们定义了多个CommandLineRunner或ApplicationRunner实现，有的时候又需要有个先后顺序来执行，那就可以用org.springframework.core.annotation.Order 这个注解来定义下。



#### 11、应用退出（ Application Exit）

每个SpringApplication 都会像JVM注册一个关闭钩子（shutdown hook ），来确保能够正常的退出。保证@PreDestroy 注解和DisposableBean 接口这些回调都被执行。



另外，如果你想在使用SpringApplication.exit() 时返回一些特殊的退出代码，可以实现org.springframework.boot.ExitCodeGenerator接口，传递给System.exit() 进行返回。如：

```java
@SpringBootApplication
public class ExitCodeApplication {

    @Bean
    public ExitCodeGenerator exitCodeGenerator() {
        return () -> 42;
    }

    public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(ExitCodeApplication.class, args)));
    }

}
```

ExitCodeGenerator接口可以通过异常实现。遇到此类异常时，Spring Boot返回实现的getExitCode() 方法提供的退出代码





#### 12、管理员功能（Admin Features）

我们可以使用spring.application.admin.enabled 属性来开启管理员功能。开了的话就会把你自己的[`SpringApplicationAdminMXBean`](https://github.com/spring-projects/spring-boot/tree/v2.3.4.RELEASE/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/admin/SpringApplicationAdminMXBean.java)  全部暴露给MBeanServer咯，当然你也可以用这种特性来远程操作你应用。但你要想明白其中的安全性问题，没啥必要的话还是不要乱搞。



## 三、总结

好了，今天太晚了，就先写到这，其实这些内容官方写的都是明明白白的，但自己撸一遍还是很爽，很舒服。在撸的过程中其实你会结合自己实际所用的知识点进行一些思考，新的特性也会给你带来很多未来基础设计上的启发。希望大家有时间精力的话多看看，最起码在需要的时候我知道有这么东西，就有了方向。

> 爱生活，爱编码，微信搜一搜【**架构技术专栏**】关注这个喜欢分享的地方。本文 [架构技术专栏](http://47.104.79.116:4321/) 已收录，有各种视频、资料以及技术文章。

