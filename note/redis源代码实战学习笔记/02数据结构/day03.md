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
  * redis准备了两个哈希表，用于rehash时交替保存数据[dict.h](../../src/dict.h)**#dict**
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
    * 什么时候触发rehash，Redis用来判断是否出发rehash的函数是_dictExpandIfNeeded,_idctExpandIfNeeded定义了三个扩容条件
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
    * rehash扩容多大
      * 
    * rehash如何执行
  