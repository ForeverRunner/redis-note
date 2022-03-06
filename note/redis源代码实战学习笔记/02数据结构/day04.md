* 内存友好的数据结构如何设计
  * Redis通过两大方面的技术来提升内存使用效率的：
    * 数据结构的优化设计与使用
    * 内存数据按照一定规则淘汰
  * Redis数据结构在面向内存使用效率方面的优化
    * 内存友好的数据结构设计，有三种数据结构针对内存使用效率做了设计优化，分别是简单动态字符串SDS，压缩列表ziplist和整数集合intset
      * SDS的内存友好设计
        * sds设计了不同类型的结构头，这些不同类型的结构头可以适配不同大小的字符串，从而避免了内存浪费
        * 在保存较小字符串时，使用了嵌入式字符串的设计方法，这种方法避免了给字符串分配额外的空间，而是可以让字符串直接保存在redis的基本数据对象结构体中
        * redisObject结构体与位域定义方法
          * redisObject是在[server.h](../../../src/server.h)中定义的，主要功能是用来保存键值对中的值，这个结构体一共定义了4个元数据和一个指针
            * type: redisObject的数据类型，是应用程序在redis中保存的数据类型，包括String，List，Hash等
            * encoding：redisObject的编码类型，是Redis内部实现各种数据类型所用的数据结构
            * lru：redisObject的LRU时间
            * refcount：redisObject的引用计数
            * ptr指向值的指针

        * 变量后使用冒号和数值的定义方法，是C语言中位域定义方法，可以用来有效滴节省内存开销。
          * 适用的场景是，当一个变量占用不了一个数据类型的所有bits时，就可以使用位域定义方法，把一个类型中的bits划分成多个位域，每个位域占一定的bits数，这样一个数据类型的所有bits就可以定义多个变量了，从而也有效的节省了内存开销。
          * 例如：对于type，encoding,lru三个变量，数据类型都是unsigned。已知一个unsigned类型时4字节，但是这三个变量分别占用一个unsigned的4bit、4bit、24bit。因此节省了8字节内存
            ```C
                typedef struct redisObject {
                    unsigned type:4;//redisObject的数据类型，占用4bit
                    unsigned encoding:4;//redisObject的编码类型，占用4bit
                    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                                            * LFU data (least significant 8 bits frequency
                                            * and most significant 16 bits access time). */
                    int refcount;//redisObject的引用计数，4个字节
                    void *ptr;//指向值的指针，8个字节
                } robj;
            ```
        * 嵌入式字符串
          * 创建一个String类型的值，Redis会调用createStringObject函数，来创建相应的redisObject，而这个redisObject中的ptr指针，就会指向sds数据结构[如图](../02数据结构/img/day04.drawio)
          * createStringObject函数会根据要创建的字符串长度，决定具体调用哪个函数来完成创建
            * 对于createRawStringObject函数：创建string类型的值的时候，会调用createObject函数
              * createObject函数主要是用来创建Redis的数据对象的。因为Redis的数据对象有很多类型，比如String,List,Hash等，createObject函数两个参数中：一个是用来表示所要创建的数据对象类型，而另一个是指向数据对象的指针
            * 创建字符串的流程[如图](../02数据结构/img/day04.drawio),在创建普通字符串时，需要分别给redisObject和SDS分配一次内存，既带来了内存分配的开销，也导致内存碎片。因此当字符串小于等于44字节时，就使用嵌入式字符串
            * [createEmbeddedStringObject](../../../src/object.c):使用一块连续的内存空间，同时保存redisObject和SDS结构，内存分配只有一次，也避免了内存碎片```c robj *createEmbeddedStringObject(const char *ptr,size_t len)```,创建过程见[字符串创建流程](../02数据结构/img/day04.drawio)

        * 压缩列表和整数集合的设计
          * List
    * 内存友好的数据使用方式