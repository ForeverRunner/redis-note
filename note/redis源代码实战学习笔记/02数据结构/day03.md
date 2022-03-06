# Hash
* 概述
  * 在数据库系统中，Hash表被用来辅助SQL查询。对于Redis数据库来熟，Hash既是一种值类型，同时Redis使用一个全局的Hash表来保存所有的KV对，从而既满足应用存取Hash结构数据需求，又提供快速查询功能
* 遇到的问题
  * 随着数据量的增加，性能收到哈希冲突和rehash开销的影响
* Redis时如果解决的
  * 针对Hash冲突，采用了链式哈希，在不扩容哈希表的前提下，将具有相同hash值的数据连接起来，以便这些数据在表中仍然可以被查询到
  * 对于rehash开销，采用渐进式rehash设计，进而缓解了rehash操作带来的额外开销对系统的性能影响
* redis如何实现链式哈希
  * 用一个链表把映射到Hash表同一桶中的Key给连接起来，[dict.h](../../../src/dict.h)定义了Hash表结构、哈希项，以及Hash表的各种操作函数，而[dict.c](../../../src/dict.c)文件包含了Hash表各种操作的实现代码
  * 在**dict.h**中,Hash表被定义为一个二维数组```(dictEntry **table)```这个数组的每个元素是指向哈希项的指针
  * 在dictEntry结构体中，键值对的值是由一个**联合体v**定义的，包含了指向实际值的指针*val,还包含了无符号的64位整数、有符号的64位整数、double的值。这是一种节省内存的开发技巧：当值为整数或双精度浮点数是，其本身是64位，就可以不用指针指向，可以直接存在键值对的结构体中

* redis如何实现rehash
  * rehash操作就是扩大Hash表空间
  * redis准备了两个哈希表，用于rehash时交替保存数据[dict.h](../../../src/dict.h)**#dict**
    ```c 
      typedef struct dict{
        dictht ht[2];//两个hash表交替使用，用于rehash操作
        long rehashidx;//Hash表是否在进行rehash的标识，-1表示没有进行rehash
      }dict
    ```
  * 在正常服务请求阶段，所有的KV对写入哈希表ht[0]中
  * 当进行rehash时，KV对被迁移到ht[1]中
  * 当迁移完成后，ht[0]的空间会被释放，并把ht[1]的地址赋值给ht[0],ht[1]的表大小设置为0，这样一来又回到了正常服务请求阶段，ht[0]接收和服务请求，ht[1]作为下一次rehash时的迁移表
  * rehash时要解决的问题
    * 什么时候触发rehash，Redis用来判断是否出发rehash的函数是_dictExpandIfNeeded,_dictExpandIfNeeded定义了三个扩容条件:
      * ht[0]的大小为0,扩容为初始大小4
      * ht[0]承载的元素个数超过了ht[0]的大小，同时Hash表可以进行扩容
      * ht[0]承载的元素个数是ht[0]的大小的dict_force_resize_ratio倍，其中dict_force_resize_ratio的默认值是5
      ```c
        /* Expand the hash table if needed */
        static int _dictExpandIfNeeded(dict *d)
        {
            /* Incremental rehashing already in progress. Return. */
            if (dictIsRehashing(d)) return DICT_OK;
            //如果Hash表为空，将Hash表扩容为初始大小4
            /* If the hash table is empty expand it to the initial size. */
            if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

            /* If we reached the 1:1 ratio, and we are allowed to resize the hash
            * table (global setting) or we should avoid it but the ratio between
            * elements/buckets is over the "safe" threshold, we resize doubling
            * the number of buckets. */
            //如果Hash表中的元素个数超过其当前大小，并且可以进行扩容，或者hash表中的元素个数已经是当前大小的5倍
            if (d->ht[0].used >= d->ht[0].size &&
                (dict_can_resize ||
                d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
            {
                return dictExpand(d, d->ht[0].used*2);
            }
            return DICT_OK;
        }
      ```
      * 负载因子**loadFactor**（d->ht[0].used/d->ht[0].size)；判断是否进行rehash的条件，就是看loadFactor是否大于等于1和是否大于5,当loadFactor大于等于1时，还会判断**dict_can_resize**这个变量值。这个变量值是dictEnableResize和dictDisableResize两个函数中设置值，作用分别是启用和精致哈希表执行rehash的功能.这个函数又封装到[server.c](../../../src/server.c)**updateDictResizePolicy**函数中，用来启用或者禁用rehash扩容功能的。调用dictEnableResize的条件是：当前没有RDB子进程，并且也没有aof子进程。对应Redis没有执行RDB快照和没有进行AOF重写的场景
      * _dictExpandIfNeeded是被_dictKeyIndex调用的，而_dictKeyIndex是被dictAddRaw调用，然后dictAddRaw会被三个函数调用，调用关系如下图:[img](../02数据结构/img/_dictExpandIfNeeded调用.drawio)
        * dictAdd：用来往Hash表中添加一个KV对
        * dictReplace:用来往Hash表中添加一个KV对，如果Key存在时，修改V
        * dictAddorFind:直接调用
    * rehash扩容多大:通过调用**dictExpand**函数来完成的,```c int dictExpand(dict *d,unsigned long size);``` 参数有两个，要扩容的Hash表，要扩容到的容量
      * 如果当前表的已用空间大小为size,就将表扩容到size*2的大小，而在dictExpand函数中，具体执行是由_dictNextPower函数完成的
    * 渐进式rehash如何执行
      * 为什么要实现渐进式Hash
        * Hash表在执行rehash时，由于hash表空间扩大，原本映射到某一位置的key可能会被映射到新的位置上就会出现很多key拷贝到新的位置。在拷贝的过程，redis主线程无法执行其他请求，会被阻塞，就会产生rehash开销。渐进式rehash的意思就是：redis不会一次性把当前Hash表中所有的key都拷贝到新的位置，而是分批拷贝，每次的key拷贝只拷贝hash表中一个buckey中的哈希项，这样每次key拷贝的时常有限，对主线程影响也就有限
      * 代码层面有两个关键函数dictRehash和_dictRehashStep
        * dictRehash函数的整体逻辑包括两部分[执行逻辑](../../redis源代码实战学习笔记/02数据结构/img/dictRehash.drawio)
          * 首先，该函数回执行一个循环，根据要进行拷贝的bucket的数量n，依次完成这些buckey内存所有key的迁移。如果ht[0]中的数据都已经迁移完成,键拷贝的循环也会停止执行
          * 在完成了n个bucket拷贝后，dictRehash函数第二部分逻辑，就是判断ht[0]表中数据是否都迁移完成，如果都迁移完成，ht[0]的空间会释放，然后将ht[1]赋值给ht[0]，以便其他部分的代码逻辑正常使用
          * 当ht[1]复制给ht[0]后，ht[1]的大小被重置为0，等待下一次rehash,全局哈希表中的rehashidx变量标为-1，标识rehash结束
        * _dictRehashStep函数实现了每次只对一个bucket进行rehash
          * 
  