> 每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解，此处不涉及详细的原理性解读，只作为一种抛砖引玉。真正的理解一定是你自我研究探索所收获的知识。



**每日一个知识点：一个 SQL 执行的很慢应该怎么分析**

#### 1、大多数情况下很正常，偶尔很慢，则有如下原因：

(1)、数据库在刷新脏页，例如 redo log 写满了需要同步到磁盘。

(2)、执行的时候，遇到锁，如表锁、行锁。



#### 2、这条 SQL 语句一直执行的很慢，则有如下原因。

(1)、没有用上索引：例如该字段没有索引；由于对字段进行运算、函数操作导致无法用索引。

(2)、数据库选错了索引。



