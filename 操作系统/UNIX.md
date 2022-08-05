# UNIX

## UNIX IO

### IO模型

五种IO模型：

- 阻塞式IO
- 非阻塞式IO
- IO复用
- 信号驱动IO
- 异步IO

#### 阻塞式IO

![image-20210317002155905](http://qiniu.qraffa.cn/image-20210317002155905.png)

当进程调用recvfrom函数时，会阻塞等待，直到数据报达到且复制到进程的缓冲区或者发生错误，才会返回。

从调用recvfrom函数到该函数返回的过程，都是阻塞的，当recvfrom成功返回后，进程才能开始处理数据报。

#### 非阻塞式IO

![image-20210317002614117](http://qiniu.qraffa.cn/image-20210317002614117.png)

当进程调用recvfrom函数时，如果没有数据报准备好，内核不会阻塞等待，而是立即返回一个EWOULDBLOCK错误，如果已有数据报准备好时，将数据报复制到进程缓冲区，并成功返回。

因此进程调用recvfrom函数时，虽然在该函数上不会阻塞等待，但需要持续轮询内核，以检查是否有数据报到达。

#### IO复用

![image-20210317003357255](http://qiniu.qraffa.cn/image-20210317003357255.png)

进程调用select或poll函数，阻塞在这两个系统调用上，阻塞等待数据报到达，当数据报到达后，则可用调用recvfrom函数将数据报复制到进程缓冲区。

使用select或poll系统调用阻塞，相较于阻塞式IO，优势在于可以等待多个文件描述符。

#### 事件驱动IO

![image-20210317003736455](http://qiniu.qraffa.cn/image-20210317003736455.png)

进程通过sigaction系统调用安装一个信号处理函数，该函数立即返回。当数据报准备好时，内核产生一个SIGIO信号，进程就可以在信号处理函数中调用recvfrom函数读取数据报。

优势在于等待数据报到达期间，进程将不会被阻塞，只需要等待信号处理函数的通知。

#### 异步IO

![image-20210317004503032](http://qiniu.qraffa.cn/image-20210317004503032.png)

进程通过调用aio_read函数，向内核传递文件描述符、缓冲区指针、缓冲区大小、文件偏移，当数据报到达，内核将数据报复制到进程缓冲区时，通知进程。

与事件驱动IO的区别在于，事件驱动IO是内核通知进程可以开始IO操作，异步IO是内核将IO操作完成再通知进程。

#### 比较

![image-20210317005032467](http://qiniu.qraffa.cn/image-20210317005032467.png)

前四种都为同步IO操作，IO操作会将进程阻塞。最后一种为异步IO操作，进行IO操作不会导致进程阻塞。

### IO复用

io复用的使用场景

- 当客户处理多个描述符时，必须使用io复用
- 一个客户同时处理多个套接字
- 一个tcp服务器既要处理监听套接字，又要处理已连接套接字
- 一个服务器既要处理tcp，又要处理udp
- 一个服务器要处理多个服务或者多个协议

#### select

select函数原型：

```c
int select(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict exceptfds, struct timeval *restrict tvptr);
// 返回值：准备就绪的描述符集合；若超时，返回0；若出错，返回-1
```

**参数说明：**

- maxfdp1：指定待测试描述符的个数，其值为待测试描述符集合的最大描述符+1，因此会扫描测试描述符0,1,2...maxfdp1-1

- readfds、writefds和exceptfds：指向了描述符集合的指针，分别表示我们关心的可读、可写或处于异常条件的描述符集合

- tvptr：指定了select函数愿意等待的时间。参数由结构体timeval表示。

  ```c
  struct timeval {
      long tv_sec;
      long tv_usec;
  }
  ```

  - 当参数为NULL时，表示永远等待，直到存在一个描述符准备好时，select函数返回
  - 当参数为timeval，且两个值都为0时，表示不等待，select函数检查一次描述符集合，然后返回
  - 当参数为timeval，且两个值不为0时，表示等待指定的时间，当等待了超出指定时间后，返回0；当等待时间小于指定时间内，存在一个或多个描述符准备好IO，则返回值大于0。

**描述符集合**

对于描述符集合有如下4个函数可操作：

```c
// 检查fd描述符是否在描述符集合fdset中
int FD_ISSET(int fd, fd_set *fdset);
// 将描述符集合fdset所有位置0
void FD_ZERO(fd_set *fdset);
// 将描述符集合fdset中，对应fd的位置置于1
void FD_SET(int fd, fd_set *fdset);
// 将描述符集合fdset中，对应fd的位置置于0
void FD_CLR(int fd, fd_set *fdset);
```

**demo**

```c
while(1) {
    // 必须在循环中重置集合，并重新添加关心的描述符。
    // 因为每次select函数调用后，会将未就绪的描述符对应位置于0
    // 将集合置于0
    FD_ZERO(&fdset);
    // 添加关心的描述符
    FD_SET(fd, &fdset);
    // 表示关心读事件，且永远等待
    res = select(fd+1, &fdset, NULL, NULL, NULL);
    if (res > 0) {
        // 检查select函数返回后，关心的描述符位置是否为1,
        // 为1表示，该描述符准备好IO操作
        // 为0表示，该描述符未准备好，被select函数置于0了，
        // 如果我们关心该描述符的事件，则需要重置描述符集合
        if (FD_ISSET(fd, &fdset)) {
            // read...
        }
    }
}
```

#### poll

poll函数：

```c
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
// 返回值：准备就绪的描述符数目；若超时，返回0；若出错，返回-1

struct pollfd {
    int fd;
    short events;
    short revents;
}
```

#### epoll

epoll函数：

```c
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

**epoll_create：**用于创建epollfd

**epoll_ctl：**对fd进行操作，

op表示进行的操作，包含添加(EPOLL_CTL_ADD)，修改(EPOLL_CTL_MOD)，删除(EPOLL_CTL_DEL)

event表示关注的事件

```cpp
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

**epoll_wait：**等待epollfd上的事件，最多返回maxevents个事件，通过events参数来接收事件的集合。

**LT和ET：**epoll对fd有两种工作模式，为**LT（level trigger）**和**ET（edge trigger）**，默认使用LT。

LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序可以不立即处理该事件`。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序必须立即处理该事件`。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

#### 总结

**select：**

基本用法：通过FD_SET()设置我们关注的fd，然后通过select()阻塞等待关注的fd的事件就绪，将进程唤醒。关注读、写、异常都是使用同一个fd_set结构，一个bitmap，保存1024位。

基本流程：

1. 调用select方法后，需要在内核空间复制用户空间的相关参数，包括事件，读写异常的fd_set
2. 遍历所有的fd，调用该fd的poll()方法，根据结果，设置返回值的fd_set
3. 如果遍历完全部fd都没有就绪的fd，那么会进入睡眠
4. 当关注的fd上有数据就绪时，会将该进程唤醒，此时又会执行一次步骤2，那么这次就能够找到就绪的fd
5. 最后将返回的fd_set从内核态再拷贝给用户态
6. 用户进程从select()返回后，手动遍历fd_set找到关注的fd并检查关注的事件是否就绪

**epoll：**

基本用法：epoll通过epoll_create()方法创建epollfd，然后使用epoll_ctl()方法添加我们关注的fd，以及关注的事件，最后通过epoll_wait()阻塞等待关心的fd事件就绪，将进程唤醒。

基本流程：

1. epoll_create()主要任务就是创建并初始化eventpoll结构体

   ```c
   struct eventpoll {
   	/* Wait queue used by sys_epoll_wait() */
       /* 等待队列，调用epoll_wait()而阻塞的进程添加到该队列中 */
   	wait_queue_head_t wq;
   
   	/* List of ready file descriptors */
       /* 就绪队列，fd就绪后调用回调函数添加到这里，并且epoll_wait()方法也是通过检查该队列是否为空来判断是否需要阻塞当前进程 */
   	struct list_head rdllist;
   
   	/* RB tree root used to store monitored fd structs */
       /* 红黑树根节点，挂所有的epitem */
   	struct rb_root rbr;
   };
   ```

2. epoll_ctl()添加fd时

   1. 创建epitem
   2. 设置fd的等待队列，在其中添加一项，并且回调函数为ep_poll_callback。注意这里操作的等待队列是我们关注的那个fd，比如socket
   3. 将epitem添加到eventpoll的红黑树中

3. epoll_wait()

   1. 我们在当前进程调用epoll_wait()后，会先检查eventpoll的就绪队列，如果为空，则将当前进程添加到eventpoll的等待队列中，然后会进入休眠阻塞

4. 关注的fd就绪

   1. 当fd上的数据准备就绪后，会检查该fd上的等待队列，然后执行等待队列上的回调函数。这里就是我们在上面添加的ep_poll_callback函数
   2. 通过该等待队列中的信息找到eventpoll相关结构，然后将epitem添加到eventpoll的就绪队列中，然后将挂在eventpoll等待队列上的进程唤醒等待重新调度

5. 用户进程从epoll_wait()唤醒后，就可以通过参数来处理我们关注的fd的相关事件

![img](https://pic4.zhimg.com/80/v2-3672c51408eb089e6f3bd8c25fa81c8b_720w.jpg)

#### 对比

select：

1. 每次调用select需要从用户态复制数据到内核态
2. 在内核态检查事件时需要遍历整个fd_set
3. 由于使用bitmap，限制了最大1024个fd，并且最大值是1024

poll：

1. poll通过pollfd来替代select中的bitmap，突破了1024的限制
2. 其他与select类似，还是需要复制，还是需要遍历

epoll：

1. 通过epoll_create创建fd来节省用户空间到内核空间的数据拷贝
2. 每次对fd的关注操作，通过epoll_ctl方法，节省用户空间到内核空间的数据拷贝
3. 在内核态监听过程中，不需要遍历整个fd集合，只需要判断eventpoll的就绪队列即可
4. 在返回时，epoll只返回就绪的fd，而不是返回整个fd集合
5. 没有fd数量限制

#### 参考

[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

[select，poll，epoll实现分析—结合内核源代码](https://www.linuxidc.com/Linux/2012-05/59873.htm)

[select、poll、epoll的原理与区别](https://blog.csdn.net/jiejiemcu/article/details/107083724)

[epoll原理剖析#2： select & poll](https://heshaobo2012.medium.com/epoll%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90-3-select-poll-8d23b0a12906)

https://www.cnblogs.com/zhangyjblogs/p/15862358.html#select

[程序员练级攻略（2018）：系统知识](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E5%B7%A6%E8%80%B3%E5%90%AC%E9%A3%8E/075%20%20%E7%A8%8B%E5%BA%8F%E5%91%98%E7%BB%83%E7%BA%A7%E6%94%BB%E7%95%A5%EF%BC%882018%EF%BC%89%EF%BC%9A%E7%B3%BB%E7%BB%9F%E7%9F%A5%E8%AF%86.md)

[The C10K problem翻译](https://www.cnblogs.com/fll/archive/2008/05/17/1201540.html)

https://github.com/Liu-YT/IO-Multiplexing/

[图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的！](https://zhuanlan.zhihu.com/p/361750240)

[select 实现分析 –1 【整理】](https://www.cnblogs.com/apprentice89/archive/2013/05/09/3064975.html)