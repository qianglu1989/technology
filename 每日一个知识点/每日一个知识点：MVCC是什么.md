> 每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解，此处不涉及详细的原理性解读，只作为一种抛砖引玉。真正的理解一定是你自我研究探索所收获的知识。



MVCC，Multi-Version Concurrency Control，多版本并发控制 

数据库有四种隔离级别，通过MVCC在每种查询下的情况也是不一样的。

在每次insert 、update 、delete都会生成一个事物id,事物id 是递增的。聚簇索引记录中都包含两个必要字段 trx_id 、roll_pointer ,每次修改都会生成undo日志（insert没有，因为没历史）。

这里有几个主要就是概念就是版本链、readview。

**版本链：**

对于使用`InnoDB`存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列（`row_id`并不是必要的，我们创建的表中有主键或者非NULL唯一键时都不会包含`row_id`列）

- `trx_id`：每次对某条聚簇索引记录进行改动时，都会把对应的事务id赋值给`trx_id`隐藏列。

- `roll_pointer`：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到`undo日志`中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

  

**Readview:**

 (每种隔离级别也不太一样)，主要包含当前系统中还有哪些活跃的读写事务，把它们的事务id放到一个列表中，把这个列表命名为为`m_ids`，根据版本链trx_id 和m_ids 的比较来进行版本选择（具体规则可以自己查查）

- READ COMMITTED（用到版本链）：每次读取数据前都生成一个ReadView（在每一个select操作前都生成ReadView）
- REPEATABLE READ（用到版本链）：在第一次读取数据时生成一个ReadView（只在第一次进行普通`SELECT`操作前生成一个`ReadView`）
- READ UNCOMMITTED： 直接读取记录的最新版本
- SERIALIZABLE：使用加锁的方式来访问记录

 完。。。