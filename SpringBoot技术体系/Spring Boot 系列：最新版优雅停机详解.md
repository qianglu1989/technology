> 爱生活，爱编码，本文已收录[架构技术专栏](http://www.jiagoujishu.com/)关注这个喜欢分享的地方。

开源项目：

- 分布式监控（Gitee GVP最有价值开源项目 ）：https://gitee.com/sanjiankethree/cubic

- 摄像头视频流采集：https://gitee.com/sanjiankethree/cubic-video



## 优雅停机

目前Spring Boot已经发展到了2.3.4.RELEASE，伴随着2.3版本的到来，优雅停机机制也更加完善了。

目前版本的Spring Boot 优雅停机支持Jetty, Reactor Netty, Tomcat和 Undertow 以及反应式和基于 Servlet 的 web 应用程序都支持优雅停机功能。



**优雅停机的目的：**

如果没有优雅停机，服务器此时直接直接关闭(kill -9)，那么就会导致当前正在容器内运行的业务直接失败，在某些特殊的场景下产生脏数据。



**增加了优雅停机配置后：**

在服务器执行关闭（kill -2）时，会预留一点时间使容器内部业务线程执行完毕，此时容器也不允许新的请求进入。新请求的处理方式跟web服务器有关，Reactor Netty、 Tomcat将停止接入请求，Undertow的处理方式是返回503.



## 新版配置

#### YAML配置

新版本配置非常简单，server.shutdown=graceful 就搞定了（注意，优雅停机配置需要配合Tomcat 9.0.33（含）以上版本）

```yaml
server:
  port: 6080
  shutdown: graceful #开启优雅停机
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s #设置缓冲时间 默认30s
```



在设置了缓冲参数timeout-per-shutdown-phase 后，在规定时间内如果线程无法执行完毕则会被强制停机。

下面我们来看下停机时，加了优雅停日志和不加的区别：

****

```java
//未加优雅停机配置
Disconnected from the target VM, address: '127.0.0.1:49754', transport: 'socket'
Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

加了优雅停机配置后，可明显发现的日志 **Waiting for active requests to cpmplete**,此时容器将在ShutdownHook执行完毕后停止。


![](https://imgkr2.cn-bj.ufileos.com/6bba6a13-beb4-466e-942d-76828d74e99c.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=VjHXDoKrahydmHRzhuUw05JFkjU%253D&Expires=1602773654)



#### 关闭方式

1、 一定不要使用kill -9 操作，使用kill -2 来关闭容器。这样才会触发java内部ShutdownHook操作，kill -9不会触发ShutdownHook。

2、可以使用端点监控 POST 请求 **/actuator/shutdown** 来执行优雅关机。



#### 添加ShutdownHook

通过上面的日志我们发现Druid执行了自己的ShutdownHook，那么我们也来添加下ShutdownHook，有几种简单的方式：

1、实现DisposableBean接口，实现destroy方法

```java
@Slf4j
@Service
public class DefaultDataStore implements DisposableBean {


    private final ExecutorService executorService = new ThreadPoolExecutor(OSUtil.getAvailableProcessors(), OSUtil.getAvailableProcessors() + 1, 1, TimeUnit.MINUTES, new ArrayBlockingQueue<>(200), new DefaultThreadFactory("UploadVideo"));


    @Override
    public void destroy() throws Exception {
        log.info("准备优雅停止应用使用 DisposableBean");
        executorService.shutdown();
    }
}
```





2、使用@PreDestroy注解

```java
@Slf4j
@Service
public class DefaultDataStore {


    private final ExecutorService executorService = new ThreadPoolExecutor(OSUtil.getAvailableProcessors(), OSUtil.getAvailableProcessors() + 1, 1, TimeUnit.MINUTES, new ArrayBlockingQueue<>(200), new DefaultThreadFactory("UploadVideo"));


    @PreDestroy
    public void shutdown() {
        log.info("准备优雅停止应用 @PreDestroy");
        executorService.shutdown();
    }

}
```

这里注意，@PreDestroy 比 DisposableBean 先执行

#### 关闭原理

1、使用kill pid关闭，源码很简单，大家可以看下GracefulShutdown

```java
	private void doShutdown(GracefulShutdownCallback callback) {
		List<Connector> connectors = getConnectors();
		connectors.forEach(this::close);
		try {
			for (Container host : this.tomcat.getEngine().findChildren()) {
				for (Container context : host.findChildren()) {
					while (isActive(context)) {
						if (this.aborted) {
							logger.info("Graceful shutdown aborted with one or more requests still active");
							callback.shutdownComplete(GracefulShutdownResult.REQUESTS_ACTIVE);
							return;
						}
						Thread.sleep(50);
					}
				}
			}

		}
		catch (InterruptedException ex) {
			Thread.currentThread().interrupt();
		}
		logger.info("Graceful shutdown complete");
		callback.shutdownComplete(GracefulShutdownResult.IDLE);
	}
```





2、使用端点监控 POST 请求 **/actuator/shutdown**关闭

因为actuator 都使用了SPI的扩展方式，所以我们看下AutoConfiguration，可以看到关键点就是ShutdownEndpoint

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnAvailableEndpoint(
    endpoint = ShutdownEndpoint.class
)
public class ShutdownEndpointAutoConfiguration {
    public ShutdownEndpointAutoConfiguration() {
    }

    @Bean(
        destroyMethod = ""
    )
    @ConditionalOnMissingBean
    public ShutdownEndpoint shutdownEndpoint() {
        return new ShutdownEndpoint();
    }
}

```



**ShutdownEndpoint**,为了节省篇幅只留了一点重要的

```java
@Endpoint(
    id = "shutdown",
    enableByDefault = false
)
public class ShutdownEndpoint implements ApplicationContextAware {
     
    @WriteOperation
    public Map<String, String> shutdown() {
        if (this.context == null) {
            return NO_CONTEXT_MESSAGE;
        } else {
            boolean var6 = false;

            Map var1;
            try {
                var6 = true;
                var1 = SHUTDOWN_MESSAGE;
                var6 = false;
            } finally {
                if (var6) {
                    Thread thread = new Thread(this::performShutdown);
                    thread.setContextClassLoader(this.getClass().getClassLoader());
                    thread.start();
                }
            }

            Thread thread = new Thread(this::performShutdown);
            thread.setContextClassLoader(this.getClass().getClassLoader());
            thread.start();
            return var1;
        }
    }
  
      private void performShutdown() {
        try {
            Thread.sleep(500L);
        } catch (InterruptedException var2) {
            Thread.currentThread().interrupt();
        }

        this.context.close();  //这里才是核心
    }
}
```

在调用了 this.context.close() ，其实就是AbstractApplicationContext 的close() 方法 （重点是其中的doClose()）

```java
/**
	 * Close this application context, destroying all beans in its bean factory.
	 * <p>Delegates to {@code doClose()} for the actual closing procedure.
	 * Also removes a JVM shutdown hook, if registered, as it's not needed anymore.
	 * @see #doClose()
	 * @see #registerShutdownHook()
	 */
	@Override
	public void close() {
		synchronized (this.startupShutdownMonitor) {
			doClose(); //重点：销毁bean 并执行jvm shutdown hook
			// If we registered a JVM shutdown hook, we don't need it anymore now:
			// We've already explicitly closed the context.
			if (this.shutdownHook != null) {
				try {
					Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
				}
				catch (IllegalStateException ex) {
					// ignore - VM is already shutting down
				}
			}
		}
	}
```



## 后记

到这里，关于`单机`版本的Spring Boot优雅停机就说完了。为什么说`单机`？因为大家也能发现，在关闭时，其实只是保证了服务端内部线程执行完毕，调用方的状态是没关注的。

不论是Dubbo还是Cloud 的分布式服务框架，需要关注的是怎么能在服务停止前，先将提供者在注册中心进行反注册，然后在停止服务提供者，这样才能保证业务系统不会产生各种503、timeout等现象。



好在当前Spring Boot 结合Kubernetes已经帮我们搞定了这一点，也就是Spring Boot 2.3版本新功能Liveness（存活状态） 和Readiness（就绪状态）

简单的提下这两个状态：

- Liveness（存活状态）：Liveness 状态来查看内部情况可以理解为health check，如果Liveness失败就就意味着应用处于故障状态并且目前无法恢复，这种情况就重启吧。此时Kubernetes如果存活探测失败将杀死Container。
- Readiness（就绪状态）：用来告诉应用是否已经准备好接受客户端请求，如果Readiness未就绪那么k8s就不能路由流量过来。

