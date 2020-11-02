

## 0、简述

Spring Boot 版本：2.3.4.RELEASE

随着大数据的发展，目前Kafka可以说在我们项目中的使用是越来越多了。其高性能的特点也是满足了我们大部分的场景，所以对于学习Kafka的兼容使用也是一件很重要的事情。

下面我们从几个点来说：

- 发送消息
- 发送回调
- 实现原理
- 异步和同步



## 1、添加依赖

```
 <dependency>
     <groupId>org.springframework.kafka</groupId>
     <artifactId>spring-kafka</artifactId>
 </dependency>
```



## 2、添加配置

在Spring Boot 中kafka的配置属性都是`spring.kafka.*` 开头的,最简配置如下（application.properties中 ）

```
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=myGroup
```



## 3、发送代码

Spring Boot kafka 依然沿用老套路 XXXTemplate，所以这里发送自然就使用了`KafkaTemplate`

```java
@Component
public class Producer {

   private static final Logger logger = LoggerFactory.getLogger(Producer.class);

    @Resource
    private KafkaTemplate<String, String> kafkaTemplate;

    public void send(String msg) {
         kafkaTemplate.send("testTOPIC", msg);
    }

}
```



## 4、参数说明

```java
ListenableFuture<SendResult<K, V>> sendDefault(V data);

ListenableFuture<SendResult<K, V>> sendDefault(K key, V data);

ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, K key, V data);

ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, Long timestamp, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, V data);

ListenableFuture<SendResult<K, V>> send(String topic, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key, V data);

ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);

ListenableFuture<SendResult<K, V>> send(Message<?> message);
```



使用KafkaTemplate.send 会有很多不同的发送参数，这里说明下:

- topic : 填写要发送的topic名称
- partition ： 要发送的分区id,从0开始
- timestamp：时间戳
- key：消息的key
- data：消息数据
- ProducerRecord：消息的封装类，包含了上面的参数
- Message<?> ：Spring自带的Message封装类，包含消息和消息头
  

## 5、发送回调



Spring Boot KafkaAutoConfiguration 为我们提供了处理消息回调的handler，以供我们来处理结果。成功调用onSuccess，失败调用onError，增加如下类：



```java
@Component
public class KafkaSendResultHandler implements ProducerListener {

    private static final Logger log = LoggerFactory.getLogger(KafkaSendResultHandler.class);

    @Override
    public void onSuccess(ProducerRecord producerRecord, RecordMetadata recordMetadata) {
        log.info("Message send success : " + producerRecord.toString());
    }

    @Override
    public void onError(ProducerRecord producerRecord, Exception exception) {
        log.info("Message send error : " + producerRecord.toString());
    }
}
```



## 6、实现原理

从上面来看，我们基本两三行代码就完成了kafka消息的发送，那他们到底是怎么加载实现的呢。

熟悉Spring Boot 的小伙伴想必也能猜到，基于其扩展SPI的机制，`spring-boot-autoconfigure`包下一定会有一个`KafkaAutoConfiguration`配置类。



**1、KafkaAutoConfiguration**

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(KafkaTemplate.class)
@EnableConfigurationProperties(KafkaProperties.class)
@Import({ KafkaAnnotationDrivenConfiguration.class, KafkaStreamsAnnotationDrivenConfiguration.class })
public class KafkaAutoConfiguration {
 
  
}
```

通过`KafkaAutoConfiguration`我们可以看出几件事情

1、`@ConditionalOnClass(KafkaTemplate.class)` 表示我们必须依赖了spring-kafka 包才会加载KafkaAutoConfiguration

2、此类配置了KafkaProperties属性供我们使用

3、这里在加载时帮我们引入了两个类`KafkaAnnotationDrivenConfiguration` 提供消费注解支持 和`KafkaStreamsAnnotationDrivenConfiguration` 提供stream 注解支持



**2、KafkaTemplate**

这里就不大段贴代码了，可以去看完整的`KafkaAutoConfiguration`。

```java
  @Bean
	@ConditionalOnMissingBean(KafkaTemplate.class)
	public KafkaTemplate<?, ?> kafkaTemplate(ProducerFactory<Object, Object> kafkaProducerFactory,
			ProducerListener<Object, Object> kafkaProducerListener,
			ObjectProvider<RecordMessageConverter> messageConverter) {
		KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<>(kafkaProducerFactory);
		messageConverter.ifUnique(kafkaTemplate::setMessageConverter);
		kafkaTemplate.setProducerListener(kafkaProducerListener);
		kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic());
		return kafkaTemplate;
	}
	
  @Bean
	@ConditionalOnMissingBean(ProducerListener.class)
	public ProducerListener<Object, Object> kafkaProducerListener() {
		return new LoggingProducerListener<>();
	}

	
	@Bean
	@ConditionalOnMissingBean(ProducerFactory.class)
	public ProducerFactory<?, ?> kafkaProducerFactory() {
		return factory;
	}
```



这里摘取了KafkaTemplate 主要相关的方法，关键的几个点如下：

- 三个方法使用了@ConditionalOnMissingBean 注解，根本原因就是为了方便我们进行扩展而存在的
- kafkaProducerListener 方法是为了在调用doSend 方法是构建Callback 使用的，方便我们来监控发送成功或失败的信息（KafkaTemplate 的305行buildCallback）
- ProducerFactory 是真正用于创建producer的，如果配置了 transactionIdPrefix 那就代表开启了producer 对于事物的支持。如果开启了事物那就会先从本地ThreadLocal 中获取producer，拿不到才去创建。（KafkaTemplate 的 341 行 getTheProducer）
- 这里如果配置了`spring.kafka.producer.transaction-id-prefix`还会创建一个`KafkaTransactionManager`事务管理器



加载流程：

1、因为我们加入`spring-kafka` jar，所以在启动的时候会通过SPI 机制加载到 `KafkaAutoConfiguration` 

2、这时配置类通过@ConditionalOnMissingBean 发现我们没有独立配置 KafkaTemplate时，会依次加载默认的`ProducerListener`和`ProducerFactory`来构建`KafkaTemplate`



发送流程：

1、调用`KafkaTemplate.send(String topic, @Nullable V data)`方法

2、调用`KafkaTemplate` 内部 `doSend(ProducerRecord<K, V> producerRecord)`方法

3、调用`KafkaTemplate` 内部 `getTheProducer()` 方法，如果是事物发送就从Threadlocal 获取，否则创建一个Producer

4、构造`SettableListenableFuture`回调

5、调用最终的发送方法`KafkaProducer` 内的 `doSend(ProducerRecord<K, V> record, Callback callback)`





## 7、异步和同步发送

通过`KafkaTemplate` 的源码我们可以发现，其实发送消息都是采用异步发送的。`KafkaTemplate`会把我们传入的参数封装成`ProducerRecord`，然后调用`doSend `方法，源码如下：

```java
public ListenableFuture<SendResult<K, V>> send(String topic, K key, @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, key, data);
        return this.doSend(producerRecord);
    }
```



而`doSend(producerRecord) `方法先检查了下是否开启事物，调用`this.getTheProducer() `获取到 producer，后面主要是构造了一个SettableListenableFuture 回调，最后在使用`KafkaProducer.send(ProducerRecord<K, V> record, Callback callback)` 进行数据发送，返回一个`Future<RecordMetadata>`

```java
protected ListenableFuture<SendResult<K, V>> doSend(ProducerRecord<K, V> producerRecord) {
        if (this.transactional) {
            Assert.state(this.inTransaction(), "No transaction is in process; possible solutions: run the template operation within the scope of a template.executeInTransaction() operation, start a transaction with @Transactional before invoking the template method, run in a transaction started by a listener container when consuming a record");
        }

        Producer<K, V> producer = this.getTheProducer();
        this.logger.trace(() -> {
            return "Sending: " + producerRecord;
        });
        SettableListenableFuture<SendResult<K, V>> future = new SettableListenableFuture();
        producer.send(producerRecord, this.buildCallback(producerRecord, producer, future));
        if (this.autoFlush) {
            this.flush();
        }

        this.logger.trace(() -> {
            return "Sent: " + producerRecord;
        });
        return future;
    }
```



**同步发送消息**

因为在某些业务场景下我需要同步发送消息，实现其实也很简单。因为返回了一个`Future<RecordMetadata>` 所以我们只需要调用get方法就行了。

```java
public void syncSend() throws ExecutionException, InterruptedException {
        kafkaTemplate.send("demo", "test sync message").get();
    }
```



