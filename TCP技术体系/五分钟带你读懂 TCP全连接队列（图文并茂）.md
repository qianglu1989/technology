## 一、问题
今天有个小伙伴跑过来告诉我有个奇怪的问题需要协助下，问题确实也很奇怪。客户端调用RT比较高并伴随着间歇性异常Connection reset出现，而服务端CPU 、线程栈等看起来貌似都很正常，而且服务端的RT很短。


**这里先说下结果：**
因为TCP全连接队列太小导致的连接被丢弃，因为项目使用Spring Boot 内置的Tomcat，而默认accept-count是100，而这个参数在这里就代表了全连接队列大小。所以在请求波峰的时候全连接队列被打满导致有连接丢弃。所以我们调整server.tomcat.accept-count这个参数解决了问题。

## 二、半连接队列和全连接队列

好了为了知其然知其所以然，从异常信息来看可能是TCP连接出现了什么问题，其中重点就是半连接队列和全连接队列。下面就来看看什么是TCP 半连接队列和全连接队列，其为什么会出现这种奇怪的现象。



#### 1、TCP 三次握手流程和队列

TCP三次握手时，Linux内核会维护两个队列：
- 半连接队列，被称为SYN队列
- 全连接队列，被称为 accept队列

老生常谈，还要从大家都熟悉TCP三次握手说起，来看一张图：
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200708225906164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2x1cWlhbmc4MTE5MTI5Mw==,size_16,color_FFFFFF,t_70)

1、客户端发送SYN包，并进入SYN_SENT状态

2、服务端接收到数据包将相关信息放入半连接队列（SYN 队列）,并返回SYC+ACK包给客户端。

3、服务端接收客户端ACK数据包，这时如果全连接队列（accept 队列）没满,就会从半连接队列里面将数据取出来放入全连接队列，等待应用使用,当队列已满就会跟据tcp_abort_on_overflow配置执行策略。

这里半连接队列（SYN 队列）和全连接队列（accept 队列）就是重点了。

#### 2、全连接队列查看

当查询问题的时候，我们就需要查看全连接队列的状态。服务端我们可以使用 ss 命令进行查看，ss 命令获取数据又分为LISTEN 状态，和非LISTEN 状态。

**LISTEN 状态下数据：**
```shell
# -l 显示正在Listener 的socket
# -n 不解析服务名称
# -t 只显示tcp

# Recv-Q 完成三次握手并等待服务端 accept() 的 TCP 全连接总数，
# Send-Q 全连接队列大小

[root@server ~]#  ss -lnt |grep 6080
State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port
LISTEN     0   100       :::6080                  :::*
```


**非LISTEN 状态下数据：**
```shell
# Recv-Q 已收到但未被应用进程读取的字节数
# Send-Q 已发送但未收到确认的字节数

[root@server ~]#  ss -nt |grep 6080
State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port
ESTAB     0   433       :::6080                  :::*
```


#### 3、全连接队列溢出

当有大量请求进入，如果TCP全连接队列过小的话就会出现全连接队列溢出，当出现全连接队列溢出现象的时候，后续的请求就会被丢弃，就会出现服务请求数量上不去的现象。

前面提到在TCP三次握手的最后一步，当全连接队列已满就会根据tcp_abort_on_overflow策略进行处理。Linux 可通过 /proc/sys/net/ipv4/tcp_abort_on_overflow 进行配置。

- 当tcp_abort_on_overflow=0，服务accept 队列满了，客户端发来ack,服务端直接丢弃该ACK，此时服务端处于【syn_rcvd】的状态，客户端处于【established】的状态。在该状态下会有一个定时器重传服务端 SYN/ACK 给客户端（不超过 /proc/sys/net/ipv4/tcp_synack_retries 指定的次数，Linux下默认5）。超过后，服务器不在重传，后续也不会有任何动作。如果此时客户端发送数据过来，服务端会返回RST。（这也就是我们的异常原因了）

- 当tcp_abort_on_overflow=1，服务端accept队列满了，客户端发来ack，服务端直接返回RST通知client，表示废掉这个握手过程和这个连接，client会报connection reset by peer。

##### 1）. **当全连接队列溢出时，有哪些指标可以说明呢，我们又有哪些有效的查询手段呢？**

命令查询，我们可以根据TCP 的握手特性来看:
```shell
[root@server ~] netstat -s | egrep "listen|LISTEN"
  7102 times the listen queue of a socket overflowed 全连接队列溢出的次数
  7102 SYNs to LISTEN sockets ignored 表示半连接队列溢出次数
  
  710 2times表示全连接队列溢出的次数，隔几秒查询一次，如果这个数字一直在递增，说明全连接队列出现了溢出的状态

```

##### 2）. **配置全连接队列和半连接队列？**

全连接队列大小取决于backlog 和somaxconn 的最小值，也就是 min(backlog,somaxconn)

- somaxconn 是Linux内核参数，默认128，可通过/proc/sys/net/core/somaxconn进行配置
- backlog是 listen(int sockfd,int backlog)函数中的参数backlog，Tomcat 默认100，Nginx 默认511.


半连接队列的长度可以通过 /proc/sys/net/ipv4/tcp_max_syn_backlog来设置.os层面，只能设一个，由所有程序共享）

##### 3）.  **查看半连接状态**
半连接，也就是服务端处于SYN_RECV状态的TCP连接，这种状态的都在半连接队列，因此可以使用如下命令进行计算：
```shell
#查看半连接队列
[root@server ~]  netstat -natp | grep SYN_RECV | wc -l
233 #表示半连接状态的TCP连接有233个
```
## 三、总结
通过以上的知识点可以定位到这次事件的原因了，因为Spring Boot tomcat 默认的配置导致应用在启动时全连接队列只有默认的100，在流量激增的情况下导致全连接队列打满出现了第三次握手数据包被丢弃的现象。

所以总结下：
- 1、TCP三次握手时，Linux维护了全连接和半连接两个队列
- 2、在全连接队列满的时候丢弃策略根据tcp_abort_on_overflow的配置执行
- 3、全连接队列大小会取Linux系统配置和应用配置中的最小值
- 4、Linux 中的backlog 就是我们所说的全连接队列大小
- 5、应用部署时记得检查全连接队列是否正确配置

backlog配置
- Tomcat AbstractEndpoint默认参数是100，如果使用独立Tomcat配置了 server.xml，其实 connector 中 acceptCount 最终是 backlog的值。而使用Spring Boot内置Tomcat记得配置server.tomcat.accept-count参数，否则默认值就是
- Nginx 配置 server{ listen 8080  default_server backlog=512}
- Redis 配置redis.conf文件 tcp-backlog 511参数

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLWFlNDE5YjJiYTk0Njc3ZGQxZWYxMzA3MGFlODVhYzU5MTk1LkpQRUc?x-oss-process=image/format,png)


>专注于分享技术干货文章的地方，内容涵盖java基础、中间件、分布式、apm监控方案、异常问题定位等技术栈。多年基础架构经验，擅长基础组件研发，分布式监控系统，热爱技术，热爱分享
>![在这里插入图片描述](https://img-blog.csdnimg.cn/20200708225916542.jpg#pic_center)