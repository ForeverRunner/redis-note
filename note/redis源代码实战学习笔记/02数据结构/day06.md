# ziplist->quicklist->listpack进化
* ziplist利与弊
  * 节省内存开销
  * 不能保存过多元素，否则访问性能降低
  * 不能保存过大的元素，否则容易导致内存重新分配，甚至引发连锁更新的问题：每一项都要被重新分配内存空间，造成ziplist的性能降低
* ziplist的不足
  * 查找复杂度高
    * ziplist头尾元数据的大小固定，并且头部记录了最后一个元素的位置，当在ziplist中查找第一个或最后一个元素，复杂度为O(1),但是查找中间元素就得从头遍历，复杂度为O(n).当ziplist保存的是字符串，逐一判断元素的每一个字符，又增加了复杂度
    * 在ziplist保存hash或者sorted set数据时，在redis.conf中通过hash-max-ziplist-entries和zset-max-ziplist-entries控制保存在ziplist中的元素个数
    * 插入元素时，如果内存空间不够，ziplist还需要重新分配一块连续的内存空间，进一步引发连锁更新问题
  * 连锁更新风险
    * ziplist必须使用一块连续的内存空间来保存数据，当新插入一个元素是，ziplist就需要计算其所需空间大小，并申请相应的空间，代码见函数[__ziplistInsert](../../../src/ziplist.c)
      * __ziplistInsert函数首先会计算获得当前ziplist的长度，通过ZIPLIST_BYTES宏定义完成，同时还声明reqlen记录插入元素后所需新增空间
      * __ziplistInsert判断当前要插入的位置是否是列表的末尾
        * 如果不是末尾，获取位于当前插入位置元素的prevlen和prevlensize
        * 在ziplist中每一个元素都会记录其前一项的长度prevlen，为了节省内存开销，ziplist会使用不同的空间记录prevlen,这个prevlen空间大小就是prevlensize
      * 在计算插入元素所需空间时，__ziplistInsert分别计算prevlen、encoding、data三部分
        * 1. 计算实际插入元素的长度：如果是整数：按照不同整数的大小计算encoding和data各自所需空间，如果是字符串:先把字符串长度记录为所需的新增空间大小
        * 2. 调用zipStorePrevEntryLength将插入位置元素的prevlen也计算到所需空间中
        * 3. 调用zipStoreEntryEncoding函数，根据字符串的长度，计算相应encoding的大小
        * 4. 调用zipPrevLenByteDiff函数，判断插入位置元素的prevlen和实际所需prevlen大小
          * 如果nextdiff大于0表明插入位置元素的空间不够，需要新增nextdiff大小的空间，以便表村新的prevlen,然后__ziplistInsert函数在新增空间是，调用ziplistResize函数重新分配ziplist所需的空间
    * ziplistResize函数的实现：会调用zrealloc函数完成内存分配，如果往ziplist频繁插入过多数据的话，可能引起多次内存分配，对性能造成影响
* quicklist的设计与实现[quicklist.c](../../../src/quicklist.c)和[quicklist.h](../../../src/quicklist.h)
  * 一个quicklist就是一个链表，而链表中的每个元素又是一个ziplist
    * 首先quicklist元素的定义quicklistNode：
    ```c 
        typedef struct quicklistNode {
            struct quicklistNode *prev;  /*指向前序节点的指针*/
            struct quicklistNode *next;  /*指向后续节点的指针*/
            unsigned char *zl;           /*指向ziplist的指针zl*/
            unsigned int sz;             /* ziplist size in bytes */
            unsigned int count : 16;     /* count of items in ziplist */
            unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
            unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
            unsigned int recompress : 1; /* was this node previous compressed? */
            unsigned int attempted_compress : 1; /* node can't compress; too small */
            unsigned int extra : 10; /* more bits to steal for future usage */
        } quicklistNode;
    ```
    * quicklist的定义
    ```C 
        typedef struct quicklist {
            quicklistNode *head;        /*指向头节点*/
            quicklistNode *tail;        /*指向尾节点*/
            unsigned long count;        /* total count of all entries in all ziplists */
            unsigned long len;          /* number of quicklistNodes */
            int fill : 16;              /* fill factor for individual nodes */
            unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
        } quicklist;
    ```
    * quicklist插入元素流程：通过_quicklistNodeAllowInsert计算插入元素后的大小new_sz：quicklistNode的当前大小(node->sz)+插入元素的大小sz+插入元素后ziplist的prevlen占用大小，然后判断new_sz是否不超过8KB,或者单个ziplist里的元素个数是否满足要求，如果满足就在当前的quicklistNode中插入新的元素，否则会新建一个quicklistNode保存新插入的元素
* listpack设计与实现[listpack.c](../../../src/listpack.c)和[listpack.h](../../../src/listpack.h)及[listpack_malloc.h](../../../src/listpack_malloc.h)
  * 用一块连续的内存空间来紧凑地保存数据，是用了多种编码方式来表示不同长度的数据
  * LP_HDR_SIZE=32位总长度+16位置元素个数
  * LP_EOF(结束标志)=1字节，存储的默认值为255
  * 如何避免连锁更新
    * 因为listpack每个列表项目只是记录当前项目的长度，在新增或者修改元素时，只会涉及每个列表项自己的操作，不会影响后续列表项的长度变化，避免了连锁更新