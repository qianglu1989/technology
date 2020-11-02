>世界上最快的捷径，就是脚踏实地
>
>文章收录：https://www.jiagoujishu.com
>
>
>
>每日一个知识点系列的目的是针对某一个知识点进行概括性总结，可在一分钟内完成知识点的阅读理解。
>
>此处不涉及详细的原理性解读，只作为一种抛砖引玉。
>
>真正的理解一定是你自我研究探索所收获的知识，加入组织带你一起进步成长。



## 一、存储引擎架构



主要有两方面：

- 后台线程
- 内存



<img src="/Users/luqiang/Downloads/公众号图片/innodb存储引擎架构.jpg" alt="innodb存储引擎架构" style="zoom:50%;" />



上图是InnoDB的存储引擎的体系架构，从图可见，InnoDB存储引擎有多个后台线程来对其进行内存池等操作的。

因为InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。





## 二、后台线程



后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据。

此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态



 ####  Master Thread

- 一个非常核心的后台线程
- 主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性
- 包括脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页的回收



####  IO Thread

- 引擎中大量使用了AIO（Async IO）来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调（call back）处理
- InnoDB 1.0版本之前共有4个IO Thread，分别是write、read、insert buffer和log IO thread
- 从InnoDB 1.0.x版本开始，read thread和write thread分别增大到了4个
- 查看版本 SHOW VARIABLES LIKE 'innodb_version'
- 查看线程数 SHOW VARIABLES LIKE 'innodb_%io_threads'
- 查看线程状态 SHOW ENGINE INNODB STATUS





#### Purge Thread

- 用于清理的线程，配置参数 innodb_purge_threads=1
- 事务被提交后，其所使用的undolog可能不再需要，因此需要PurgeThread来回收已经使用并分配的undo页
- 1.1版本之前，purge操作仅在InnoDB存储引擎的Master Thread中完成
- 1.1版本开始，purge操作可以独立到单独的线程中进行，以此来减轻MasterThread的工作
- 1.1版本中，即使将innodb_purge_threads设为大于1，InnoDB存储引擎启动时也会将其设为1
-  1.2版本开始，InnoDB支持多个Purge Thread，这样做的目的是为了进一步加快undo页的回收



#### Page Cleaner Thread

- 在InnoDB 1.2.x版本中引入的
- 其作用是将之前版本中脏页的刷新操作都放入到单独的线程中来完成
- 其目的是为了减轻原Master Thread的工作及对于用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能