### Redis事件类型

- IO事件：可读事件、可写事件、屏障事件
- 时间事件

### aeEventLoop

#### aeEventLoop结构

```c
//ae.h
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events IO事件数组*/
    aeFiredEvent *fired; /* Fired events 已触发事件数组*/
    aeTimeEvent *timeEventHead; //记录时间事件的链表头
    int stop;
    void *apidata; /* This is used for polling API specific data 和api接口调用相关数据*/
    aeBeforeSleepProc *beforesleep; //进入事件循环流程前执行的函数
    aeBeforeSleepProc *aftersleep;  //退出事件循环流程后执行的函数
    int flags;
} aeEventLoop;
```

#### aeEventLoop初始化

aeEventLoop初始化是在initServer中调用aeCreateEventLoop初始化的

```c
//server.c
initServer() {
    /...
    //调用aeCreateEventLoop函数创建aeEventLoop结构体，并赋值给server结构的el变量
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    //...
}
```

参数是setsize的大小是由maxclients和CONFIG_FDSET_INCR宏决定的，其中maxclients在redis.conf中配置，默认1000，而 CONFIG_FDSET_INCR = CONFIG_MIN_RESERVED_FDS+96

CONFIG_MIN_RESERVED_FDS = 32

#### aeCreateEventLoop初始化步骤

1、调用aeCreateEventLoop函数创建一个aeEventLoop结构体变量eventLoop

2、aeCreateEventLoop里面会调用aeApiCreate函数，以linux为例的话，就是创建epoll_create创建epoll实例

3、aeCreateEventLoop会把所有网络IO事件对应的文件描述符的掩码，初始化为AE_NONE，表示不对任何事件进行监听

```c
//ae.c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;
    
    //给eventLoop变量分配内存空间
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    //给IO事件、已触发事件分配内存空间
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    //...
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    //设置时间事件的链表头为NULL
    eventLoop->timeEventHead = NULL;
    /...
    //调用aeApiCreate函数，去实际调用操作系统提供的IO多路复用函数
    if (aeApiCreate(eventLoop) == -1) goto err;
    
    //将所有网络IO事件对应文件描述符的掩码设置为AE_NONE
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;
    
    //初始化失败后的处理逻辑，
    err:
    /...
}
```

### IO事件处理

#### IO事件结构体aeFileEvent

```c
//ae.h
typedef struct aeFileEvent {
    int mask; //掩码标记，包括可读事件、可写事件和屏障事件
    aeFileProc *rfileProc;   //处理可读事件的回调函数
    aeFileProc *wfileProc;   //处理可写事件的回调函数
    void *clientData;  //私有数据
} aeFileEvent;
```

#### IO事件创建

```c
//ae.c, initServer函数调用
/**
 * IO事件创建
 * @param eventLoop 事件循环流程结构体对象
 * @param fd io事件对应的文件描述符
 * @param mask 事件类型掩码
 * @param proc 事件处理回调函数
 * @param clientData 事件私有数据
 * @return
 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    //调用epoll_ctl添加监听事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

```c
//ae_epoll.c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata; //获取epoll实例
    struct epoll_event ee = {0}; /* avoid valgrind warning */ //创建epoll_event类型变量
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
    EPOLL_CTL_ADD : EPOLL_CTL_MOD; //如果文件描述符fd对应的IO事件已存在，则操作类型为修改，否则为添加
    
    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    //将可读或可写IO事件类型转换为epoll监听的类型EPOLLIN或EPOLLOUT
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd; //将要监听的文件描述符赋值给ee
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

#### 读事件处理

```c
//server.c
void initServer(void) {
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler, NULL) == AE_ERR)
        {
            serverPanic(
            "Unrecoverable error creating server.ipfd file event.");
        }
    }
}
```

可以注意到epoll_ctl的回调函数是acceptTcpHandler

```c
//networking.c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);
    
    while(max--) {
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
        if (errno != EWOULDBLOCK)
            serverLog(LL_WARNING,
            "Accepting client connection: %s", server.neterr);
            return;
        }
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip); //创建已连接套接字，传给acceptCommonHandler进行处理
    }
}
```

在acceptCommonHandler里面会调用

```c
//networking.c
client *createClient(int fd) {
    //...
    if (fd != -1) {
        /...
        //调用aeCreateFileEvent，监听读事件，对应客户端读写请求，使用readQueryFromclient回调函数处理
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        } 
    }
   /...
}
```
