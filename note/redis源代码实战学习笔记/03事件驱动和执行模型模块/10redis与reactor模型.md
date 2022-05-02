# redis如何实现reactor模型
1. reactor模型是什么
   * 是一种网络服务器用来处理高并发网络io请求的编程模型
     * 三类处理事件：连接事件、读事件、写事件
     * 三个关键角色：reactor、acceptor、handler
   * 事件与关键角色的关系
     * 当客户端连接服务器时，对应服务端的连接事件
     * 建立连接后，客户端与服务端发送读请求，以便读取数据。服务器端在处理读请求时，需要向客户端写回数据，这对应了服务器端的写事件
     * 无论客户端给服务器端发送读或写请求，服务器端都需要从客户端读取请求内容，读或写请求的读取对应了服务器端的读事件
   * 三类事件处理的对象
     * 连接事件由acceptor来处理，负责接收连接；acceptor在接收连接后会创建handler，用于网络连接上对后续对鞋事件的处理
     * 其次读写事件由handler处理
     * 在高并发场景中，连接事件、读写事件会同时发生，需要有一个角色专门监听和分配事件，这就是reactor负责的事情。
       * 当有连接请求时，reactor将产生的连接事件交由acceptor处理
       * 当有读写请求时，reactor将读写事件交由handler处理
   * 事件驱动框架
     * 事件初始化
       * 在服务器程序启动时就执行，作用时创建需要监听的事件类型，以及该事件对应的handler
     * 事件捕获、分发和处理循环
   * reactor模型的基本工作机制：客户端的不同请求会在服务器端触发连接、读、写三类事件，这三类事件的监听、分发、处理是由reactor、acceptor、handler三类角色完成，这三类角色会通过事件驱动框架来实现交互和事件处理
2. redis代码如何与reactor模型相对应
   * 实现文件[ae.c](../../../src/ae.c),对应头文件[ae.h](../../../src/ae.h)
   * redis在[ae.h](../../../src/ae.h)定义了事件的数据结构、框架主循环函数、事件捕获分发函数、事件和handler注册函数
   * 事件数据结构的定义[aeFileEvent](../../../src/ae.h)
     * redis定义了IO事件和时间事件，分别对应客户端发送的网络请求和redis自身的周期性操作
     ```c
          typedef struct aeFileEvent {
              int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
              aeFileProc *rfileProc;//读事件处理还书，分发事件后调用
              aeFileProc *wfileProc;//写事件处理函数，也就是handler
              void *clientData;//指向client客户端私有数据的指针
          } aeFileEvent;
     ``` 
     * mask用来表示事件类型的掩码，主要有AE_READABLE|AE_WRITEABLE|AE_BARRIER三类事件
     * [主要函数](../../../src/ae.h)
       * 框架主循环函数aeMain
         * 在server.c中服务器程序初始化完成后、开始执行
       * 负责事件捕获与分发的aeProcessEvents  
         * 主要功能包括：捕获事件、判断事件类型和调用具体的事件处理函数，从而实现事件的处理
       * 负责事件和handler注册的aeCreateFileEvent
         * 事件注册：aeCreateFileEvent函数
           * 当redis启动后在initServer函数中进行初始化，调用aeCreateFileEvent用于注册要监听的事件及相应事件处理函数
           * initServer根据启用的ip端口个数为每个ip端口上的网络事件调用aeCreateFileEvent，创建对AE_READABLE事件监听，并且注册AE_READABLE事件的处理handler也就是acceptTcpHandler
           * aeCreateFileEvent如何实现事件和处理函数的注册
             * linux上提供了epoll_ctl API用于增加新的观察事件，redis在此基础上封装了aeApiAddEvent函数，对epoll_ctl调用
     