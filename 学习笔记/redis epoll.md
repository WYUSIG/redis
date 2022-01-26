### Redis为什么选择多路复用

单线程服务器模型，面临的最大的问题就是，一个线程如何处理多个客户端请求？
解决这种问题的办法就是「IO 多路复用」。它本质上是应用层不用维护多个客户端的连接状态，而是把它们「托管」给了操作系统，操作系统维护这些连接的状态变化，
之后应用层只管问操作系统，哪些 socket 有数据可读/可写就好了，大大简化了应用层的复杂度

> Linux 针对每一个套接字都会有一个文件描述符，也就是一个非负整数，用来唯一标识该套接字。所以，在多路复用机制的函数中，Linux 通常会用文件描述符作为参数。

### select

```c
int select (int __nfds, fd_set *__readfds, fd_set *__writefds, fd_set *__exceptfds, struct timeval *__timeout)
```

参数：
 - __nfds： 套接字的文件描述符数量
 - *__readfds：读数据事件
 - *__writefds：写数据事件
 - *__exceptfds：异常事件
 - *__timeout：监听时阻塞等待的超时时长

fd_set结构

```c
typedef struct { 
    ...
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS]; 
    ...
} fd_set
```

其中__FD_SETSIZE固定1024，__NFDBITS固定32

fd_set 结构体的定义，其实就是一个 long int 类型的数组，该数组中一共有 32 个元素（1024/32=32），每个元素是 32 位（long int 类型的大小），而每一位可以用来表示一个文件描述符的状态。可以监听 1024 个描述符

### poll

```c
int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```

参数：
 - *__fds：pollfd结构体参数，里面包含了需要监听的套接字文件描述符号、事件
 - __nfds：*__fds数组元素个数
 - __timeout：监听时阻塞等待的超时时长

pollfd结构

```c

#define POLLRDNORM  0x040       //可读事件
#define POLLWRNORM  0x100       //可写事件
#define POLLERR     0x008       //错误事件

struct pollfd {
    int fd;         //进行监听的文件描述符
    short int events;       //要监听的事件类型
    short int revents;      //实际发生的事件类型
};
```

### epoll

```c
sock_fd = socket() //创建套接字
epfd = epoll_create(EPOLL_SIZE); //创建epoll实例  **********
//创建epoll_event变量
struct epoll_event ee
//监听读事件
ee.events = EPOLLIN;
//监听的文件描述符是刚创建的监听套接字
ee.data.fd = sock_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ee);    //*********
//等待返回已经就绪的描述符 
n = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);   //*********
//遍历ep_events
```

epoll_event结构

```c
typedef union epoll_data
{
  ...
  int fd;  //记录文件描述符
  ...
} epoll_data_t;

struct epoll_event
{
  uint32_t events;  //epoll监听的事件类型
  epoll_data_t data; //应用程序数据
};
```


#### select、poll、epoll区别

- select最多监听1024个文件描述符，而且返回后仍然需要遍历每个文件描述符，检测该描述符是否就绪，然后再进行处理。

- poll可以自定义监听的文件描述符数量，但是返回后仍然需要遍历每个文件描述符，检测该描述符是否就绪，然后再进行处理。

- epoll可以自定义监听的文件描述符数量，epoll 实例内部维护了两个结构，分别是记录要监听的文件描述符和已经就绪的文件描述符，使用 epoll_wait 函数直接就能获取就绪的文件描述符



### 附录

#### select使用示例

```c
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字

fd_set rset;  //被监听的描述符集合，关注描述符上的读事件
 
int max_fd = sock_fd

//初始化rset数组，使用FD_ZERO宏设置每个元素为0 
FD_ZERO(&rset);
//使用FD_SET宏设置rset数组中位置为sock_fd的文件描述符为1，表示需要监听该文件描述符
FD_SET(sock_fd,&rset);

//设置超时时间 
struct timeval timeout;
timeout.tv_sec = 3;
timeout.tv_usec = 0;
 
while(1) {
   //调用select函数，检测rset数组保存的文件描述符是否已有读事件就绪，返回就绪的文件描述符个数
   n = select(max_fd+1, &rset, NULL, NULL, &timeout);
 
   //调用FD_ISSET宏，在rset数组中检测sock_fd对应的文件描述符是否就绪
   if (FD_ISSET(sock_fd, &rset)) {
       //如果sock_fd已经就绪，表明已有客户端连接；调用accept函数建立连接
       conn_fd = accept();
       //设置rset数组中位置为conn_fd的文件描述符为1，表示需要监听该文件描述符
       FD_SET(conn_fd, &rset);
   }

   //依次检查已连接套接字的文件描述符
   for (i = 0; i < maxfd; i++) {
        //调用FD_ISSET宏，在rset数组中检测文件描述符是否就绪
       if (FD_ISSET(i, &rset)) {
         //有数据可读，进行读数据处理
       }
   }
}
```

#### poll使用示例

```c

int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字

//poll函数可以监听的文件描述符数量，可以大于1024
#define MAX_OPEN = 2048

//pollfd结构体数组，对应文件描述符
struct pollfd client[MAX_OPEN];

//将创建的监听套接字加入pollfd数组，并监听其可读事件
client[0].fd = sock_fd;
client[0].events = POLLRDNORM; 
maxfd = 0;

//初始化client数组其他元素为-1
for (i = 1; i < MAX_OPEN; i++)
    client[i].fd = -1; 

while(1) {
   //调用poll函数，检测client数组里的文件描述符是否有就绪的，返回就绪的文件描述符个数
   n = poll(client, maxfd+1, &timeout);
   //如果监听套件字的文件描述符有可读事件，则进行处理
   if (client[0].revents & POLLRDNORM) {
       //有客户端连接；调用accept函数建立连接
       conn_fd = accept();

       //保存已建立连接套接字
       for (i = 1; i < MAX_OPEN; i++){
         if (client[i].fd < 0) {
           client[i].fd = conn_fd; //将已建立连接的文件描述符保存到client数组
           client[i].events = POLLRDNORM; //设置该文件描述符监听可读事件
           break;
          }
       }
       maxfd = i; 
   }
   
   //依次检查已连接套接字的文件描述符
   for (i = 1; i < MAX_OPEN; i++) {
       if (client[i].revents & (POLLRDNORM | POLLERR)) {
         //有数据可读或发生错误，进行读数据处理或错误处理
       }
   }
}
```

#### epoll使用示例

```c
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字
    
epfd = epoll_create(EPOLL_SIZE); //创建epoll实例，
//创建epoll_event结构体数组，保存套接字对应文件描述符和监听事件类型    
ep_events = (epoll_event*)malloc(sizeof(epoll_event) * EPOLL_SIZE);

//创建epoll_event变量
struct epoll_event ee
//监听读事件
ee.events = EPOLLIN;
//监听的文件描述符是刚创建的监听套接字
ee.data.fd = sock_fd;

//将监听套接字加入到监听列表中    
epoll_ctl(epfd, EPOLL_CTL_ADD, sock_fd, &ee); 
    
while (1) {
   //等待返回已经就绪的描述符 
   n = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1); 
   //遍历所有就绪的描述符     
   for (int i = 0; i < n; i++) {
       //如果是监听套接字描述符就绪，表明有一个新客户端连接到来 
       if (ep_events[i].data.fd == sock_fd) { 
          conn_fd = accept(sock_fd); //调用accept()建立连接
          ee.events = EPOLLIN;  
          ee.data.fd = conn_fd;
          //添加对新创建的已连接套接字描述符的监听，监听后续在已连接套接字上的读事件      
          epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ee); 
                
       } else { //如果是已连接套接字描述符就绪，则可以读数据
           ...//读取数据并处理
       }
   }
}
```