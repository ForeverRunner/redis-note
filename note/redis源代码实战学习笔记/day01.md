# Redis源代码阅读

1. 为啥阅读源码

    * 进一步掌握redis的实现细节、便于排查性能故障问题
    * 从原理到源码，学习源码阅读的方法，培养源码习惯、掌握学习主动权
    * 学习良好的编程规范、写出高质量的代码
    * 举一反三，学习计算机系统设计思想，实现职业能力进阶
        * 掌握单机KV数据库的关键技术：
            * 包括支持高性能访问的数据结构
            * 支持高效空间利用率的数据结构
            * 网络服务器高并发通信
            * 高效线程执行模型
            * 内存管理
            * 日志机制
        * 掌握分布式系统的关键技术
            * 分布式系统主从复制机制
            * 可扩展集群数据切片与放置技术
            * 可扩展集群通信机制
        * 掌握计算机系统设计思想
            * 数据结构
                * 数据结构内存优化
                * 高性能数据结构设计
            * 高并发网络通信
                * IO复用
                * 事件驱动框架
            * 线程模型
                * 异步线程任务
                * 线程通信
            * 内存管理
                * 惰性删除
                * LRU/LFU替换算法与优化
            * 主从复制
                * 数据主从同步
                * 网络故障容错
            * 切片集群
                * Gossip协议开发
                * 数据放置
            * 日志

2. 目录结构
    * deps 第三方依赖包，可以独立于redis存在和发展
        * hiredis redis的C语言版本客户端
        * jemalloc 内存分配器代码
        * lua lua脚本支持
        * linenoise 命令行解析
    * src
        * modules示例代码
        * 各功能模块代码
    * 配置文件
        * redis.conf服务配置文件
        * sentinel哨兵配置文件
    * utils目录
        * create-cluster子目录，创建集群工具代码
        * hashtable子目录，rehash过程可视化代码
        * hyperloglog子目录，hyperloglog误差率计算和展示代码
        * lru子目录，LRU算法效果展示代码
    * test目录
        * 测试代码子目录
            * unit单元测试代码
            * cluster集群功能测试代码
            * sentinel哨兵功能测试代码
            * integration主从赋值功能测试代码
        * 测试支撑代码子目录
            * assets
            * helpers
            * modules
            * support

3. 代码路径
    * 服务器实例
        * server.c/server.h:Redis运行时是一个网络服务器实例，因此需要有代码实现服务器实例的初始化和主题控制流程
        * 对于一个网络服务器，还需要提供网络通信功能，Redis使用了事件驱动机制的网络通信框架，涉及的代码：ae.h/ae.c,ae_epoll.c,ae_evport.c,ae_kqueue.c,ae_select.c
        * 与网络通信相关的功能：包括底层的TCP网络通信和客户端实现，Redis对TC网络通信的Socket连接、设置等操作进行了封装、封装后的函数在anet.h/anet.c中
        * 客户端在redis运行过程中也会被广泛使用，比如实例返回对去的数据，主从复制时主从库间传输数据，RedisCluster的切片实例通信等都会用到客户端，Redis将客户端的创建、消息回复等功能实现在了networking.c
    * 数据库数据类型操作
        * 数据类型(底层数据结构：对应数据类型：对应源码文件)
            * SDS:String:sds.h:sdc.h/sdsalloc.h
            * 双向链表：List:adlist.h/adlist.c
            * 压缩链表：List,Hash,Sorted Set:ziplist.h/ziplist.c
            * QuickList:List,Hash,Sorted Set:quicklist.h/quicklist.c
            * 整数列表:Set:intset.h/intset.c
            * Zipmap:Hash:zipmap.h/zipmap.c
            * 哈希表:Hash:dict.h/dict.c
            * HyperLogLog:HyperLogLog:hyperloglog.c
            * GeoHash:Geo:geo.h/geo.c,geohash.h/geohash.c,geohash_helper.h/geohash_helper.c
            * 位图:位图：bitops.c
            * Stream:时序数据：stream.h/t_stream.c
        * 数据库KV对的新增、查询、修改、删除等操作接口封装在db.c
        * Redis优化内存是从：内存分配、内存回收、数据替换
            * 内存分配方面
                * Redis支持不同的内存分配器，包括glibc库提供的默认分配器tcmalloc、第三方库提供的jemalloc,Redis把对内存分配器的封装实现在了zmalloc.h/zmalloc.c
            * 内存回收上
                * 支持设置过期key，并针对过期key可以使用不同删除策略，代码实现在expire.c;为了避免大量key删除回收内存，会对系统性能产生影响，在lazyfree.c实现了异步删除的功能
            * 数据替换
                * 如果内存满了，按照一定规则清除不需要的数据，实现的策略有多种，包括LRU/LFU，封装在evict.c中
    * 高可靠性和高可扩展性（体现在对数据持久化，实现了主从复制机制，从而提供了故障恢复功能）
      * 数据持久化实现（内存快照RDB和AOF日志）
        * 内存快照，rdb.h/rdb.c
        * AOF日志,aof.c
        * 使用AOF或者RDB恢复时，RDB和AOF文件可能会因为所在服务器宕机而未能完整保存，所以恢复时对这两类文件进行完整性校验，封装在：redis-check-rdb.c和redis-check-aof.c
      * 主从复制功能实现
        * 主从复制功能在replication.c
        * 在进行恢复时，主要依赖哨兵机制，实现在sentinel.c
      * 集群实现高可扩展
        * 通过Redis Cluster实现，在cluster.h/cluster.c
    * 辅助功能
      * 系统运维功能
        * 查看不同操作的延迟产生来源，latency.h/latency.延迟监控
        * 查看运行过慢的操作命令，slowlog.h/slowlog.c实现慢命令记录功能
        * 性能评测功能，redis-benchmark.c