

# 0x01 运行时数据区

![img](https://mmbiz.qpic.cn/mmbiz_png/T8zZwgeAkRbasncZVCYhBibd5ySx5ZicXHeYQrwxN6d774IBbX0gCDxyPsliaNogM4icoILADunOkruHPE2tYMxOcQ/640?wx_fmt=png)

#### 程序计数器：

可以看成当前线程执行的字节码行号的指示器，是流程控制指示器分支、循环 跳转 异常处理 线程切换都依赖它，属于线程私有。

如果线程执行的是一个java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址。

如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined） 是内存中唯一一个没有OutOfMemiryError的地方

  

#### 虚拟机栈：

是线程私有的，生命周期和线程相同。

方法调用时都会创建栈针虚拟机栈包含，局部变量表、操作数栈、动态链接、方法出口，方法调用直到完成就是一个入栈出栈的过程。

局部变量表存储了基础数据 long int byte char double float boolean short、对象引用、和返回地址。

局部变量表空间以局部变量槽表示，其中long double占两个64位，其余占一个。

局部变量表所在的内存空间在编译期就完成分配。  

异常问题，有两类，stackOverFlowError ,OutOfMemoryError,只要栈申请内存分配完成就不会发生第二类异常。

  

#### 本地方法栈：

和虚拟机栈非常相似、是为虚拟机使用的本地方法服务也有stackOverFlowError ,OutOfMemoryError 两类异常。

其区别只是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native）方法服务。

  

#### java 堆

所有线程共享，几乎所有的实例对象和数组都在此上分配内存。 

可以划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB），以提升对象分配时的效率

会有OutOfMemoryError异常

#### 方法区

和堆一样都是线程共享的，用于存储虚拟机加载的类型信息、常量、静态变量、即使编译器编译后的代码。

jdk7 永久带中字符串常量池、 静态变量被移到堆 ，配置参数 -XX:MaxPermSize -XX:PermSize

jdk8 完全废弃了永久代概念，变成元空间，把JDK 7中永久代还剩余的内容（主要是类型信息）全部移到元空间中，配置参数 -XX:MetaspaceSize  -XX:MaxMetaspaceSize

注意：jdk8一定要配置元空间参数，其中XX:MetaspaceSize 不管配置多大初始默认都是20.8m，这点和-XX:PermSize 的配置多大是多大有点不一样。所以元空间区间范围 [20.8m, MaxMetaspaceSize)

  

#### 运行时常量池

方法区的一部分，class文件中除了有类的版本、字段、方法、接口描述信息外，还有一个常量表用于存储字面量和符号引用、这部分在类加载后放到方法区的运行时常量池。

常量池是方法区的一部分，所以受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

  

#### 直接内存

不算虚拟机运行时数据区的一部分

jdk1.4加入了NIO（New Input/Output）类，引入基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

既然是内存，则肯定还是会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制，也可能导致OutOfMemoryError异常

  

# 0x02 对象创建

都知道一般创建对象直接 new就可以了，那具体内部到底发生了什么！

1、遇到new 关键字后会先去常量池寻找是否有此类的符号引用，此引用的类是否被加载、解析初始化过,如果没有，那必须先执行相应的类加载过程。

2、内存分配，检查完毕后，为新对象分配内存，对象所需要的内存在类加载后就完全确定了，根据内存是否规整有两种分配方式，而内存是否规整又根据垃圾收集器决定的。

空闲列表：假如内存不规整，虚拟机内部必须维护一个列表，记录哪块内存可用，分配时从列表中找出一块足够大的空间划分给对象实例。

指针碰撞：假如内存规整，分配内存就仅仅把指针向空闲空间方向挪动一段与对象大小相等的距离。

所以当使用Serial、ParNew 带压缩的收集器时，系统采用指针碰撞分配内存，使用CMS 时会使用空闲列表。

对象创建是非常频繁的，所以仅修改指针分配在并发情况下是不安全的。为了解决并发分配的问题有两种可选方案：

本地线程缓冲区 TLAB（Thread Local Allocation Buffer） ：每个线程创建时预分配一块内存，称为本地线程分配缓冲，线程内创建对象时先使用TLAB

分配动作同步处理：虚拟机采用CAS配上失败重试的方式保证更新原子性

3、初始化，内存分配完毕后就需要将分配的内存空间（不包括对象头）进行初始化，如果使用了TLAB ，初始化工作将提前到TLAB分配时顺便进行。

此操作保证了实例字段会附有初始值，对应着类加载的初始化过程。<client>()方法也在此执行，这方法由编译器自动收集类中所有类变量的赋值动作和静态语句块static{}中的语句块合并产生。



4、设置对象头， 需要设置下这个对象是哪个类的实例、如何找到类的元数据信息、对象哈希码、对象GC分代年龄等信息。

5、 到目前为止，从虚拟机来看一个对象已经产生了。从java角度看，对象创建才刚开始，当前还没有执行类的构造函数，即CLass 文件中的<init>()方法还没执行，所有的字段都是默认值0/null。其实从开发角度看来在new 指令后就会执行方法，对对象进行初始化操作。  

# 0x03 对象内存布局

对象已经创建了，但对象内部到底是什么情况，到底有哪些属性？

其实对象在内存中也没那么复杂，分为三个部分   **对象头**  对象头就和你人的名片一样，表明了你这个对象在整个jvm世界中的一种状态属性信息，对象头包含了两种类型的信息：  

1、存储对象自身的运行时数据   

 哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳，这部分数据在虚拟机32位 ｜ 64位 分别占用32bit | 64bit ，称为Mark Word。

都说Mark Word被设计成动态的数据结构，其实指的就是上面不同属性所占用的存储空间是变动的。

2、对象指向它的类型元数据的指针 该指针用于确定该对象属于哪个类，如果对象是数组，对象头中还必须有一块用于记录数组长度。 

**实例数据**：对象真正存储的有效信息，就是程序代码里面定义的各种类型字段内容。

**对齐填充** ：纯占位使用，因为虚拟机自动内存管理系统要求对象起止地址必须是8字节整数倍。

# 0x04 对象的访问定位

创建完了的对象后程序可以通过栈上的reference来引用和操作对象，那到底怎么定位到内存中的对象呢？

其实对象的访问方式是由虚拟机实现而定。主流有如下两种方式：

**句柄：**

使用句柄其实就是堆中搞了一个句柄池，reference中存储的其实就是句柄地址，而句柄中包含了对象实例数据和类型数据的具体地址信息。

好处就是reference引用的句柄稳定，在对象移动时（GC的时候）只会改变句柄实例数据指针而已

**直接指针：**

reference中存储的直接就是对象地址，如果只访问对象，就不需要间接开销 

![img](https://mmbiz.qpic.cn/mmbiz_png/T8zZwgeAkRbasncZVCYhBibd5ySx5ZicXHEgTzdpD0JvXeDXThbRbDKv4IPyqQyQu2Hz4CxN6xdD3a5vrxrsjSTg/640?wx_fmt=png)

  

# 0x05 OutOfMemoryError

上面说了在JVM中除了程序计数器没有OutOfMemoryError，其他基本都有可能发生，下面总结下各个区域

**堆溢出**

```
 错误：java.lang.OutOfMemoeyError: Java heap spaceDumping heap to java_pid1120.hprof ...Heap dump file created [22045981 bytes in 0.66 secs]
```

我们可以配置jvm 启动参数 -XX：+HeapDumpOnOutOfMemoryError 在堆溢出时Dump当前内存快照，后续可以使用Mat进行数据分析



**1、虚拟机栈和本地方法栈溢出**

```
错误：java.lang.StackOverflowErrorjava.lang.OutOfMemoeyError
```

一般我们使用-Xss来设置栈容量，栈有两种异常表现 1、线程执行时请求栈深度超过了你设置的容量，就会抛出 StackOverflowError，比如你搞了个死循环 2、创建线程申请内存时，如果无法活的足够内存就会出现OutOfMemoeyError（所以程序中如果线程开得太多而不回收也容易造成OOM ）

**2、方法区和运行时常量池溢出**

经典案例，while(true) String::intern() 本地方法，它的意思就是字符串常量池中有就返回引用，没有就把这个添加进去返回此对象引用。

```
错误：
 java.lang.OutOfMemoeyError: PermGen space  at java.lang.String.intern(Native Method)
```

经测试：  

JDK1.6  会发生 java.lang.OutOfMemoeyError: PermGen space 从信息“PermGen space” 看出运行时常量池属于方法区

JDK1.7 JDK1.8  不会发生异常，是因为从JDK1.7开始 字符串常量池被移到了堆中 

JDK1.7 要注意使用动态代理的情况下新类型的创建，否则容易溢出 ，而一个类如果要被垃圾回收，条件是比较苛刻的。  

JDK1.8  永久代被元空间代替了，默认设置下，再也不需要担心方法区溢出了，-XX：MaxMetaspaceSize：设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小，但在实际使用中还需要进行设置，防止资源耗尽。

**3、直接内存溢出**

```
 错误：    Exception in thread "main"  java.lang.OutOfMemoeyError:            at sum.misc.Unsafe.allocateMemory(Native Method)
```

上面提过NIO 可以使用直接内存，容量大小可通过-XX：MaxDirectMemorySize参数来指定，如果不指定默认与java 堆最大值（-Xmx）一样。  

DirectByteBuffer类直接通过反射获取Unsafe实例进行内存分配