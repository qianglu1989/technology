> 每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解，此处不涉及详细的原理性解读。

![img](https://mmbiz.qpic.cn/mmbiz_png/T8zZwgeAkRYrb8E8hwseAFXHgPjjaEMibrVIIENHgG9YUjNhHbNWWpwVl1VauQYiaWmUSTlhcryhHCLBbl5w8Uzw/640?wx_fmt=png)img

看图说话

**关键点1：** 总线嗅探器（MESI 缓存一致性原理 ）

**关键点2：** 总线锁、缓存锁，为了解决并发问题，会在内存区域的值加锁（内存锁），是在store 之前会给总线内的值加一个锁，write 完成后在解锁（这里大部分是缓存行锁的，总线锁看情况）。

**关键点3：**

就是为了使一个CPU上运行的线程能够读取到另外一个CPU线程的共享变量更新。这个CPU必须先根据无效化队列中存储的消息，删除相应高速缓存内的数据副本，从而在其他CPU更新共享变量时能通过缓存一致性协议同步到该CPU的高速缓存中。内存屏障中的加载屏障 （Load Barrier）就是用来解决这个问题的。Load Barrier会根据会根据无效化队列内容的内存地址，将其他CPU上使用了该缓存的高速缓存中对应的数据状态标记为I，从而使用该CPU后续针对这个的读操作时必须先发送Read消息，以将其他处理器对相关共享变量所做的更新同步到该处理器的高速缓存中。

**总结：**

当修改了增加volatile 的变量时，会马上将变量值写回到主内存中，这时会在store 前对主内存的这个变量加锁，在store 通过总线的时候触发MESI缓存一致性协议，通过总线嗅探器将其他cpu工作内存中的此变量置为无效状态（涉及内存屏障）。当次cpu 完成变量的write 操作时，在对变量进行解锁。



书籍：

推荐一本非常好的关于这方面的书籍，一本书帮你扫平多线程内容，**扫码回复暗号获取：01**

<img src="https://mmbiz.qpic.cn/mmbiz_png/T8zZwgeAkRYrb8E8hwseAFXHgPjjaEMibFoQojbTGmrYJ0NSJAVMU2B3sxibLLvebBUEW9hY9rxMMVJaFYReYEfg/640?wx_fmt=png" alt="img" style="zoom:50%;" />





往期推荐





[(最新 9000字)  Spring Boot 配置特性解析](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975618&idx=1&sn=79e7f89bf1da2dbe81c3e2ff3f76ddfb&chksm=f110acddc66725cba8a7667b81a90a51fccd9bf840bc6daa162204119b8e18d05f33a9b3ed40&scene=21#wechat_redirect)

[Spring Boot 知识清单（一）SpringApplication](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975588&idx=1&sn=87b63472d26ef9f2a60890d4825604ac&chksm=f110ad3bc667242d9831447fe1ed11dc00be836d6f21b68297aae71f02a7a9b0d11b6ad2bf0e&scene=21#wechat_redirect)

[何时用多线程？多线程需要加锁吗？线程数多少最合理？](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975571&idx=1&sn=57c4ca669794a9137a1377eac654a3a6&chksm=f110ad0cc667241a724552eae7a247ca4af6a2b047d29264e133888b4a81eb2e4706b4b19066&scene=21#wechat_redirect)

[高并发系统，你需要知道的指标（RT...）](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975553&idx=1&sn=afd1ff3f61edfab5c9e43e7ffd3e32ee&chksm=f110ad1ec66724086514bcefe8467b8fc6f208c6b97f16b5a6ee2e0fc658f8db3016218fafcf&scene=21#wechat_redirect)

[来，我们在重新说下，线程状态？](http://mp.weixin.qq.com/s?__biz=MzI0MjE4NTM5Mg==&mid=2648975547&idx=1&sn=8af72f78dabe47e858318938f7177ea4&chksm=f110ad64c667247268fee71a5ab1cfa82b159974cfea7fcc595df5f4ebd06865b8b96245aa67&scene=21#wechat_redirect)



![img](https://mmbiz.qpic.cn/mmbiz_jpg/T8zZwgeAkRaWjkVYIq2obzleDdyAb3wju35l48udfwoX9MBS4XyTibibo0zxtXl82uC10eibP4foFfcAg3Yic1mibBA/640?wx_fmt=jpeg)