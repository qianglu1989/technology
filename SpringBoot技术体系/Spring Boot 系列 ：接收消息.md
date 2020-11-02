## 二、接收消息



对于Spring Boot来说，那就更简单了，核心就是一个`@KafkaListener `走天下了



#### 1、接收消息

```java
@Service
public class Consumer {
    private final static Logger logger = LoggerFactory.getLogger(Consumer.class);

    @KafkaListener(topics = "test")
    public void consumes(String msg) {
        logger.info("<<<<<<<<<< consumer msg:[{}]", msg);
    }
}
```



#### 2、参数说明

通过上面的例子可以发现，接收真是太简单了，但其实`@KafkaListener`还有很多属性来帮助我们实现各种场景

- 

 



三、Kafka Streams

四、属性使用

五、嵌入式Kafka

六、扩展监控组件