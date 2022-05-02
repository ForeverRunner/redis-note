# Redis有哪些事件
1. IO事件和时间事件以及相应的处理机制
2. Redis事件驱动框架循环流程对应的数据结构aeEventLoop结构体和初始化[ae.h](../../../src/ae.h)
   * aeFileEvent类型的指针*events，表示IO事件，之所以类型名称aeFileEvent是因为所有的IO事件都会用文件描述符进行标识
   * aeTimeEvent类型的指针*timeEventHead表示时间事件，即按一定时间周期触发的事件
   * aeFiredEvent类型的指针，并不是一类专门的事件类型，只是用来记录已触发事件对应的文件描述符信息
   * aeCreateEventLoop创建事件执行框架，只有setSize=maxclients+32+96
     * aeCreateEventLoop创建一个aeEventLoop结构体类型的变量
       * 给eventLoop成员变量分配内存空间：给IO事件数组和已触发事件数组分配相应的内存空间，给eventLoop的成员变量赋初值
     * 调用aeApiCreate函数（封装了操作系统提供的IO多路复用函数），linux下多路复用函数假如是epoll则调用epoll_create创建epoll实例，同时创建epoll_event结构的数组，数组大小等于setsize.创建的epoll实例描述符和epoll_event数组保存在aeApiStat结构体变量state中
     * aeCreateEventLoop会把所有网络IO事件对应文件描述符的掩码初始化为AE_NONE
3. IO事件处理[ae.h](../../../src/ae.h)
   * 可读事件：客户端读取事件
   * 可写事件：向客户端写入数据
   * 屏障事件：用来反转事件的处理顺序
4. IO事件创建
   * ```int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,aeFileProc *proc, void *clientData);```
   * 循环流程结构体、IO事件对应的文件描述符fd、事件类型掩码mask、事件处理回调函数、事件私有数据clientData
   * NOTE: eventLoop中有IO事件数组，这个数组的元素是aeFileEvent类型，每个数组元素对应记录了一个文件描述符(比如一个套接字)相关联的监听事件类型和回调函数
   * ae