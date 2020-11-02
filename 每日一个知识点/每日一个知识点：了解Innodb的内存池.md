

>每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解。
>
>此处不涉及详细的原理性解读，只作为一种抛砖引玉。
>
>真正的理解一定是你自我研究探索所收获的知识，加入组织带你一起进步成长。



通过前几天的知识点我们知道，在Innodb体系架构里，主要包含后台线程和内存两大块，今天就来说下Innodb体系架构里内存的一些知识点。架构图如下：

<img src="/Users/luqiang/Library/Application Support/typora-user-images/image-20201023105308493.png" alt="image-20201023105308493" style="zoom:50%;" />





## Innodb的内存



老规矩，先上图，从全局看一下：

<img src="/Users/luqiang/Downloads/公众号图片/内存池.jpg" alt="内存池" style="zoom:50%;" />



目前我们可以看出，InnoDB内存主要有两大部分：

- 缓冲池
- 重做日志缓冲



InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。

因此可将其视为基于磁盘的数据库系统（Disk-base Database）。

在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能



## 缓冲池



缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。



**缓冲池中缓存的数据页类型：**

- 索引页
- 数据页
- undo页
- 插入缓冲（insert buffer）
- 自适应哈希索引（adaptive hash index）
- InnoDB存储的锁信息（lock info）
- 数据字典信息（data dictionary



**读数据操作**：

- 首先将从磁盘读到的页存放在缓冲池中，这个过程称为将页“FIX”在缓冲池中

- 下一次再读相同的页时，首先判断该页是否在缓冲池中

- 若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页



**写数据操作**：

- 首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上

- 页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发
- 通过一种称为Checkpoint的机制刷新回磁盘



配置参数：

- innodb_buffer_pool_instances， 缓冲池实例个数默认1，InnoDB 1.0.x版本开始使用，每个页根据哈希值平均分配到不同缓冲池实例中。减少数据库内部的资源竞争，增加数据库的并发处理能力。
- innodb_buffer_pool_size，缓冲池大小



命令参数：

- 可通过命令查看缓冲池大小 SHOW VARIABLES LIKE 'innodb_buffer_pool_size'
- 通过命令可查看缓冲池实例个数 SHOW VARIABLES LIKE 'innodb_buffer_pool_instances'
  

## 缓冲池的管理



缓冲池是一个很大的内存区域，其中存放各种类型的页。其实这么大的内存区域是通过LRU算法来进行管理的，而这里的LRU也是InnoDB 对传统的做了一些优化



#### LRU List

传统LRU：

- 最频繁使用的页在LRU列表的前端
- 最少使用的页在LRU列表的尾端
- 缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页



InnoDB改进版LRU：

- LRU列表中加入了midpoint位置
- 新读取到的页是放入到LRU列表的midpoint位置
- 默认配置下，该位置在LRU列表长度的5/8处。midpoint位置可由参数innodb_old_blocks_pct控制
- 引入innodb_old_blocks_time 参数，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端



#### Free List

- LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何的页。这时页都存放在Free列表中.



- 当需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入到LRU列表中。否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页



- 当页从LRU列表的old部分加入到new部分时，称此时发生的操作为page made young，而因为innodb_old_blocks_time的设置而导致页没有从old部分移动到new部分的操作称为page not made young
- 

- 可以通过命令SHOW ENGINE INNODBSTATUS来观察LRU列表及Free列表的使用情况和运行状态



#### Flush List

- LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷新回磁盘，二者互不影响

- 在LRU列表中的页被修改后，称该页为脏页（dirty page），即缓冲池中的页和磁盘上的页的数据产生了不一致。

- 这时数据库会通过CHECKPOINT机制将脏页刷新回磁盘，而Flush列表中的页即为脏页列表。

- 脏页既存在于LRU列表中，也存在于Flush列表中。







缓冲池中的页还可能会被分配给自适应哈希索引、Lock信息、InsertBuffer等页，而这部分页不需要LRU算法进行维护，因此不存在于LRU列表中。



## 重做日志缓冲(redo log buffer)



- InnoDB存储引擎首先将重做日志信息先放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件



- 重做日志缓冲一般不需要设置得很大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件



- 该值可由配置参数innodb_log_buffer_size控制，默认为8MB



- 可通过命令查看大小  SHOW VARIABLES LIKE  'innodb_log_buffer_size'



**什么时候刷新到磁盘？**

- Master Thread每一秒将重做日志缓冲刷新到重做日志文件

- 每个事务提交时会将重做日志缓冲刷新到重做日志文件

- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件

  