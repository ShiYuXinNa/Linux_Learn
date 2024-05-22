# 同步阻塞

**单线程：**某个socket阻塞，会影响到其他socket处理。
**多线程：**客户端较多时，会造成资源浪费，全部socket中可能每个时刻只有几个就绪。同时，线程的调度、上下文切换乃至它们占用的内存，可能都会成为瓶颈。

| 时序 | socket1 | socket2 | socket3 | socket4 |
| :--: | :-----: | :-----: | :-----: | :-----: |
|  1   |  就绪   |         |         |         |
|  2   |         | 就绪    |         |         |
|  3   |         |         |  就绪   |         |
|  4   |         |         |         | 就绪    |

# 同步非阻塞

总结

- 从操作系统层面解决了阻塞问题。

优点
- 单个 socket 阻塞，不会影响到其他 socket

缺点
- 需要不断的遍历进行系统调用，有一定开销

# Select

## 总结

过程：

​	如果服务端监听了4个socket，在用户空间调用内核函数时，首先将用户空间的fd_set拷贝到内核空间，然后内核空间会遍历fd_set,标记就绪的fd，遍历结束后如果有fd就绪，就返回fd就绪的数量。如果没有fd就绪，将当前用户进程堵塞，当客户端向服务端发送数据时，数据通过网络传输到达服务端的网卡，网卡通过dma的方式将这个数据包写入到指定的内存中，处理完成后会通过中断信号告诉cpu有新的数据包到达，cpu收到中断信号后会进行响应中断，调用中断处理函数处理，根据数据包的ip和端口号找到这个socket，将数据保存到这个socket的一个接收队列，检查这个socket对应的一个等待队列里面是否有用户进程正在阻塞等待，如果有就唤醒该用户进程，用户进程唤醒后就会继续检查一遍fd_Set。

总结：将socket是否就绪检查逻辑下沉到操作系统层面，避免大量系统调用。告诉你有事件就绪，但是没告诉你具体是哪个FD

优点：
不需要每个 FD 都进行一次系统调用，解决了频繁的用户态内核态切换问题

缺点：

- 单进程监听的 FD 存在限制，默认1024
- 每次调用需要将 FD 从用户态拷贝到内核态
- 不知道具体是哪个文件描述符就绪，需要遍历全部文件描述符
- 入参的3个 fd_set 集合每次调用都需要重置

## API

```c
#include <sys/select.h>
struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
};

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval * timeout);

```

函数参数：

- nfds：委托内核检测的这三个集合中最大的文件描述符 + 1
  - 内核需要线性遍历这些集合中的文件描述符，这个值是循环结束的条件
  - 在Window中这个参数是无效的，指定为-1即可
- readfds：文件描述符的集合, 内核只检测这个集合中文件描述符对应的读缓冲区
  - 传入传出参数，读集合一般情况下都是需要检测的，这样才知道通过哪个文件描述符接收数据
- writefds：文件描述符的集合, 内核只检测这个集合中文件描述符对应的写缓冲区
  - 传入传出参数，如果不需要使用这个参数可以指定为NULL
- exceptfds：文件描述符的集合, 内核检测集合中文件描述符是否有异常状态
  - 传入传出参数，如果不需要使用这个参数可以指定为NULL
- timeout：超时时长，用来强制解除select()函数的阻塞的
- NULL：函数检测不到就绪的文件描述符会一直阻塞。
  - 等待固定时长（秒）：函数检测不到就绪的文件描述符，在指定时长之后强制解除阻塞，函数返回0
  - 不等待：函数不会阻塞，直接将该参数对应的结构体初始化为0即可。

函数返回值：

- 大于0：成功，返回集合中已就绪的文件描述符的总个数
- 等于-1：函数调用失败
- 等于0：超时，没有检测到就绪的文件描述符

```c
// 将文件描述符fd从set集合中删除 == 将fd对应的标志位设置为0        
void FD_CLR(int fd, fd_set *set);
// 判断文件描述符fd是否在set集合中 == 读一下fd对应的标志位到底是0还是1
int  FD_ISSET(int fd, fd_set *set);
// 将文件描述符fd添加到set集合中 == 将fd对应的标志位设置为1
void FD_SET(int fd, fd_set *set);
// 将set集合中, 所有文件文件描述符对应的标志位设置为0, 集合中没有添加任何文件描述符
void FD_ZERO(fd_set *set);
```

## 并发处理

### 处理流程

如果在服务器基于select实现并发，其处理流程如下：

1. 创建监听的套接字 lfd = socket();

2. 将监听的套接字和本地的IP和端口绑定 bind()

3. 给监听的套接字设置监听 listen()

4. 创建一个文件描述符集合 fd_set，用于存储需要检测读事件的所有的文件描述符

   - 通过 FD_ZERO() 初始化

   - 通过 FD_SET() 将监听的文件描述符放入检测的读集合中

5. 循环调用select()，周期性的对所有的文件描述符进行检测

6. select() 解除阻塞返回，得到内核传出的满足条件的就绪的文件描述符集合

   - 通过FD_ISSET() 判断集合中的标志位是否为 1

     - 如果这个文件描述符是监听的文件描述符，调用 accept() 和客户端建立连接
       - 将得到的新的通信的文件描述符，通过FD_SET() 放入到检测集合中

     - 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信

       - 如果客户端和服务器断开了连接，使用FD_CLR()将这个文件描述符从检测集合中删除

       - 如果没有断开连接，正常通信即可

7. 重复第6步

![image-20231008172939977](C:\Users\Xin\AppData\Roaming\Typora\typora-user-images\image-20231008172939977.png)

### 通信代码

#### 服务器端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建监听的fd
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));

    // 3. 设置监听
    listen(lfd, 128);

    // 将监听的fd的状态检测委托给内核检测
    int maxfd = lfd;
    // 初始化检测的读集合
    fd_set rdset;
    fd_set rdtemp;
    // 清零
    FD_ZERO(&rdset);
    // 将监听的lfd设置到检测的读集合中
    FD_SET(lfd, &rdset);
    // 通过select委托内核检测读集合中的文件描述符状态, 检测read缓冲区有没有数据
    // 如果有数据, select解除阻塞返回
    // 应该让内核持续检测
    while(1)
    {
        // 默认阻塞
        // rdset 中是委托内核检测的所有的文件描述符
        rdtemp = rdset;
        int num = select(maxfd+1, &rdtemp, NULL, NULL, NULL);
        // rdset中的数据被内核改写了, 只保留了发生变化的文件描述的标志位上的1, 没变化的改为0
        // 只要rdset中的fd对应的标志位为1 -> 缓冲区有数据了
        // 判断
        // 有没有新连接
        if(FD_ISSET(lfd, &rdtemp))
        {
            // 接受连接请求, 这个调用不阻塞
            struct sockaddr_in cliaddr;
            int cliLen = sizeof(cliaddr);
            int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &cliLen);

            // 得到了有效的文件描述符
            // 通信的文件描述符添加到读集合
            // 在下一轮select检测的时候, 就能得到缓冲区的状态
            FD_SET(cfd, &rdset);
            // 重置最大的文件描述符
            maxfd = cfd > maxfd ? cfd : maxfd;
        }

        // 没有新连接, 通信
        for(int i=0; i<maxfd+1; ++i)
        {
			// 判断从监听的文件描述符之后到maxfd这个范围内的文件描述符是否读缓冲区有数据
            if(i != lfd && FD_ISSET(i, &rdtemp))
            {
                // 接收数据
                char buf[10] = {0};
                // 一次只能接收10个字节, 客户端一次发送100个字节
                // 一次是接收不完的, 文件描述符对应的读缓冲区中还有数据
                // 下一轮select检测的时候, 内核还会标记这个文件描述符缓冲区有数据 -> 再读一次
                // 	循环会一直持续, 知道缓冲区数据被读完位置
                int len = read(i, buf, sizeof(buf));
                if(len == 0)
                {
                    printf("客户端关闭了连接...\n");
                    // 将检测的文件描述符从读集合中删除
                    FD_CLR(i, &rdset);
                    close(i);
                }
                else if(len > 0)
                {
                    // 收到了数据
                    // 发送数据
                    write(i, buf, strlen(buf)+1);
                }
                else
                {
                    // 异常
                    perror("read");
                }
            }
        }
    }

    return 0;
}
```

#### 客户端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建用于通信的套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 连接服务器
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;     // ipv4
    addr.sin_port = htons(9999);   // 服务器监听的端口, 字节序应该是网络字节序
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr.s_addr);
    int ret = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("connect");
        exit(0);
    }

    // 通信
    while(1)
    {
        // 读数据
        char recvBuf[1024];
        // 写数据
        // sprintf(recvBuf, "data: %d\n", i++);
        fgets(recvBuf, sizeof(recvBuf), stdin);
        write(fd, recvBuf, strlen(recvBuf)+1);
        // 如果客户端没有发送数据, 默认阻塞
        read(fd, recvBuf, sizeof(recvBuf));
        printf("recv buf: %s\n", recvBuf);
        sleep(1);
    }

    // 释放资源
    close(fd); 

    return 0;
}

```

# POLL

总结：跟 select基本类似，主要优化了监听1024的限制(用链表存储)

优点：

- 不需要每个 FD 都进行一次系统调用，导致频繁的用户态内核态切换

缺点：

- 每次需要将 FD 从用户态拷贝到内核态不知道具体是哪个文件描述符就绪，需要遍历全部文件描述符

# EPOLL

克服POLL缺点:

1. 每次需要将fd从用户态拷贝到内核态，如果fd_set过大也会需要一定的开销。
2. 返回就绪事件时，用户态不知道具体是哪个fd就绪只知道数量，没次进行一个O(n)遍历，才能找到。

## API

```c
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

```

select/poll低效的原因之一是将“添加/维护待检测任务”和“阻塞进程/线程”两个步骤合二为一。每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket个数相对固定，并不需要每次都修改。epoll将这两个操作分开，先用epoll_ctl()维护等待队列，再调用epoll_wait()阻塞进程（解耦）。通过下图的对比显而易见，epoll的效率得到了提升。

![img](https://subingwen.cn/linux/epoll/image-20210403181746358.png)

epoll_create()函数的作用是创建一个红黑树模型的实例，用于管理待检测的文件描述符的集合。

```c
int epoll_create(int size);
```

- 函数参数 size：在Linux内核2.6.8版本以后，这个参数是被忽略的，只需要指定一个大于0的数值就可以了。

- 函数返回值：

  - 失败：返回-1

  - 成功：返回一个有效的文件描述符，通过这个文件描述符就可以访问创建的epoll实例了


epoll_ctl()函数的作用是管理红黑树实例上的节点，可以进行添加、删除、修改操作。

```c
// 联合体, 多个变量共用同一块内存        
typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 通常情况下使用这个成员, 和epoll_ctl的第三个参数相同即可
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;      /* Epoll events */
	epoll_data_t data;        /* User data variable */
};
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

函数参数：

- epfd：epoll_create() 函数的返回值，通过这个参数找到epoll实例

- op：这是一个枚举值，控制通过该函数执行什么操作

  - EPOLL_CTL_ADD：往epoll模型中添加新的节点
  - EPOLL_CTL_MOD：修改epoll模型中已经存在的节点
  - EPOLL_CTL_DEL：删除epoll模型中的指定的节点

- fd：文件描述符，即要添加/修改/删除的文件描述符

- event：epoll事件，用来修饰第三个参数对应的文件描述符的，指定检测这个文件描述符的什么事件

  - events：委托epoll检测的事件
    - EPOLLIN：读事件, 接收数据, 检测读缓冲区，如果有数据该文件描述符就绪
    - EPOLLOUT：写事件, 发送数据, 检测写缓冲区，如果可写该文件描述符就绪
    - EPOLLERR：异常事件
  - data：用户数据变量，这是一个联合体类型，通常情况下使用里边的fd成员，用于存储待检测的文件描述符的值，在调用epoll_wait()函数的时候这个值会被传出。

- 函数返回值：

  - 失败：返回-1
  - 成功：返回0


epoll_wait()函数的作用是检测创建的epoll实例中有没有就绪的文件描述符。

```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

函数参数：

- epfd：epoll_create() 函数的返回值, 通过这个参数找到epoll实例
- events：传出参数, 这是一个结构体数组的地址, 里边存储了已就绪的文件描述符的信息
- maxevents：修饰第二个参数, 结构体数组的容量（元素个数）
- timeout：如果检测的epoll实例中没有已就绪的文件描述符，该函数阻塞的时长, 单位ms 毫秒
  - 0：函数不阻塞，不管epoll实例中有没有就绪的文件描述符，函数被调用后都直接返回
  - 大于0：如果epoll实例中没有已就绪的文件描述符，函数阻塞对应的毫秒数再返回
  - -1：函数一直阻塞，直到epoll实例中有已就绪的文件描述符之后才解除阻塞
- 函数返回值：
  - 成功：
    - 等于0：函数是阻塞被强制解除了, 没有检测到满足条件的文件描述符
    - 大于0：检测到的已就绪的文件描述符的总个数
  - 失败：返回-1

## 使用

在服务器端使用epoll进行IO多路转接的操作步骤如下：

1. 创建监听的套接字
2. 设置端口复用（可选）
3. 使用本地的IP与端口和监听的套接字进行绑定
4. 给监听的套接字设置监听
5. 创建epoll实例对象
6. 将用于监听的套接字添加到epoll实例中
7. 检测添加到epoll实例中的文件描述符是否已就绪，并将这些已就绪的文件描述符进行处理
   - 如果是监听的文件描述符，和新客户端建立连接，将得到的文件描述符添加到epoll实例中
   - 如果是通信的文件描述符，和对应的客户端通信，如果连接已断开，将该文件描述符从epoll实例中删除
8. 重复第7步的操作

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }

    struct epoll_event evs[1024];
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 调用一次, 检测一次
        int num = epoll_wait(epfd, evs, size, -1);
        for(int i=0; i<num; ++i)
        {
            // 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                ev.events = EPOLLIN;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[1024];
                memset(buf, 0, sizeof(buf));
                int len = recv(curfd, buf, sizeof(buf), 0);
                if(len == 0)
                {
                    printf("客户端已经断开了连接\n");
                    // 将这个文件描述符从epoll模型中删除
                    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                    close(curfd);
                }
                else if(len > 0)
                {
                    printf("客户端say: %s\n", buf);
                    send(curfd, buf, len, 0);
                }
                else
                {
                    perror("recv");
                    exit(0);
                } 
            }
        }
    }

    return 0;
}

```

