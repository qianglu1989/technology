> 每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解，此处不涉及详细的原理性解读。

##  

## 一、什么是总线风暴

总线风暴，听着真是一个帅气的词语，但如果发生在你的系统上那就不是很美丽了，废话不多说，先看图说结论。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/T8zZwgeAkRZWs5lcVtvxBURpko5uxSJrujumhPWj6tMTGCZbFK4uBdH44jYKlfd5Vxvjq9eRhAnib8ibNbNhNAgg/640?wx_fmt=jpeg)

什么是总线风暴，先来看结论

在java中使用unsafe实现cas,而其底层由cpp调用汇编指令实现的，如果是多核cpu是使用lock cmpxchg指令，单核cpu 使用compxch指令。如果在短时间内产生大量的cas操作在加上 volatile的嗅探机制则会不断地占用总线带宽，导致总线流量激增，就会产生总线风暴。     总之，就是因为volatile 和CAS 的操作导致BUS总线缓存一致性流量激增所造成的影响。

![img](https://mmbiz.qpic.cn/mmbiz_gif/T8zZwgeAkRZWs5lcVtvxBURpko5uxSJrJrwbB7clDLic9GAASHlFI5KUKgUfXvotSp3nQPGUGTtAHq6yqXUFZpA/640?wx_fmt=gif)

## 二、一些需要的基础知识

这里有些基础需要铺垫下，了解过volatile和cas 的朋友都知道由于一个变量在多个高速缓存中都存在，但由于高速缓存间的数据是不共享的，所以势必会有数据不一致的问题，为了解决这种问题处理器是通过总线锁定**和**缓存锁定这两个机制来保证复杂内存操作的原子性的。



![img](https://mmbiz.qpic.cn/mmbiz_gif/T8zZwgeAkRZWs5lcVtvxBURpko5uxSJrN5R1x3okiadql9GVaO5HiatNCc2qhf4ibpesfZUQcBG3gUfdTkgHXmNYA/640?wx_fmt=gif)

### 1、总线锁

在早期处理器提供一个 LOCK# 信号，CPU1在操作共享变量的时候会预先对总线加锁，此时CPU2就不能通过总线来读取内存中的数据了，但这无疑会大大降低CPU的执行效率。

### 2、缓存一致性协议

由于总线锁的效率太低所以就出现了缓存一致性协议，Intel 的MESI协议就是其中一个佼佼者。MESI协议保证了每个缓存变量中使用的共享变量的副本都是一致的。

### 3、MESI 的核心思想

modified（修改）、exclusive（互斥）、share（共享）、invalid（无效）

如上图，CPU1使用共享数据时会先数据拷贝到CPU1缓存中,然后置为独占状态(E)，这时CPU2也使用了共享数据，也会拷贝也到CPU2缓存中。通过总线嗅探机制，当该CPU1监听总线中其他CPU对内存进行操作，此时共享变量在CPU1和CPU2两个缓存中的状态会被标记为共享状态(S)；

若CPU1将变量通过缓存回写到主存中，需要先锁住缓存行，此时状态切换为（M），向总线发消息告诉其他在嗅探的CPU该变量已经被CPU1改变并回写到主存中。接收到消息的其他CPU会将共享变量状态从（S）改成无效状态（I），缓存行失效。若其他CPU需要再次操作共享变量则需要重新从内存读取。

**缓存一致性协议失效的情况：**

- 共享变量大于缓存行大小，MESI无法进行缓存行加锁；
- CPU并不支持缓存一致性协议

### 4、嗅探机制

每个处理器会通过嗅探器来监控总线上的数据来检查自己缓存内的数据是否过期，如果发现自己缓存行对应的地址被修改了，就会将此缓存行置为无效。当处理器对此数据进行操作时，就会重新从主内存中读取数据到缓存行。

### 5、缓存一致性流量

通过前面都知道了缓存一致性协议，比如MESI会触发嗅探器进行数据传播。当有大量的volatile 和cas 进行数据修改的时候就会产大量嗅探消息。



## 三、总结性言论

通过上面一顿巴拉，大家应该对开局图有一定的了解了，也大概知道了总线风暴的原因。这里再做一下概括性的总结（当前内部还有很有详细的机制，大家感兴趣可以撸一波）

在多核处理器架构上，所有的处理器是共用一条总线的，都是靠此总线来和主内存进行数据交互。当主内存的数据同时存在于多个处理的高速缓存中时，某一处理器更新了此共享数据后。会通过总线触发嗅探机制来通知其他处理器将自己高速缓存内的共享数据置为无效，在下次使用时重新从主内存加载最新数据。而这种通过总线来进行通信则称之为”缓存一致性流量“。

因为总线是固定的，所有相应可以接受的通信能力也就是固定的了，如果缓存一致性流量突然激增，必然会使总线的处理能力受到影响。而恰好CAS和volatile 会导致缓存一致性流量增大。如果很多线程都共享一个变量，当共享变量进行CAS等数据变更时，就有可能产生总线风暴。

![img](https://mmbiz.qpic.cn/mmbiz_gif/T8zZwgeAkRZWs5lcVtvxBURpko5uxSJrO8E5EzVqWbD2BAbEPfZVrmQrc0J6MMHlrKUbEcHqPAFxTdFGMlAibdg/640?wx_fmt=gif)



往期推荐





[每日一个知识点系列：volatile的可见性原理](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975640&idx=1&sn=a8e85ae9ae8d17013490cf09ff92c54e&chksm=f110acc7c66725d19ca1c743fb721434e48ee028804a808493e959aa4196625e246ae772a965&scene=21#wechat_redirect)

[(最新 9000字)  Spring Boot 配置特性解析](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975618&idx=1&sn=79e7f89bf1da2dbe81c3e2ff3f76ddfb&chksm=f110acddc66725cba8a7667b81a90a51fccd9bf840bc6daa162204119b8e18d05f33a9b3ed40&scene=21#wechat_redirect)

[何时用多线程？多线程需要加锁吗？线程数多少最合理？](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975571&idx=1&sn=57c4ca669794a9137a1377eac654a3a6&chksm=f110ad0cc667241a724552eae7a247ca4af6a2b047d29264e133888b4a81eb2e4706b4b19066&scene=21#wechat_redirect)

[Spring Boot 知识清单（一）SpringApplication](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975588&idx=1&sn=87b63472d26ef9f2a60890d4825604ac&chksm=f110ad3bc667242d9831447fe1ed11dc00be836d6f21b68297aae71f02a7a9b0d11b6ad2bf0e&scene=21#wechat_redirect)

[高并发系统，你需要知道的指标（RT...）](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975553&idx=1&sn=afd1ff3f61edfab5c9e43e7ffd3e32ee&chksm=f110ad1ec66724086514bcefe8467b8fc6f208c6b97f16b5a6ee2e0fc658f8db3016218fafcf&scene=21#wechat_redirect)



![img](https://mmbiz.qpic.cn/mmbiz_jpg/T8zZwgeAkRaWjkVYIq2obzleDdyAb3wju35l48udfwoX9MBS4XyTibibo0zxtXl82uC10eibP4foFfcAg3Yic1mibBA/640?wx_fmt=jpeg)