InnoDB引擎有几个重点特性，为其带来了更好的性能和可靠性：

-  插入缓冲（Insert Buffer）
-  两次写（Double Write）
-  自适应哈希索引（Adaptive Hash Index）
-  异步IO（Async IO）
-  刷新邻接页（Flush Neighbor Page）



今天我们的主题就是 `两次写（Double Write）`,由于InnoDB引擎底层数据存储结构式B+树，而对于索引我们又有聚集索引和非聚集索引。