> 每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解。
>
> 此处不涉及详细的原理性解读，只作为一种抛砖引玉。
>
> 真正的理解一定是你自我研究探索所收获的知识，加入组织带你一起进步成长。

![](/Users/luqiang/Downloads/公众号图片/jvm.png)

**1.** **调用** **System.gc()**

只是建议虚拟机执行 Full GC，但是虚拟机不一定真正去执行。不建议使用这种方式，而是让虚拟机管理内存。



**2. 未指定老年代和新生代大小，堆伸缩时会产生fullgc,所以一定要配置-Xmx、-Xms**



**3.** **老年代空间不足**

老年代空间不足的常见场景比如大对象、大数组直接进入老年代、长期存活的对象进入老年代等。

为了避免以上原因引起的 Full GC，应当尽量不要创建过大的对象以及数组。

除此之外，可以通过 -Xmn 虚拟机参数调大新生代的大小，让对象尽量在新生代被回收掉，不进入老年代。

还可以通过 -XX:MaxTenuringThreshold 调大对象进入老年代的年龄，让对象在新生代多存活一段时间。

在执行Full GC后空间仍然不足，则抛出错误：`java.lang.OutOfMemoryError: Java heap space `



**4. JDK 1.7** **及以前的（永久代）空间满**

在 JDK 1.7 及以前，HotSpot 虚拟机中的方法区是用永久代实现的，永久代中存放的为一些 Class 的信息、常量、静

态变量等数据。

当系统中要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，在未配置为采用 CMS GC 的情况下也

会执行 Full GC。

如果经过 Full GC 仍然回收不了，那么虚拟机会抛出` java.lang.OutOfMemoryError PermGen space `

措施： 为避免以上原因引起的 Full GC，可采用的方法为增大Perm Gen或转为使用 CMS GC。



**5.** **空间分配担保失败**

空间担保，下面两种情况是空间担保失败：

1、每次晋升的对象的平均大小 > 老年代剩余空间

2、Minor GC后存活的对象超过了老年代剩余空间



注意GC日志中是否有promotion failed和concurrent mode failure两种状况，当出现这两种状况的时候就有可能会触发Full GC。

promotion failed 是在进行 Minor GC时候，survivor space空间放不下只能晋升老年代，而此时老年代也空间不足时发生的。

concurrent mode failure 是在进行CMS GC过程，此时有对象要放入老年代而空间不足造成的，这种情况下会退化使用Serial Old收集器变成单线程的，此时是相当的慢的。



**怎么调优**

围绕一个点，策略就是尽量把对象在新生代使用回收，减少晋升老年代的几率





> 世界上最快的捷径，就是脚踏实地。本文已收录[架构技术专栏](http://www.jiagoujishu.com)关注这个喜欢分享的地方。

开源项目：

- 分布式监控（Gitee GVP最有价值开源项目 ）：https://gitee.com/sanjiankethree/cubic

- 摄像头视频流采集：https://gitee.com/sanjiankethree/cubic-video

- Github 文章收录：https://github.com/qianglu1989/technology