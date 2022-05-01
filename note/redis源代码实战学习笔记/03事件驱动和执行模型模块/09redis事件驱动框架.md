# redis使用的epoll网络模型
1. 基本的网络模型
    * 调用socket函数，创建一个套接字。通常把这个套接字称为主动套接字
    * 调用bind函数，将主动套接字与当前服务器和监听端口进行绑定
    * 调用listen函数，将主动套接字转化为监听套接字，开始监听客户端连接
    * 为了及时收到连接请求，可以运行一个循环流程，使用accept函数，用于接收客户端连接
      * accept函数是阻塞函数：一旦有客户端连接请求到达，accept将不再阻塞，而是处理连接请求，和客户端建立连接，并返回已连接套接字
2. redis的主执行流程是由一个线程在执行，无法使用多线程的方式来提升并发处理能力，所以使用的是**IO多路复用功能**
3. IO多路复用的三个问题
    * 多路复用机制会监听套接字的哪些事件
    * 多路复用机制可以监听多少个套接字
    * 当有套接字就绪时，多路复用机制要如何找到就绪的套接字
4. select机制与使用[ae_select.c](../../../src/ae_select.c)
    * ```int select(int, fd_set * __restrict, fd_set * __restrict,fd_set * __restrict, struct timeval * __restrict)```
    * linux针对每一个套接字都会有一个文件描述符，也就是一个非负整数，用来唯一标识该套接字。
    * 在多路复用机制的函数中，linux通常会用该文件描述符作为参数。有了文件描述符，函数也就能找到对应的套接字，进而进行监听、读写操作
    * select函数的参数__readfds、__writefds和__exceptfds表示的是：被监听的描述符集合，主要分为3类事件
      * 读数据事件__readfds
      * 写数据事件__writefds
      * 异常事件__exceptfds
    * fds_set是个包含1024/32个元素(大小为32位)的数组，每个元素是32位，每一位可以用来表示一个文件描述符的状态。select函数对每一个描述符集合，都可以监听1024个描述符
    * 不足
      * select函数对单个进程监听的文件描述符数量是有限的，
      * select函数返回后，需要遍历描述符集合，才能找到具体就绪的描述符
5. poll机制与使用[poll.h](poll.h)
    * ```int poll(struct pollfd *__fds, nfds_t __nfds, int __timeout)```
    * *__fds是pollfd结构体数组、__nfds表示*__fds元素的个数,__timeout表示poll函数阻塞的超时时间
    ```struct pollfd {
        int     fd;//进行监听的描述符
        short   events;//要监听的事件类型
        short   revents;//实际发生的事件类型
    };
    ```
    * 使用poll函数分为3步
      * 创建pollfd数组监听套接字，并进行绑定
      * 将监听套接字加入pollfd数组，并设置其监听读事件，也就是客户端的连接请求
      * 循环调用poll函数，检测pollfd数组中是否有就绪的描述符
        * 如果连接套接字就绪，表明有客户端连接，可调用accept接受连接，并创建已连接套接字，并将其加入pollfd数组，并监听读事件
        * 如果是已连接套接字就绪，表明客户端有读写请求，调用recv/send函数处理读写请求
6. epoll机制与使用[ae_poll.c](../../../src/ae_epoll.c)

     
