---
title: "网络IO"
date: 2025-03-19
categories: [操作系统, 网络]
tags: [os, network, io]
---

# 网络 IO

*   多路：指的是多个 socket 网络连接
*   复用：指的是复用一个进程(线程)来处理多个文件描述符(也就是 socket 连接)
*   技术：select、poll、epoll

历史原因得出的结论，为每个网络请求都分配一个进程/线程的方式都不合适，所以就产生了**IO多路复用技术**：只使用一个进程来维护多个 socket；

对于单核CPU来说，一个进程/线程虽然任一时刻只能处理一个请求，但是每个请求如果控制在 1 毫秒内，这样 1 秒就能处理上千个请求了，这就是IO多路复用技术，利用并发的思想提高单个进程处理网络请求的效率；

![image-20260324170507522](/assets/img/posts/image-20260324170507522.png)

## 1 Socket

socket 其实就是一个文件，它处于用户空间和内核空间之间，向用户空间提供一些接口让我们使用内核的功能，从而实现网络传输功能；

在用户空间创建 socket 后会将文件描述符 fd 暴露给用户，用户根据这个 sock\_fd 去操作内核空间暴露出来的接口：send()、recv()、bind()、listen()、connect() 等；

协议类型：

1.  **SOCK\_STREAM**
2.  **SOCK\_DGRAM**
3.  **SOCK\_RAW**

建立连接和发送数据的过程：

![image-20260324170519436](/assets/img/posts/image-20260324170519436.png)

客户端调用 connect 方法时会根据传入的 sock\_fd 句柄找到对应的文件，再根据文件里的信息找到内核的 sock 结构，通过这个 sock 结构主动发起三次握手，至此连接就准备好了；

为了实现发送/接收数据功能，sock 结构体中带了一个发送/接收缓冲区(链表)；

当应用执行 send 或 recv 时，也会根据传入的 sock\_fd 句柄找到对应的文件再找到对应的 sock 结构体，然后将数据放入发送缓冲区 或者 从接收缓冲区中取一个数据出来；

socket的唯一标识：

socket 通过：源ip、源端口、目标ip、目标端口 这四元组来唯一标识一个 socket；

服务端会维护一个哈希表，将所有连接到服务端的 socket 四元组转换成一个哈希值与对应 sock 的映射关系存到这个表中，下次有消息进来时通过四元组确定取哪个 sock 结构来进行网络传输即可；

服务器可以承载的最大TCP连接数是多少：

由于服务器的本地 ip 和端口是固定不变的，对于服务端 TCP 连接的四元组只有对端 IP 和端口是会变化的，所以 **最大 TCP 连接数 = 客户端 IP 数×客户端端口数**

对于 IPv4 来说，客户端 IP 数最多为 2 的 32 次方，客户端的端口数最多为 2 的16 次方，所以理论上 **服务端单机最大 TCP 连接数约为 2 的 48 次方**

现实中服务器可能承载不了这么多客户端连接，主要限制有以下几个方面：是

1.  文件描述符：Linux下，单个进程打开的文件描述符是有限制的，socket本身也是个文件受这个限制影响；可以通过 ulimit 修改文件描述符数量；
2.  系统内存：每个 TCP 连接在内核中都有对应的数据结构，所以每个连接都会占用一定内存；
3.  网络带宽：每个 TCP 连接都会占用一定的网络带宽；

## 2 IO 网络模型

**阻塞式I/O模型：**

进程/线程调用 read 后会阻塞，直到 recv 成功返回后，应用进程/线程开始处理数据。阻塞过程中进程/线程挂起不消耗CPU资源；适用并发量小的网络应用开发，不适用并发量大的应用，因为一个请求IO会阻塞进程，所以每请求分配一个处理进程（线程）去响应，系统开销大。

**非阻塞式I/O模型：**

进程发起IO系统调用后，如果内核缓冲区没有数据，需要到IO设备中读取，进程返回一个错误而不会被阻塞；进程发起IO系统调用后，如果内核缓冲区有数据，内核就会把数据返回进程，如果没数据就会返回错误。

*   进程轮询（重复）调用，消耗CPU的资源；
*   实现难度低、开发应用相对阻塞IO模式较难；
*   适用并发量较小、且不需要及时响应的网络应用开发；

**信号驱动 IO：**

当进程发起一个IO操作，会向内核注册一个信号处理函数，然后进程返回不阻塞；当内核数据就绪时会发送一个信号给进程，进程便在信号处理函数中调用IO读取数据。

特点：回调机制，实现、开发应用难度大；

**异步IO：**

当进程发起一个IO操作，进程返回（不阻塞），但也不能返回果结；内核把整个IO处理完后，会通知进程结果。如果IO操作成功则进程直接获取到数据。

特点：

*   不阻塞，数据一步到位；Proactor模式；
*   需要操作系统的底层支持，LINUX 2.5 版本内核首现，2.6 版本产品的内核标准特性；
*   实现、开发应用难度大；
*   非常适合高性能高并发应用；

**IO多路复用：**

在Linux系统中，磁盘控制器会先将磁盘数据拷贝到操作系统内核缓冲区，**然后才会从操作系统内核缓冲区拷贝到应用程序的地址空间中**；

然后 io多路复用技术中 提供的三个系统调用接口：select()、poll()、epoll()

这三个系统调用的作用就是从内核中拷贝多个网络事件到应用程序中；

当使用I/O多路复用系统调用时，监听的 socket 会被唤醒的情况有：

1.  数据可读
2.  数据可写
3.  有连接
4.  异常事件
5.  套接字关闭

## 3 IO 多路复用

目前支持 IO多路复用 的系统调用有：select()、poll()、epoll() 等；

与 多进程模型、多线程模型 相比，IO多路复用技术最大的优势就是系统开销小，系统不必创建和销毁进程/线程，不需要维护进程/线程的上下文关系；

IO多路复用技术是一种机制，通过一个进程/线程监听内核中的多个网络事件(文件描述符)，一旦某个网络事件就绪，就可以通过这些接口将内核中的事件拷贝用户空间中；这三个接口的拷贝方式各有不同，从而适用于不同的场景；

### 3.1 select

select 实现多路复用的方式是 将已连接的 socket 都放到一个**文件描述符集合**中，然后调用 select 系统调用函数将这个集合**拷贝**到内核中，让内核通过轮询的方式检查是否有就绪事件产生，若有就绪事件产生则返回给进程，并且告诉进程有几个网络事件已就绪；

1.  这个函数是阻塞的
2.  检查就绪事件是在内核中进行的
3.  文件描述符集合不可重复使用，需要重新赋值

几个细节：

1.  调用 select 时是将当前进程放入到每个 sock 的等待队列上的
2.  某个 sock 被唤醒后是会遍历整个 sock 集合将这个进程逐个从 sock 的等待队列上移除

```c
// sizeof(fd_set) = 128(Bytes) -> 1024(Bit) 每一位对应一个文件描述符
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);
    - 参数：
        - nfds : 委托内核检测的 最大文件描述符 的值 + 1  
            eg. 最大文件描述符 = 101   在内核中遍历 [0,102) 中需要检测的文件描述符属性是否发生改变
        - readfds : 要检测的文件描述符的 读的集合，委托内核检测那些文件描述符的读的属性
                    - 一般检测读操作
                    - 对应的是对方发送过来的数据，因为读是被动的接收数据，检测的就是读缓冲区
                    - 是一个传入传出参数
        - writefds : 要检测的文件描述符的 写的集合，委托内核检测哪些文件描述符的写的属性
        			- 委托内核检测写缓冲区是不是还可以写数据（缓冲区不满就可以写）
        - exceptfds : 检测发生异常的文件描述符的集合
        - timeout : 设置的超时时间
            struct timeval {
            long tv_sec; 	/* seconds */ 
            long tv_usec; 	/* microseconds */
            };
                - NULL : 永久阻塞，直到检测到了文件描述符有变化
                - tv_sec = 0 tv_usec = 0， 不阻塞
                - tv_sec > 0 tv_usec > 0， 阻塞对应的时间
    - 返回值 :
        - -1 : 失败
        -  0 ：无就绪IO 
        -  n : 成功，返回集合中已就绪的IO总个数
// 将参数文件描述符fd对应的标志位设置为0
void FD_CLR(int fd, fd_set *set);

// 判断fd对应的标志位是0还是1， 返回值 ： fd对应的标志位的值，0，返回0， 1，返回1
int FD_ISSET(int fd, fd_set *set);

// 将参数文件描述符fd 对应的标志位，设置为1
void FD_SET(int fd, fd_set *set);

// fd_set一共有1024 bit, 全部初始化为0
void FD_ZERO(fd_set *set);
```

实例：基于 select 实现的IO多路复用，使单个进程的服务器同时多个客户端连接：

```c
#include <stdio.h>
#include <ctype.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/select.h>
int main()
{
    // 创建socket
    int lfd = socket(AF_INET,SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(-1);
    }
    struct sockaddr_in saddr;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    
    // 绑定
    int ret = bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
    if(ret == -1) {
        perror("bind");
        return -1;
    }
    
    // 监听
    listen(lfd, 128);
    
    // 创建一个fd_set集合，存放的是需要检测的文件描述符
    fd_set rdset, tmp;
    FD_ZERO(&rdset);
    FD_SET(lfd, &rdset);
    int maxfd = lfd;
    
    while(1)
    {
        tmp = rdset;
        // 调用select系统调用，让内核帮忙检测哪些文件描述符有数据
        ret = select(maxfd + 1, &tmp, NULL, NULL, NULL);
        if(ret == -1)
        {
            perror("select");
            exit(-1);
        }
        else if(ret == 0) continue;
        else if(ret > 0)
        {
            // 说明检测到了有文件描述符对应的读缓冲区的数据发生了改变
            if(FD_ISSET(lfd, &tmp))
            {
                // 有新的客户端连接进来
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
                
                char cliIP[16];
                inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, cliIP, sizeof(cliIP));
                int port = ntohs(cliaddr.sin_port);
                // 输出客户端的信息
                printf("client's ip is %s, and port is %d\n", cliIP, port);

                // 将新的文件描述符加入到集合中
                FD_SET(cfd, &rdset);
                maxfd = maxfd > cfd ? maxfd: cfd;
            }
            for(int i = lfd + 1; i <= maxfd; ++i)
            {
                if(FD_ISSET(i, &tmp))
                {
                    // 说明这个文件描述符对应的客户端发来了数据
                    char buf[1024] = {0};
                    ret = read(i, buf, sizeof(buf));
                    if(ret == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    else if(ret == 0)
                    {
                        printf("client closed...\n");
                        close(i);
                        FD_CLR(i, &rdset);
                    }
                    else if(ret > 0)
                    {
                        printf("read buf = %s\n", buf);
                        write(i, buf, strlen(buf) + 1);
                    }
                }
            }
        }
    }
    close(lfd);
    return 0; 
}
```

select 存在几个问题：

1.  两次拷贝：
    1.  每次调用 select 时会将待监听的文件描述符集合拷贝到内核中；
    2.  每次有事件唤醒时，又会将集合从内核中拷贝到用户空间；

2.  两次遍历
    1.  每次调用 select 时都需要将进程加入到每个 socket 的等待队列中(阻塞)；这里是第一次遍历文件描述符集合 -> 发生在内核中；
    2.  有网络事件被唤醒后需要将进程从每个 socket 的等待队列中移除，然后将 socket 加入到CPU的工作队列中；这里是第二次遍历文件描述符集合 -> 发生在内核中；
    3.  内核中有就绪事件发生后，会将集合返回给用户空间，程序此时并不知道哪些 socket 收到数据，还需要遍历一次集合；这是第三次遍历文件描述符集合 -> 发生在用户空间中；

3.  正是因为有频繁的遍历和拷贝，处于效率考虑所以 select 使用固定长度的 BitsMap 来存储事件集合，默认长度为 1024 个，所以这方面也会有瓶颈；

### 3.2 poll

poll 和 select 的实现方式非常相似，只是存储**文件描述符集合**的数据结构有更新，使用动态数组存储，这样就解决了 select 固定集合大小的问题；

1.  这个函数是阻塞的
2.  检查就绪事件是在内核中进行的
3.  文件描述符集合可重复使用

```c
#include <poll.h>
struct pollfd {
    int fd; 		/* 委托内核检测的文件描述符 */
    short events; 	/* 委托内核检测文件描述符的什么事件 */
    short revents; 	/* 文件描述符实际发生的事件 */
};
struct pollfd myfd;
myfd.fd = 5;
myfd.events = POLLIN | POLLOUT;

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
    - 参数：
        - fds : 是一个struct pollfd 结构体数组，这是一个需要检测的文件描述符的集合
        - nfds : 这个是第一个参数数组中最后一个有效元素的下标 + 1
        - timeout : 阻塞时长(单位：ms)
            0  : 不阻塞
            -1 : 阻塞，当检测到需要检测的文件描述符有变化，解除阻塞
            >0 : 阻塞的时长
    - 返回值：
        -1 : 失败
         n : 成功,n表示检测到集合中有n个文件描述符发生变化
```

示例：基于poll实现IO多路复用，使服务器同时支持多个客户端进行接入：

```c
#include <stdio.h>
#include <ctype.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/select.h>
#include <poll.h>
#define LINKMAXCOUNTS 1024

int main()
{
    // 创建socket
    int lfd = socket(AF_INET,SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(-1);
    }
    struct sockaddr_in saddr;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;

    // 绑定
    int ret = bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
    if(ret == -1) {
        perror("bind");
        return -1;
    }

    // 监听
    listen(lfd, 128);

    // 初始化检测的文件描述符数组
    struct pollfd fds[LINKMAXCOUNTS];
    for(int i = 0; i < LINKMAXCOUNTS; i++)
    {
        fds[i].fd = -1;
        fds[i].events = POLLIN;
    }
    fds[0].fd = lfd;
    int nfds = 0;

    while(1)
    {
        // 调用poll系统调用，让内核帮忙检测哪些文件描述符有数据
        int ret = poll(fds, nfds + 1, -1);
        if(ret == -1)
        {
            perror("select");
            exit(-1);
        }
        else if(ret == 0) continue;
        else if(ret > 0)
        {
            // 说明检测到了有文件描述符对应的读缓冲区的数据发生了改变
            if(fds[0].revents & POLLIN)
            {
                // 有新的客户端连接进来
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
                
                char cliIP[16];
                inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, cliIP, sizeof(cliIP));
                int port = ntohs(cliaddr.sin_port);
                // 输出客户端的信息
                printf("client's ip is %s, and port is %d\n", cliIP, port);

                // 将新的文件描述符加入到集合中
                for(int i = 1; i < LINKMAXCOUNTS; ++i)
                {
                    if(fds[i].fd == -1)
                    {
                        fds[i].fd = cfd;
                        fds[i].events = POLLIN;
                        // 更新最大的文件描述符的索引
                        nfds = nfds > i ? nfds: i;
                        break;
                    }
                }
            }
            for(int i = 1; i <= nfds; ++i)
            {
                if(fds[i].revents & POLLIN)
                {
                    // 说明这个文件描述符对应的客户端发来了数据
                    char buf[1024] = {0};
                    ret = read(fds[i].fd, buf, sizeof(buf));
                    if(ret == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    else if(ret == 0)
                    {
                        printf("client closed...\n");
                        close(fds[i].fd);
                        fds[i].fd = -1;
                    }
                    else if(ret > 0)
                    {
                        printf("read buf = %s\n", buf);
                        write(fds[i].fd, buf, strlen(buf) + 1);
                    }
                }
            }
        }
    }
    close(lfd);
    return 0; 
}
```

poll 只解决了 select 固定集合大小的问题，所以还会存在 select 的两次拷贝三次遍历的问题，而且就算用动态数组代替 bitMap，集合长度也不宜过大，否则拷贝u和遍历的成本会很高；

select 和 poll 并没有太大本质区别，存储文件描述符集合的是**线性结构**，因此都需要遍历整个集合来找到就绪的 socket，事件复杂度为O(N)，而且也需要在用户空间和内核空间之间拷贝文件描述符集合，这两种方式随着并发数的增加，性能损耗也会成倍增长的；

### 3.3 epoll

![H-IO多路复用-epoll.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/85e0a9e45b364fa18755caf13d0a18a9~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgX-aWsOS4gA==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMjM1MzQxODcyNjgxMDg1NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1774947776&x-orig-sign=Ubyxos7Z1h7OVVFg4LszICaAIto%3D)

分析了前两个系统调用函数，于是引出了 epoll，epoll通过两方面很好的解决了 select 和 poll 的问题：

1.  epoll 在内核中创建了一个**红黑树**，来跟踪进程所有待检测的文件描述符；红黑树是个高效的数据结构，增删改的时间复杂度是 `O(logN)` 。优化了 select/poll遍历线性列表和拷贝整个列表的性能问题；`epoll_ctl` 每次只把待监听的 socket 加入到内核红黑树中，是增量传递；
2.  epoll 采用**事件驱动**机制，在内核中**维护了一个双向链表来记录就绪事件**，当某个 socket 被唤醒时，通过回调函数内核会将其加入到这个双向链表中，当用户调用 `epoll_wait` 函数时，只会返回已就绪的文件描述符，而不会返回整个文件描述符集合。优化了 select/poll 遍历线性列表和拷贝整个列表的性能问题，大大提高了检测效率；

所以总结 epoll 相比于 select/poll 的核心性能优势在于：

1.  内核中使用高效的数据结构存储文件描述符，提高增删改的时间复杂度；
2.  就绪事件有单独的数据结构存储，每次有事件被唤醒时能明确直到被唤醒的就绪事件的数量，减少了一次额外遍历的操作；
3.  epoll 通过将创建池子和添加文件描述符两个操作解耦，实现了池子中文件描述符的复用，减少了用户态和内核态之间数据拷贝的成本；

```c
#include <sys/epoll.h>
/* 
    创建一个新的epoll实例。在内核中创建了一个数据，这个数据中有两个比较重要的数据，一个是需要检
    测的文件描述符的信息（红黑树），还有一个是就绪列表，存放检测到数据发送改变的文件描述符信息（双向
    链表）。
*/
int epoll_create(int size);
    - 参数：
        size : 目前没有意义了。随便写一个数，必须大于0
    - 返回值：
        -1 : 失败
        >0 : 文件描述符，操作epoll实例的
                
// 对epoll实例进行管理：添加文件描述符信息，删除信息，修改信息
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
    - 参数：
        - epfd : epoll实例对应的文件描述符
        - op : 要进行什么操作
            EPOLL_CTL_ADD: 注册目标fd到epfd中, 同时关联event到fd上
            EPOLL_CTL_MOD: 修改已经注册到fd的监听事件
            EPOLL_CTL_DEL: 从epfd中删除/移除已注册的fd, event可以被忽略, 也可以为NULL
        - fd : 要检测的文件描述符
        - event : 检测文件描述符什么事情(读/写)
            
        struct epoll_event {
            uint32_t events; 	/* Epoll events */
            epoll_data_t data; 	/* User data variable */
        };
        常见的Epoll检测事件events：
        - EPOLLIN：表示关联的fd可以进行读操作了
        - EPOLLOUT：表示关联的fd可以进行写操作了。
        - EPOLLERR：表示关联的fd发生了错误, epoll_wait会一直等待这个事件, 一般没必要设置这个属性。
        - EPOLLET： 设置边沿触发
        - EPOLLRDHUP: 表示套接字关闭了连接, 或者关闭了正写一半的连接。
        - EPOLLHUP
        - EPOLLONESHOT:  操作系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次
// cited from: https://blog.csdn.net/weixin_33277243/article/details/116609618
            
// 联合体(union)在C语言中是一个特殊的数据类型，能够存储不同类型的数据在同一个内存位置。可以定义一个联合体使用许多成员，但只有一个部件可以包含在任何时候给定的值。
// union所占用的内存将大到足以容纳联合体的最大成员
// ps.由于联合体的成员使用同一块儿地址，因此我们可以通过联合体来巧妙的判断当前机器的存储方式是大端字节序还是小端字节序 
        typedef union epoll_data {
            void *ptr;
            int fd;
            uint32_t u32;
            uint64_t u64;
        } epoll_data_t;
                
// 检测函数
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int
timeout);
    - 参数：
        - epfd : epoll实例对应的文件描述符
        - events : 传出参数，保存了发生变化的文件描述符的信息
        - maxevents : 第二个参数结构体数组的大小
        - timeout : 阻塞时间
            - 0 : 不阻塞
            - -1 : 阻塞，直到检测到fd数据发生变化，解除阻塞
            - > 0 : 阻塞的时长（毫秒）
    - 返回值：
        - 成功，返回发送变化的文件描述符的个数 > 0
        - 失败 -1
```

示例：基于epoll实现IO多路复用，使服务器同时支持多个客户端进行接入：

```c
#include <stdio.h>
#include <ctype.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
int main()
{
    // 创建socket
    int lfd = socket(AF_INET,SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    // 绑定
    int ret = bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
    // 监听
    listen(lfd, 128);
    // 调用epoll_create()创建一个epoll示例
    int epfd = epoll_create(100);
    // 将监听的文件描述符相关的监测信息添加到epoll实例中
    struct epoll_event epev;
    epev.events = EPOLLIN;
    epev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &epev);
    struct epoll_event epevs[1024];
    while(1)
    {
        ret = epoll_wait(epfd, epevs, sizeof(epevs) / sizeof(struct epoll_event), -1);
        if(ret == -1)
        {
            perror("epoll_wait");
            exit(-1);
        }
        printf("已经检测到变化的文件描述符数量 = %d\n", ret);
        for(int i = 0; i < ret; ++i)
        {
            int curfd = epevs[i].data.fd;
            if(curfd == lfd)
            {
                // 监听的文件描述符有数据到达(有客户端连接)
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
                // epev.events = EPOLLIN | EPOLLOUT;
                epev.events = EPOLLIN;
                epev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &epev);
            }
            else{
                // 判断不同事件 并进行对应处理
                if(epevs[i].events & EPOLLIN)
                {
                    // 有数据到达，需要通信
                    char buf[1024] = {0};
                    int len = read(curfd, buf, sizeof(buf));
                    if(len == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    else if(len == 0)
                    {
                        printf("client closed...\n");
                        epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                        close(curfd);
                    }
                    else if(len > 0)
                    {
                        printf("read buf = %s\n", buf);
                        write(curfd, buf, strlen(buf) + 1);
                    }
                }
            }
        }
    }
    close(lfd);
    close(epfd);
    return 0;
}
```

**水平触发 和 边缘触发**

水平触发（level triggered）：

当被监控的 socket 上有就绪事件发生时，服务端会不断地从 epoll\_wait 中返回就绪，直到内核缓冲区数据被 read 函数读完才结束；

边缘触发（edge triggered）：

当被监控的 socket 上有就绪事件发生时，服务器端只会从 epoll\_wait 中苏醒一次，即使没有读取数据也不会重复提醒了；

实操：

如果使用水平触发模式，可以在任意时候去执行读写操作，因为会重复通知未执行的事件；

如果使用边缘触发模式，IO事件只会通知一次就绪，所以此时应该尽可能地多读写数据，所以边缘触发模式一般和非阻塞IO搭配使用：循环执行IO操作，直到系统调用返回错误（`EAGAIN` 或 `EWOULDBLOCK`）标识数据已读完，这时候退出循环即可；