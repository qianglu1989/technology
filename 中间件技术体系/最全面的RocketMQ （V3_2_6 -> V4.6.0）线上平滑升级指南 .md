## 一、背景
1、各业务系统持续迭代过程中，JDK、SpringBoot、RocketMQ Client 等框架也进行了升级，高版本的 RocketMQ Client 发送的消息到低版本中，在控制台中午无法查看消息明细，致使业务日常排查问题等相当困难。

2、原业务端发送消息与本地事务很难做到一致性，要保障不丢失数据和数据不一致开发成本非常高，RocketMQ V4.4 版本增加了事务消息，引入事务消息后可大大降低实现这一特性的难度。

3、我们对 MQ 的依赖越来越强，MQ 的重要性和稳定性都已经可以和 DB 相当了，而 V4.x 版本增加了更多的新特性和监控手段，可以使我们更好的监控 MQ 的使用情况。

4、V4.x 版本由 Alibaba 维护移交到了 Apache 社区并有他进行维护，促使使用范围更广，也有更多的参与者参与进来，可靠性和及时响应性有了更高的保障。

5、新版本在吞吐率和对新的技术有了更好的支持，基于上述这些因素，我们考虑将 MQ 进行版本升级与改造。

6、升级版本 V3_2_6 -＞ V4.6.0


## 二、流程

因业务特性需求，对当前RocketMQ 集群进行不停机版本迭代升级，步骤如下。

请升级的架构师详细查看文档，进行查漏补缺以免造成不可挽回的事故

下面是此次升级使用的基础资料：


官方文档 

https://rocketmq.apache.org/docs/quick-start/

https://rocketmq.apache.org/dowloading/releases/

Dledger 快速搭建指南：

https://github.com/apache/rocketmq/blob/master/docs/cn/dledger/quick_start.md

Apache RocketMQ 开发者指南：

https://github.com/apache/rocketmq/tree/master/docs/cn 

升级前一定要熟读的两个架构图：

#### 1、消息存储

 
![](https://imgkr2.cn-bj.ufileos.com/1df4b1f6-7860-48e0-923e-006d32682a92.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=UbKfCoa4Bq7Cde%252BfPzvckr8DfG8%253D&Expires=1600262674)



#### 2、消息刷盘

![](https://imgkr2.cn-bj.ufileos.com/67af8bad-4b28-4bec-a2dd-c6adb37f63d3.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=yfhNGBjfxGTzGHxdjaHPLtiSJQc%253D&Expires=1600262687)



## 三、前期准备

#### 1、当前环境版本状态
DEV:   http://10.0.254.191:7080/   V3_5_8    2m

TEST:  http://10.185.240.76:8081/   V3_5_8  2m

PRO：http://rocketmq.pro.siku.cn/  admin/secoo  V3_2_6   2m

#### 2、每个版本组件支持的jre环境
 

| Version | Client | Broker | NameServer |
| --- | --- | --- | --- |
|4.0.0-incubating	|>=1.7	|>=1.8|	>=1.8|
|4.1.0-incubating	>=1.6	>=1.8	>=1.8
|4.2.0	|>=1.6	|>=1.8	|>=1.8|
|4.3.x	|>=1.6	|>=1.8	|>=1.8|
|4.4.x	|>=1.6	|>=1.8	|>=1.8|
|4.5.x	|>=1.6	|>=1.8	|>=1.8|
|4.6.x	|>=1.6	|>=1.8	|>=1.8| 



#### 4、升级时使用到的命令集合
```shell
启动
nohup sh bin/mqnamesrv &
nohup sh bin/mqbroker  -c conf/2m-noslave/broker-b.properties &
 
broker的写权限关闭
bin/mqadmin updateBrokerConfig -b 192.168.x.x:10911 -n 192.168.x.x:9876 -k brokerPermission -v 4
 
恢复该节点的写权限
bin/mqadmin updateBrokerConfig -b 192.168.x.x:10911 -n 192.168.x.x:9876 -k brokerPermission -v 6
 
 
停止
bin/mqshutdown broker
bin/mqshutdown namesrv
 
 
查看集群信息，集群、BrokerName、BrokerId、TPS等信息
./bin/mqadmin clusterList -n localhost:9876
 
 
 
 
获取全部topic
./bin/mqadmin topicList -n localhost:9876 -c DevCluster > topiclist
 
 
获取topic 路由信息
./bin/mqadmin topicRoute -t demo-cluster -n localhost:9876
 
 
 
 
获取topic offset
./bin/mqadmin topicStatus -t demo-cluster -n localhost:9876
 
 
 
 
打印Topic订阅关系、TPS、积累量、24h读写总量等信息
./bin/mqadmin statsAll  -n localhost:9876
 
 
 
 
修改broker 参数
./bin/mqadmin updateBrokerConfig -n localhost:9876 -b 10.0.xxx.2:10911 -k waitTimeMillsInSendQueue -v 500 -c TestCluster
 
 
发送消息
./bin/mqadmin sendMessage -n localhost:9876 -t lqtest -p "this is test"
 
 
 
 
消费
./bin/mqadmin consumeMessage -n localhost:9876 -t lqtest
```

#### 5、官方版本特性收集
这里只标注了下重要的特性，有兴趣的可以查看 http://rocketmq.apache.org/release_notes/

4.0.0 (INCUBATING) 变为Apache

4.4.0 支持消息轨迹 、支持ACL

4.5.0 引入Dledger 的多副本技术

#### 6、新集群4.6.0 集群模型选择

1. 单Master模式

    这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。不建议线上环境使用,可以用于本地测试。


2. 多Master模式

    一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

    优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高；

    缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。



3. 多Master多Slave模式-异步复制

    每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

    优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；

    缺点：Master宕机，磁盘损坏情况下会丢失少量消息。



4. 多Master多Slave模式-同步双写（参考当前业务与并发度，选择此集群模式）

    每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即只有主备都写成功，才向应用返回成功，这种模式的优缺点如下：

    优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高；

    缺点：性能比异步复制模式略低（大约低10%左右），发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机
    
**请结合自己的集群特点和稳定性进行选择升级，不一定最新的集群模式就是最适合你们得。一定要把平滑升级放在首位。**

#### 7、TOPIC 整理 
可以写个脚本整理现有topic 目录，在升级完成后对topic 列表和分区进行整理校对。

因为topic在rocketmq的设计思想里，是作为同一个业务逻辑消息的组织形式，它仅仅是一个逻辑上的概念，而在一个topic下又包含若干个逻辑队列，即消息队列，消息内容实际是存放在队列中，而队列又存储在broker中

**一定要进行特殊业务场景梳理**
1）顺序消费
2）topic单broker配置


以免出现一个Topic只有一个queue的情况出现，导致消息丢失。随便找了个图，大家可以看下
 ![](https://imgkr2.cn-bj.ufileos.com/d5bafe71-0a6f-4396-9229-6caa68e34441.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=U90FplZSyBlCqBtyvW0fJa8gYtw%253D&Expires=1600262898)




#### 8、新集群配置
```
brokerClusterName=MQCluster
brokerName=broker-ali-76
brokerId=0
deleteWhen=04
fileReservedTime=360
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
storePathCommitLog=/data/rocketmq/store/commitlog
storePathConsumerQueue=/data/rocketmq/store/consumequeue
storePathRootDir=/data/rocketmq/store
autoCreateSubscriptionGroup=true
## if msg tracing is open,the flag will be true
traceTopicEnable=true
listenPort=10911
namesrvAddr=10.48.xx.76:9876;10.48.xx.77:9876
```

## 四、升级步骤

当攻略做完以后，我们就可以开始搞起了。我选择的最终架构模式：多Master多Slave模式-同步双写

**流程概述：**

0. 修改2m-2s-sync 、runbroker、runserver 配置参数

1. 停掉3.2.6 nameserver 原IP PORT 启动4.6.0 nameserver，逐级替换完毕

2. 停掉3.2.6 broker 启动4.6.0 broker（查看是否有单机topic问题），逐级替换完毕

3. 测试集群稳定性，为新集群增加slave,升级完成



**详细步骤：**

准备操作

1、下载最新4.6.0版本部署包
```
cd /data/xxx_tomcat

wget http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.6.0/rocketmq-all-4.6.0-bin-release.zip

unzip rocketmq-all-4.6.0-bin-release

```
2、修改配置
```
cd /data/xxx_tomcat/rocketmq-4.6.0/conf/2m-2s-sync

修改51、50 机器broker配置

修改配置 2m-2s-sync

修改 runbroker JVM配置，以免使用了默认配置导致内存不够
```

3、两台M配置如下：

```
两台差别只在brokerName 
brokerClusterName=MQCluster
brokerName=broker-60-50
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
storePathCommitLog=/data/alibaba-rocketmq/store/commitlog
storePathConsumerQueue=/data/rocketmq/store/consumequeue
storePathRootDir=/data/alibaba-rocketmq/store
autoCreateSubscriptionGroup=true
## if msg tracing is open,the flag will be true
traceTopicEnable=true
listenPort=10911
namesrvAddr=192.168.xxx.50:9876;192.168.xxx.51:9876
```

4、替换nameserver
```
jps -l

sh bin/mqshutdown namesrv
cd /data/xxx_tomcat/rocketmq-4.6.0

nohup sh bin/mqnamesrv &

./bin/mqadmin clusterList -n localhost:9876
```

5、替换broker
```
jps -l

sh bin/mqshutdown broker
cd /data/xxx_tomcat/rocketmq-4.6.0

ps -ef|grep mq  //检查使用的配置文件

nohup sh bin/mqbroker  -c conf/2m-2s-sync/broker-b.properties &

./bin/mqadmin clusterList -n localhost:9876
```

最后我们来发个消息测试下
```
./bin/mqadmin sendMessage -n localhost:9876 -t lqtest -p "this is test"

./bin/mqadmin consumeMessage -n localhost:9876 -t lqtest
```
恭喜你到此你的集群就升级完毕了！！！


------------
关注公众号获取更多视频资料：

![](https://imgkr2.cn-bj.ufileos.com/430282a1-e054-4998-9653-e05a8bcbe4f8.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=TXYscubbHlRILP4ghudtxhRZaG0%253D&Expires=1600264328)



![](https://imgkr2.cn-bj.ufileos.com/ab4277bc-87ee-417e-a145-559e84c18404.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=oAdjgaEH5%252Fec62k6PpIJG1Q5x%252B8%253D&Expires=1600263052)


