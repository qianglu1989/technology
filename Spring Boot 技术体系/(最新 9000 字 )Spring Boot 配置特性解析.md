> 爱生活，爱编码，微信搜一搜【**架构技术专栏**】关注这个喜欢分享的地方。本文 [架构技术专栏](http://47.104.79.116:4321/) 已收录，有各种JVM、多线程、源码视频、资料以及技术文章等你来拿



## 一、概述

目前Spring Boot版本： 2.3.4.RELEASE，这更新的速度也是嗖嗖的了，随着新版本的发布，也一步步针对公司基础组件进行了升级改造，其中很重要的一块就是配置文件的更新（虽然目前已经全部使用了Apollo）。针对Spring Boot 新版本的配置文件也做了一次梳理，确实发现了以前没有注意到的点。 

 

## 二、新版的外部配置



### 1、基础配置加载

Spring Boot 为我们提供了很多的外部配置参数，我们可以使用 YAML 文件（当然你也可以使用properties,但不建议）、环境变量和命令行参数，来区分不同的环境配置。



使用配置有两种方式：

- 使用注解@Value，来注入Environment 里面包含的属性
- 使用@ConfigurationProperties 来定义一个属性类，来包含我们需要的属性（这些属性都可以配置在YAML中）



**Spring Boot 外部配置这么多，那如果都配置了哪个会生效呢？**

Spring Boot会以下面的顺序来加载配置，优先级从高到低（相同配置优先级高的会覆盖低的），从外到里的来进行配置覆盖加载：



1）开发者全局配置的properties文件（当开发者工具激活时，文件在$HOME/.config/spring-boot下的spring-boot-devtools.properties）



2）测试中配置了@TestPropertySource("base.properties") 注解来加载的配置，比如base.properties这种



3）测试中 使用了@SpringBootTest的 properties



4）命令行参数配置，也就是java -jar后面使用的配置



5）可以使用**SPRING_APPLICATION_JSON** 属性加载的SON配置，加载方式有两种：

- 在系统环境变量加载 SPRING_APPLICATION_JSON='{"persion":{"name":"xxx"}}'，这种加载会将这个数据加到Spring Environment中，我们可以获得一个persion.name 的属性，值为xxx
- 使用System属性加载 java -Dspring.application.json='{"persion":{"name":"xxx"}}' -jar app.jar，这种加载方式会将spring.application.json属性的值当做一个String来加载，不会解析。



6）ServletConfig 初始化的配置



7）ServletContext初始化的配置



8）java:comp/env的JNDI特性



9）Java的系统属性，就是System.getProperties() 获取到的这些



10）操作系统配置的环境变量



11）在RandomValuePropertySource中配置的以random. 开头的属性



12）应用外部配置的 application-{profile}.properties或YAML ,可以使用spring.profiles.active 来选择配置的环境，不选择默认就是application-default.properties。我们可以使用spring.config.location 来定义文件的路径进行加载。



13）在你应用内部配置的application-{profile}.properties 或 YAML,也是用于多环境加载选择使用，可以用spring.profiles.active 来激活配置



14）应用外部配置的application.properties或 YAML



15）应用内部配置的application-{profile}.properties 或 YAML。

这里14、和15 的 SpringApplication 会从application.properties来进行配置属性的加载。

这个配置会从四个位置按照优先级从高到低的方式覆盖加载，高优先级覆盖低优先级的，来看下：

- 应用外部当前目录里  /config 文件夹下的 application.properties 或者application.yml
- 应用外部当前目录里的 application.properties 或者application.yml
- 应用内部classpath下的/config ，也就是resources/config 目录下的 application.properties 或者application.yml
- 应用内部classpath下，也就是resources 目录下的application.properties 或者application.yml



16）@Configuration 配置类上配置了 @PropertySource注解的，但在spring 上下文刷新前这个配置是不会被加载到Environment里面的。这种加载方式不能配置那些应用上下文刷新前就需要加载的属性，比如logging.* 和spring.main.* 这种。

使用方式：

```java
 //这里加载classpath:/com/myco/app.properties文件
@Configuration
 @PropertySource("classpath:/com/myco/app.properties")
 public class AppConfig {

     @Autowired
     Environment env;

     @Bean
     public TestBean testBean() {
         TestBean testBean = new TestBean();
         testBean.setName(env.getProperty("testbean.name"));
         return testBean;
     }
 }
```



17）SpringApplication.setDefaultProperties 设置的参数



下面来用一个Demo 说下：

```java
import org.springframework.stereotype.*;
import org.springframework.beans.factory.annotation.*;

@Component
public class MyBean {

    @Value("${name}")
    private String name;

    // ...

}
```

- 你可以使用 classpath 下的application.yml来配置name= laowang
- 可以使用一个外部的application.yml 来设置一个name = laoli 覆盖上一个配置 (当前name 获取的话是laoli)
- 在可以使用java -jar app.jar --name="Spring" 在来覆盖上一个配置 （当前name获取的话是 Spring）



Spring Boot 配置文件也支持通配符的方式来加载，比如使用 spring.config.additional-location和spring.config.location来加载配置的时候就可以使用通配符加载多个文件。



### 2、配置随机属性

随机属性的注入其实是通过RandomValuePropertySource 来实现的，它可以帮我们来产生integers、 longs、 uuid、strings 的随机值。这个对于我们平时进行一些测试案例还是很实用的。

```java
//可以配置在application.yml
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```



### 3、命令行属性

我们可以使用 java -jar  --server.port=9000的方式将命令行参数server.port 添加到我们应用的Environment，可以使用@Value等方式获取。正常情况下命令行添加的属性优先级是咱们优先级高的。

如果你不想将命令行的属性添加到应用的Environment中，那你可以配置SpringApplication.setAddCommandLineProperties(false)就行了。

其实使用命令行加载最多的可能就是--javaagent 这个属性了，对于现在的公司，APM监控那是必不可少的。我们也可以通过命令参数的配置来临时的加载一些属性进行测试使用。



### 4、应用加载配置文件

其实上面已经说过了，这里在重新提一下。SpringApplication 会从application.yml里面加载属性配置，并将他们添加到Spring 的Environment中供我们使用。优先级如下，高优先级覆盖低的（这里放个原版，可以自己尝试理解下）：

1. A `/config` subdirectory of the current directory
2. The current directory
3. A classpath `/config` package
4. The classpath root



如果你不喜欢配置文件叫做application.properties，也可以使用spring.config.name来进行配置。也可以使用spring.config.location 来指定配置加载路径。

举例说明：

```java
//修改我的配置文件叫myproject
java -jar myproject.jar --spring.config.name=myproject
  
  
//换一个地方来加载我得配置文件
java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties

```

因为spring.config.name 和 spring.config.location 配置是用来确定应用需要加载哪些属性的，所以需要尽可能早的加载。一般都是使用系统环境变量、系统参数、命令行加载的方式进行使用。

默认的配置加载路径如下，安装优先级从高到低排序（file:./config/ 优先级最高），所以在使用加载的时候一定要注意：

1. `file:./config/`
2. `file:./config/*/`
3. `file:./`
4. `classpath:/config/`
5. `classpath:/`



### 5、占位符的使用

在application.properties 我们可以使用占位符来进行属性的动态加载

比如我们可以借助maven 的profile 在打包的时候动态的对环境参数进行替换（比如替换mysql 、redis等域名）

上几个例子：

```java
//简单的使用
app.name=MyApp
app.description=${app.name} is a Spring Boot application
  
//配合命令行参数使用，如参数增加 --port=9000 来代替--server.port=9000，那在配置文件中我们就可以配置
server.port=${port:8080}


```



注意一点：

如果你的POM 里面集成了spring-boot-starter-parent ，那么默认的maven-resources-plugins插件会使用@maven.token@来代替${maven.token}。这么做其实是为了防止和Spring的占位符产生冲突，所以如果我们使用maven 的profile 或者其他的来动态替换application.properties 内部的属性，请使用 @name@.



### 6、YAML文件进行多环境配置

**1） 配置文件使用**

在application.yml中，你可以使用spring.profiles 来激活你想加载的环境配置内容。

例子：

```yaml
server:
    address: 192.168.1.100
---
spring:
    profiles: development
server:
    address: 127.0.0.1
---
spring:
    profiles: production & eu-central
server:
    address: 192.168.1.120
```

在上面的例子中，如果我们激活development 那server.address 就是127.0.0.1。如果production & eu-central 被激活，那server.address 就是192.168.1.120。如果这三个我都没激活，那server.address 就是192.168.1.100，环境配置直接使用---来隔离开。

注意：spring.profiles 这个属性可以是一个名字，也可以是一个表达式。



**2）@Profile注解使用**

我们很多时候会遇到组件动态选择的问题，比如我有多种的权限接入方式或者数据源选择性激活。但我又不想来来回回写点if else，那么@Profile就是一个神器了，他的到来使我们的代码更加的灵活多变。

比如我们重写一个属于源配置：

```java
//第一个
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}

//第二个
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

这样，我们就可以根据不同的配置来激活不同的逻辑了，如果再能搭配上远程配置中心，那就更美丽了。



### 7、YAML的问题

1） YAML有很多优点，那必然也是有一丢丢的小毛病的。那就是YAML文件默认不能使用@PropertySource注解来进行配置加载。如果你不想进行多余的改造，那就老实的建一个properties文件用吧。



2）在YAML中配置多环境配置信息有的时候会有奇奇怪怪的问题，比如下面的：

**application-dev.yml**

```yaml
server:
  port: 8000
---
spring:
  profiles: "!test"
  security:
    user:
      password: "secret"
```

如果此时我启动应用的时候加载了--spring.profiles.active=dev ，那我正常是应该得到security.user.password = secret，但真实的情况却不是这样。

因为我们在配置文件名上使用了xxx-dev.yml，这时候当应用加载的时候就会直接找到application-dev.yml文件.而这时我们配置文件内的多环境配置就失效了。

所以再多环境配置使用的时候，我们要不然就选择xxx-dev.yml、xxx-pro.yml 这种方式，要不然就选择在一个文件内配置多环境。二者只能选一个，以免出现恶心人的问题。



### 8、对象属性绑定

有时候我们会有一组相同类型的属性需要加载，如果使用@Value 那真是累死人。这里Spring Boot为我们提供了一个便捷的方式，我们可以使用一个类对所需要的变量进行统一的配置加载。

举个例子：

```java
//属性类
package com.example;

import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("acme")
public class AcmeProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() { ... }

    public void setEnabled(boolean enabled) { ... }

    public InetAddress getRemoteAddress() { ... }

    public void setRemoteAddress(InetAddress remoteAddress) { ... }

    public Security getSecurity() { ... }

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() { ... }

        public void setUsername(String username) { ... }

        public String getPassword() { ... }

        public void setPassword(String password) { ... }

        public List<String> getRoles() { ... }

        public void setRoles(List<String> roles) { ... }

    }
}

```

这时我在application.yml中配置如下属性，Spring Boot就会帮助我们直接将属性绑定到AcmeProperties类中

```properties
acme.enabled =false

acme.remote-address=127.0.0.1

acme.security.username=xxx
```



因为 **@EnableConfigurationProperties** 只是帮助我们进行声明，在实际使用上我们需要配合**@Configuration**，比如下面的配置，这样配置完后我们就可以使用@Resource 进行属性注入了。

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(AcmeProperties.class)
public class MyConfiguration {
}
```



### 9、宽松的绑定规则

上面聊了@ConfigurationProperties 可以帮助我们进行对象属性绑定，其实在Spring Boot中为我们提供了相当宽松的绑定规则。

比如context-path绑定到 contextPath属性，PORT绑定到 port属性，下面继续搞个Demo。

```java
@ConfigurationProperties(prefix="acme.my-project.person")
public class OwnerProperties {

    private String firstName;

    public String getFirstName() {
        return this.firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

}
```

以上面的代码为例，我们能在application.yml中下面的四种配置形式都可以将属性绑定到  firstName参数上，厉不厉害。

acme.my-project.person.first-name  

acme.myProject.person.firstName

acme.my_project.person.first_name

ACME_MYPROJECT_PERSON_FIRSTNAME

当然，为了统一不出问题，建议都使用小写进行属性声明如 acme.my-project.person.first-name  。



### 10、属性绑定校验

在@ConfigurationProperties  声明的属性类上，我们可以增加**@Validated** 来对配置属性进行校验。

```java
@ConfigurationProperties(prefix="acme")
@Validated
public class AcmeProperties {

    @NotNull
    private InetAddress remoteAddress;

    @Valid
    private final Security security = new Security();

    // ... getters and setters

    public static class Security {

        @NotEmpty
        public String username;

        // ... getters and setters

    }

}
```



> 爱生活，爱编码，微信搜一搜【**架构技术专栏**】关注这个喜欢分享的地方。本文 [架构技术专栏](http://47.104.79.116:4321/) 已收录，有各种JVM、多线程、源码视频、资料以及技术文章等你来拿