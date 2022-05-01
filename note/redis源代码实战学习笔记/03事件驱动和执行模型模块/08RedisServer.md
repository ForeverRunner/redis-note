# redisServer启动过程[server.c#main](../../../src/server.c)
1. 阶段一：基本初始化
   * 设置时区、随机数种子
   * 如果定义了REDIS_TEST宏定义且启动参数符合测试参数、执行测试程序

2. 阶段二：检查哨兵模式，并检查是否要执行RDB检测或AOP检测
   * 如果设置了哨兵模式，main函数会调用initSentinelConfig韩式，对哨兵模式进行参数设置。
   * 调用initSentinel()初始化哨兵模式运行的server
   * 如果运行的是redis-check-rdb检查rdb文件
   * 如果运行的是redis-check-aof检查aof文件

3. 阶段三：运行参数解析与设置
   * 解析命令行传入的参数，并且调用loadServerConfig函数，对命令行参数和配置文件中的参数进行合并处理
   * redis运行所需的各种参数都在[server.h](../../../src/server.h),主要分为7大类
     * 通用型参数
       * 定义了RedisServer运行的基本配置，比如server包含的数据库数量、server后台任务运行频率、server运行日志文件和级别。
       * redis运行时资源限制参数：server运行连接的最大客户端数量、server运行时运行的最大内存容量、客户端缓冲区最大大小
     * 数据结构参数
       * 定义了内存紧凑型数据结构的使用条件：基于压缩列表实现hash的最大哈希项个数或最大hash表大小
     * 网络参数
       * 定义了server的监听地址、端口、tcp缓冲区大小的参数
     * 持久化参数
       * 定义了aof和RDB执行时所需的各种配置，比如aof日志文件名、aof日志文件落盘方式、aof文件重写时是否打开日志同步写、rdb镜像文件压缩功能
     * 主从复制参数
       * 主从复制时的关键机制，包括主库信息、主从复制方式、主从复制积压缓冲区大小，从库是否只读等
     * 切片集群参数
       * 定义了切片集群运行时的关键机制：是否启动切片集群、切片集群是否支持故障切换、集群节点心跳超时时间等
     * 性能优化参数
       * server运行时的监控机制：慢查询的触发条件、延迟监控的触发阈值
       * redis提供的优化机制相关：惰性删除机制、主动内存碎片整理机制
   * Redis参数的设置方法
    * redis对运行参数的设置会经过三轮赋值：默认配置值、命令启动参数、配置文件配置值
    * 首先在main函数调用initServerConfig()函数，为各种参数设置默认值
       * 参数的默认值定义在server.h文件中以CONFIG_DEFAULT开头的宏定义变量
         * ```#define CONFIG_DEFAULT_MAX_CLIENTS 10000//最大客户端连接数```
         * ```#define CONFIG_DEFAULT_HZ  10 //server后台任务的默认运行频率```
         * ```#define CONFIG_DEFAULT_CLIENT_TIMEOUT 0//客户端超时事件，默认0，没有超时限制```
    * 命令行参数使用两个中划线"--"表示相应的参数名，redis对启动时的命令参数进行逐一解析，调用loadServerConfig函数进行第二三轮赋值
    *调用loadServerConfigFromString对配置项字符串中的每一个配置项进行匹配赋值

4. 阶段四：初始化server
   * 完成参数解析后，调用initServer函数，对server运行的各种资源进行初始化工作
   * initServer调用完后，再次判断当前server是否是哨兵模式，调用sentinelIsRunning函数，设置启动哨兵模式。否则调用loadDataFromDisk函数，从磁盘加载AOF或者RDB文件，以便恢复之前的数据
   * redisServer运行时需要对多种资源进行管理，在initServer函数中进行初始化
     * initServer创建链表来维护客户端和从库
     * 调用evictionPollAlloc函数采样生成用于淘汰的候选key集合
     * 调用resetServerStats()重置运行状态信息
   * 初始化数据库
   * 创建事件驱动框架，并开始启动端口监听接收外部请求
     * 针对每个监听的ip上可能发生的客户端连接，都创建了监听事件病设置相应的处理函数accrptTcpHandler函数 
   * server在main函数的最后，进入事件驱动循环机制
  

5. 阶段五:执行事件驱动框架
   * 调用aeMain函数进入事件驱动框架，开始循环处理各种触发的事件



   

