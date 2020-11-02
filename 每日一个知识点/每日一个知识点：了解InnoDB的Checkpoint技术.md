>每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解。
>
>此处不涉及详细的原理性解读，只作为一种抛砖引玉。
>
>真正的理解一定是你自我研究探索所收获的知识，加入组织带你一起进步成长。

一句话概括，**Checkpoint技术就是将缓存池中脏页在某个时间点刷回到磁盘的操作**



## 遇到的问题 ？

 

 <img src="https://imgkr2.cn-bj.ufileos.com/ce584b94-e75d-40ed-9481-768d126435a2.jpg?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=V0vzPbwrW5%252Fjt5F17hplb9qy01I%253D&Expires=1603786144" width = "500" height = "200" alt="图片名称" align=center />





都知道缓冲池的出现就是为了解决CPU与磁盘速度之间的鸿沟，免得我们在读写数据库时还需要进行磁盘IO操作。有了缓冲池后，所有的页操作首先都是在缓冲池内完成的。



如一个DML语句，进行数据update或delete 操作时，此时改变了缓冲池页中的记录，此时因为缓冲池页的数据比磁盘的新，此时的页就叫做脏页。



不管怎样，总会后的内存页数据需要刷回到磁盘里，这里就涉及几个问题：

- 若每次一个页发生变化，就将新页的版本刷新到磁盘，那么这个开销是非常大的
- 若热点数据集中在某几个页中，那么数据库的性能将变得非常差
- 如果在从缓冲池将页的新版本刷新到磁盘时发生了宕机，那么数据就不能恢复了





## Write Ahead Log（预写式日志）



WAL策略解决了刷新页数据到磁盘时发生宕机而导致数据丢失的问题，它是关系数据库系统中用于提供原子性和持久性（ACID 属性中的两个）的一系列技术。



#### WAL策略核心点就是

`redo log`，每当有事务提交时，先写入 `redo log`（重做日志），在修改缓冲池数据页，这样当发生掉电之类的情况时系统可以在重启后继续操作



#### WAL策略机制原理

InnoDB为了保证数据不丢失，维护了redo log。在缓冲池的数据页修改之前，需要先将修改的内容记录到redo log中，并保证redo log早于对应的数据页落盘，这就是WAL策略。

当故障发生而导致内存数据丢失后，InnoDB会在重启时，通过重放redo log，将缓冲池数据页恢复到崩溃前的状态。





## Checkpoint



按理说有了WAL策略，我们就可以高枕无忧了。但其问题点又出现在redo log上面：

- redo log 不可能是无限大的，不能没完没了的存储我们的数据等待一起刷新到磁盘
- 在数据库怠机恢复时，如果redo log 太大的话恢复的代价也是非常大的



所以为了解决脏页的刷新性能，脏页应该在什么时间、什么情况下进行脏页的刷新就用到了Checkpoint技术。



#### Checkpoint 的目的



**1、缩短数据库的恢复时间**

当数据库怠机恢复时，不需要重做所有的日志信息。因为Checkpoint前的数据页已经刷回到磁盘了。只需要Checkpoint后的redo log进行恢复就好了。



**2、缓冲池不够用时，将脏页刷新到磁盘**

当缓冲池空间不足时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘。



**3、redo log不可用时，刷新脏页**




![](https://imgkr2.cn-bj.ufileos.com/5fdfe31d-3743-44e3-bfbc-2093214fe1bd.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=WzaXYNsyY6L8XcYAArd5V%252FdSUp0%253D&Expires=1603786161)


如图redo log 的不可用是因为当前数据库对其设计都是循环使用的，所以其空间并不是无限大。

当redo log被写满, 因为此时系统不能接受更新, 所有更新语句都会被堵住。

此时必须强制产生Checkpoint需要将 write pos 向前推进，推进范围内的脏页都需要刷新到磁盘





#### Checkpoint 的种类



Checkpoint发生的时间、条件及脏页的选择等都非常复杂。

Checkpoint 每次刷新多少脏页到磁盘？

Checkpoint每次从哪里取脏页？

Checkpoint 什么时间被触发？



**面对上面的问题，InnoDB存储引擎内部为我们提供了两种Checkpoint：**

- Sharp Checkpoint

  发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，参数innodb_fast_shutdown=1

  

- Fuzzy Checkpoint

  InnoDB存储引擎内部使用这种模式,只刷新一部分脏页，而不是刷新所有的脏页回磁盘





**FuzzyCheckpoint发生的情况**

- Master Thread Checkpoint

   差不多以每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘。

   这个过程是异步的，即此时InnoDB存储引擎可以进行其他的操作，用户查询线程不会阻塞

  

- FLUSH_LRU_LIST Checkpoint

   因为LRU列表要保证一定数量的空闲页可被使用，所以如果不够会从尾部移除页，如果移除的页有脏页，就会进行此Checkpoint。  

   5.6版本后，这个Checkpoint放在了一个单独的Page Cleaner线程中进行，并且用户可以通过参数innodb_lru_scan_depth控制LRU列表中可用页的数量，该值默认为1024

  

- Async/Sync Flush Checkpoint

  指的是redo log文件不可用的情况，这时需要强制将一些页刷新回磁盘，而此时脏页是从脏页列表中选取的
  
  5.6版本后不会阻塞用户查询



- Dirty Page too much Checkpoint
  即脏页的数量太多，导致InnoDB存储引擎强制进行Checkpoint。

  其目的总的来说还是为了保证缓冲池中有足够可用的页。

  其可由参数innodb_max_dirty_pages_pct控制,比如该值为75，表示当缓冲池中脏页占据75%时，强制进行CheckPoint



## 总结


- 因为CPU和磁盘间的鸿沟的问题，从而出现缓冲池数据页来加快数据库DML操作

- 因为缓冲池数据页与磁盘数据一致性的问题，从而出现WAL策略（核心就是redo log）

- 因为缓冲池脏页的刷新性能问题，从而出现Checkpoint技术



InnoDB 为了提高执行效率，并不会每次DML操作都和磁盘交互进行持久化。而是通过Write Ahead Log 先策略写入redo log保证事物的持久化。

对于事物中修改的缓冲池脏页，会通过异步的方式刷盘，而内存空闲页和redo log的可用是通过Checkpoint技术来保证的。