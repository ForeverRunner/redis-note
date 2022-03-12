# 有序集合为何能同时支持点查询和范围查询
* 有序集合（sorted set）是redis中一种重要的数据类型，本身是集合类型，同时也支持集合中的元素带有权重，并按照权重排序
* 为什么sorted set既能支持高效的范围查询，同时还能以O（1）的复杂度获取元素权重值
  * 支持范围查询，主要因为核心屈居结构是跳表
  * 能够以常数复杂度获取元素权重，因为同时采用了Hash表进行索引
* sorted set的基本结构
  * 实现代码在[t_zset.c](../../../src/t_zset.c),结构定义在[server.h](../../../src/server.h)
    ```c
        typedef struct zset {
            dict *dict;
            zskiplist *zsl;
        } zset; 
    ```
  * 问题
    * 跳表或者hash表中各自保存了什么数据
    * 条表和hash表保存的数据如何保持一致
  * 跳表的设计与实现[server.h](../../../src/server.h)
    * 是一种多层的有序链表
    * 数据结构定义
    ```c
        typedef struct zskiplistNode {
            sds ele;//sorted set中的元素值
            double score;//元素权重值
            struct zskiplistNode *backward;//后项指针，为了便于从跳表的尾节点进行倒序查找，指向该节点的前一个节点
            //节点的level数组，保存每一层上的前项指针和跨度
            struct zskiplistLevel {
                struct zskiplistNode *forward;
                unsigned long span;//跨度，记录节点在某一层上的*forward指针和该指针指向的节点之间，跨越了level0上的几个节点
            } level[];
        } zskiplistNode;
    ```
    * 对于跳表的某个节点，可以把从头节点到该节点的查询路径上，各个节点在所查询层次上的*forward的指针跨度做一个累加，就是元素在跳表中的顺序。利用这个特点实现ZRANK,ZREVRANK
    * 跳表节点查询,最高层64层
      * 不需要逐一顺序查询跳表中的各个节点，可以利用level数组来加速查询
      * 跳表在比较节点时判断条件
        * 当查找的节点保存的元素权重，小于要查询的权重，继续访问该层的下一个节点
        * 当查找的节点保存的元素权重，等于要查询的权重，比较该节点保存的SDS数据是否比要查寻的sds数据小，如果小于要查询的元素数据，继续访问该层的下一节点
        * 当两个条件都不满足，访问下一层的指针，然后沿着下一层的指针继续查询
      * 查找、插入、更新、删除操作时都用到查询功能
    * 跳表节点层数设置  
      * 1. 每一层上的节点数是下一层节点数量的一半
        * 查找类似于二分查找，复杂度为O(logN)
        * 为了维持相邻两层上的节点数比例为2:1，一旦有节点插入或删除，需要调整姐弟爱你的层数

      * 为了避免删除或新增带来的调整，采用随机生成每个几点的层数，由[zslRandomLevel](../../../src/t_zset.c)函数决定，
        * zslRandomLevel函数会初始化层数为节点的最小层数1，然后生成随机数，如果随机数小于0.25，层数+1，每增加一层的概率不会超过25%
* 哈希表和跳表的组合使用
  * 当创建一个zset时候，使用dictCreate创建zset的哈希表，zslCreate创建跳表
  * 添加元素时候，zsetAdd函数判定Sorted set使用ziplist还是skiplist的编码方式
  * 如果zsetAdd通过dictFind函数发现元素已经存在，zsetAdd判断是否要增加函数的权重值

