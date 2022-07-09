```Redis```服务器是一个事件驱动程序, 服务器需要处理以下两类事件

- 文件事件: ```redis```服务器会通过套接字和客户端进行连接, 文件事件就是服务器对套接字操作的抽象(读写套接字)
- 时间时间: ```redis```服务器的一些操作需要定时执行, 而时间事件就是服务器对这类定时操作的抽象

```Redis```基于```Reactor```模式开发了自己的网络事件处理器, 采用I/O多路复用程序同时监听多个套接字, 

### IO多路复用程序的实现

```redis```的```I/O```多路复用程序的所有功能都是对```select,epoll,evport,kqueue```这些```I/O```多路复用库的再封装

每种实现都对应一个单独的文件(```ae_select.c```,```ae_epoll.c```,```ae_kqueue.c```)

他们都实现了相同的一组```api```, 所以```redis```的```I/O```多路复用程序底层实现可以在这三种方式下切换(通过一组宏定义,在编译时自动选择性能最好的一种实现,然后进行```#include```)



### 多路复用库实现之一(epoll) 

```redis```中使用```aeEventLoop```管理和多路复用相关的信息.

```c
typedef struct aeEventLoop {

    // 目前已注册的最大描述符
    int maxfd;   /* highest file descriptor currently registered */

    // 目前已追踪的最大描述符
    int setsize; /* max number of file descriptors tracked */

    // 用于生成时间事件 id
    long long timeEventNextId;

    // 最后一次执行时间事件的时间
    time_t lastTime;     /* Used to detect system clock skew */

    // 已注册的文件事件
    aeFileEvent *events; /* Registered events */

    // 已就绪的文件事件
    aeFiredEvent *fired; /* Fired events */

    // 时间事件
    aeTimeEvent *timeEventHead;

    // 事件处理器的开关
    int stop;

    // 多路复用库的私有数据
    void *apidata; /* This is used for polling API specific data */

    // 在处理事件前要执行的函数
    aeBeforeSleepProc *beforesleep;

} aeEventLoop;

typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    // 读事件处理器
    aeFileProc *rfileProc;
    // 写事件处理器
    aeFileProc *wfileProc;
    // 多路复用库的私有数据
    void *clientData;

} aeFileEvent;

typedef struct aeFiredEvent {
    // 已就绪文件描述符
    int fd;
    // 事件类型掩码，
    // 值可以是 AE_READABLE 或 AE_WRITABLE
    // 或者是两者的或
    int mask;

} aeFiredEvent;
```

下面我们假设当多路复用库的底层实现是```ae_epoll.c```时, 是如何包装方法的.

首先是私有数据```aeEventLoop.apidata```, 这个字段是因为在不同的多路复用库实现下需要保存不同的字段, 所以开了这个变量存储.

```epoll```的私有数据就是下面这个结构体.使用```epoll```首先需要一个```epoll```实例的文件描述符, 还需要保存监听的所有事件.

```c
typedef struct aeApiState {
    // epoll_event 实例描述符
    int epfd;
    // 事件槽
    struct epoll_event *events;
} aeApiState;
```

初始化多路复用库(具体就是创建```epoll```实例, 赋值给```aeEventLoop```)(相当于```epoll_create```)

```c
static int aeApiCreate(aeEventLoop *eventLoop) {

    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;

    // 初始化事件槽空间
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }

    // 创建 epoll 实例
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }

    // 赋值给 eventLoop
    eventLoop->apidata = state;
    return 0;
}
```



新增监听的文件描述符/增加监听的文件描述符事件(相当于一部分```epoll_ctl```)

只需要传入被监听的```fd```和监听事件类型```mask```即可. 具体是一个```EPOLL_CTL_ADD```选项还是```EPOLL_CTL_MOD```选项可以根据该文件描述符之前是否出现在事件槽中得知

```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee;

    /* 
     * 如果 fd 没有关联任何事件，那么这是一个 ADD 操作。
     * 如果已经关联了某个/某些事件，那么这是一个 MOD 操作。
     */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    // 注册事件到 epoll
    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.u64 = 0; /* avoid valgrind warning */
    ee.data.fd = fd;

    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;

    return 0;
}
```

删除监听的文件描述符/减少监听的文件描述符事件(相当于一部分```epoll_ctl```)

传入需要删除的事件```delmask```即可

这样```~delmask```和之前的```mask```做一个与运算就相当于删除了```delmask```表示的事件

```c
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee;

    int mask = eventLoop->events[fd].mask & (~delmask);

    ee.events = 0;
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.u64 = 0; /* avoid valgrind warning */
    ee.data.fd = fd;
    if (mask != AE_NONE) {
        epoll_ctl(state->epfd,EPOLL_CTL_MOD,fd,&ee);
    } else {
        /* Note, Kernel < 2.6.9 requires a non null event pointer even for
         * EPOLL_CTL_DEL. */
        epoll_ctl(state->epfd,EPOLL_CTL_DEL,fd,&ee);
    }
}
```

获取有事件发生的文件描述符(相当于```epoll_wait```)

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    // 等待时间
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);

    // 有至少一个事件就绪？
    if (retval > 0) {
        int j;

        // 为已就绪事件设置相应的模式
        // 并加入到 eventLoop 的 fired 数组中
        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;

            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    
    // 返回已就绪事件个数
    return numevents;
}
```

综上所诉, 这就是```redis```中的一个简单封装.

### ```redis```事件```api```

文件在```ae.c```

```redis```事件的```api```就是对上面的多路复用库的再包装.(在我看来,除了调用多路复用函数库外, 还有绑定事件处理器的功能)

这里的事件处理器指的是, 当文件描述符可写或可读时应该被调用的函数.

```aeCreateFileEvent```: 将给定套接字的给定事件加入到多路复用程序的监听范围内, 并与事件处理器绑定.

```c

typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    // 读事件处理器
    aeFileProc *rfileProc;
    // 写事件处理器
    aeFileProc *wfileProc;
    // 多路复用库的私有数据
    void *clientData;

} aeFileEvent;
/*
 * 根据 mask 参数的值，监听 fd 文件的状态，
 * 当 fd 可用时，执行 proc 函数
 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }

    if (fd >= eventLoop->setsize) return AE_ERR;

    // 取出文件事件结构
    aeFileEvent *fe = &eventLoop->events[fd];

    // 监听指定 fd 的指定事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;

    // 设置文件事件类型，以及事件的处理器
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;

    // 私有数据
    fe->clientData = clientData;

    // 如果有需要，更新事件处理器的最大 fd
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;

    return AE_OK;
}
```

然后删除监听的文件描述符事件也是类似的做法, 增删操作不再累赘, 下面看主函数

```aeMain```几乎是事件处理器的入口, 可以看到先把```eventLoop->stop```置为```0```表示开始运行, 然后不断调```aeProcessEvents```

```c
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)
/*
 * 事件处理器的主循环
 */
void aeMain(aeEventLoop *eventLoop) {

    eventLoop->stop = 0;

    while (!eventLoop->stop) {

        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS); //相当于文件事件和时间事件都处理
    }
}
```

看一下```aeProcessEvents```的源码, 会比较多一点

```c
/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 *
 * 处理所有已到达的时间事件，以及所有已就绪的文件事件。
 *
 *
 * 如果不传入特殊 flags 的话，那么函数睡眠直到文件事件就绪，
 * 或者下个时间事件到达（如果有的话）。
 *
 * 如果 flags 为 0 ，那么函数不作动作，直接返回。
 *
 * 如果 flags 包含 AE_ALL_EVENTS ，所有类型的事件都会被处理。
 *
 * 如果 flags 包含 AE_FILE_EVENTS ，那么处理文件事件。
 *
 * 如果 flags 包含 AE_TIME_EVENTS ，那么处理时间事件。
 *
 * 如果 flags 包含 AE_DONT_WAIT ，
 * 那么函数在处理完所有不许阻塞的事件之后，即刻返回。
 *
 * 函数的返回值为已处理事件的数量
 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    // 先要求有文件描述符被监听.或者如果没文件描述符,那么必须有事件事件,且必须允许阻塞才行
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        // 获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            // 如果时间事件存在的话
            // 那么根据最近可执行时间事件和现在时间的时间差来决定文件事件的阻塞时间
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            // 计算距今最近的时间事件还要多久才能达到
            // 并将该时间距保存在 tv 结构中
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }

            // 时间差小于 0 ，说明事件已经可以执行了，将秒和毫秒设为 0 （不阻塞）
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            
            // 执行到这一步，说明没有时间事件
            // 那么根据 AE_DONT_WAIT 是否设置来决定是否阻塞，以及阻塞的时间长度

            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                // 设置文件事件不阻塞
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                // 文件事件可以阻塞直到有事件到达为止
                tvp = NULL; /* wait forever */
            }
        }

        // 处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

           /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }

            processed++;
        }
    }

    /* Check time events */
    // 执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

我们暂且不管时间事件, 先看文件事件

```c
        // 处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

           /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }

            processed++;
```

可以看到先调用```aeApiPoll```等待一些文件事件, 并将就绪的文件描述符放在```aeEventLoop.fired```数组中

做完这步后, 循环遍历```fired```数组, 根据文件描述符去```events```数组取出具体信息(比如可写时应该执行的函数和可读时执行的函数)

根据该文件的```mask```去执行对应的函数(```mask```在```aeApiPoll```中已经初始化过.



### 文件事件处理器的类型

前面说到在```aeProcessEvents```函数中, 会调用```aeApiPoll```等待一些文件事件(可读或可写), 然后执行他们绑定的对应文件事件处理器

在```redis```中的事件处理器有很多种

- 为了对连接服务器的客户端进行应答, 服务器要为监听套接字关联连接应答处理器
- 为了接收客户端传来的命令, 服务器要为客户端套接字关联命令请求处理器
- 为了向客户端返回命令的执行结果, 服务器要为客户端套接字关联命令回复处理器
- 当主服务器和从服务器进行复制操作时, 主从服务器都要关联特别为复制功能编写的复制处理器

暂时只需要关注前面三个处理器.

#### 连接应答处理器

```networking.c/acceptTcpHandler```是服务器监听套接字, 当可读时应该执行的函数.

由于```redis```是单进程的, 所以这里设置了一个阈值```MAX_ACCEPTS_PER_CALL```为```1000```, 表示表示单次最多连接1000个客户端,因为不能阻塞太久

也因为这个特性, ```epoll```必须以```LT```模式运行, 保证这次没读完的数据, 下次还能进行读取

```c
/* 
 * 创建一个 TCP 连接处理器
 */
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    // MAX_ACCEPTS_PER_CALL是1000,表示单次最多连接1000各客户端,因为不能阻塞太久
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[REDIS_IP_STR_LEN];
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);
    REDIS_NOTUSED(privdata);

    while(max--) {
        // accept 客户端连接
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                redisLog(REDIS_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
        // 为客户端创建客户端状态（redisClient）
        acceptCommonHandler(cfd,0);
    }
}
```

这里可以看一下``anetTcpAccept``的具体实现

通过```anetGenericAccept```获取客户端的文件描述符.其实内部就是不断```Accept```知道一个能连接的客户端,没什么好说的.

然后顺便记录以下客户端的```ip```和端口即可

```c
int anetTcpAccept(char *err, int s, char *ip, size_t ip_len, int *port) {
    int fd;
    struct sockaddr_storage sa;
    socklen_t salen = sizeof(sa);
    if ((fd = anetGenericAccept(err,s,(struct sockaddr*)&sa,&salen)) == -1)
        return ANET_ERR;

    if (sa.ss_family == AF_INET) {
        struct sockaddr_in *s = (struct sockaddr_in *)&sa;
        if (ip) inet_ntop(AF_INET,(void*)&(s->sin_addr),ip,ip_len);
        if (port) *port = ntohs(s->sin_port);
    } else {
        struct sockaddr_in6 *s = (struct sockaddr_in6 *)&sa;
        if (ip) inet_ntop(AF_INET6,(void*)&(s->sin6_addr),ip,ip_len);
        if (port) *port = ntohs(s->sin6_port);
    }
    return fd;
}
```

然后接收了一个客户端请求后, ```redis```为它创建一个客户端状态```acceptCommonHandler```

#### 命令请求处理器

```networking.c/readQueryFromClient```函数是```redis```的命令请求处理器, 负责从套接字读入客户端发送的命令请求内容

当一个客户端通过连接应答处理器连接到服务器后, 服务器会将客户端套接字的```AE_READABLE```事件和命令请求处理器关联。

也就是从客户端套接字中读取数据, 解析命令, 并调用相应的函数来处理

命令请求处理器会一直和客户端套接字相关联

#### 命令回复处理器

```networking.c/sendReplyToClient```函数是```redis```的命令回复处理器.这个处理器负责将服务器执行命令得到的结果通过套接字返回客户端

当服务器有命令回复需要传送给客户端时, 服务器会将客户端套接字的```AE_WRITEABLE```事件和命令回复处理器关联起来

当命令回复发送完毕后, 服务器就会解除与客户端套接字的关联

### 一次完整的客户端与服务器连接事件实例

首先```redis```会开一个循环监听所有注册的文件描述符的对应事件, 每次循环处理那些准备好的事件.

假设一个```redis```服务器在运作, 那么在最开始, ```epoll```只会监听它的读事件.

 如果有数据来了就好被触发, 然后调用连接处理器进行应答.

其中连接处理器是以```LT```模式运行, 且单次最多接收```1000```个客户端连接.(不能阻塞, 因为在本次循环中,还有其他被监听的文件描述符等待处理)

每连接成功一个客户端就会创建相应的客户端套接字和客户端状态, 并将客户端套接字的```AE_READABLE```事件与命令请求处理器进行关联。

第一次循环, 就只有服务端监听套接字会被触发.

第二次循环, 此时已经连接了若干客户端套接字, 且在监听他们的读事件. 如果客户端向服务器发送一个命令请求, 那么读事件会被触发

调用命令请求处理器执行, 处理器会读取客户端发来的数据, 进行命令解析并调用相关程序执行。

执行命令就会产生相应的返回值, 为了将这些返回值传送回客户端, 服务器会将客户端套接字的```AE_WRITEABLE```事件与命令回复处理器进行关联.当客户端尝试读取命令回复时, 客户端会产生```AE_WRITEABLE```事件, 触发命令回复器, 把回复写入套接字中，然后服务器会解除与命令回复处理器的关联



### 时间事件





