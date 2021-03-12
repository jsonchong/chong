### 急于IO多路复用的并发编程

假设需要编写一个echo服务器，它也能对用户从标准输入键入的交互命令做出响应。在这种情况下，服务器必须响应两个相互独立的I/O事件：1.网络客户端发起连接请求，2.用户在键盘上键入命令行。我们先等待哪个事件呢？，如果在accept中等待一个连接请求，我们就不能响应输入命令。类似地，如果在read中等待一个输入命令，我们就不能响应任何连接请求。

针对这种困境的一个解决办法是I/O多路复用技术。基本的思路就是使用select函数，要求内核挂起进程，只有在一个或多个I/O事件发生后，才将控制返回给应用程序。

用open_listenfd函数打开一个监听描述符，然后使用FD_ZERO创建一个空的读集合

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201029150149509.png" alt="image-20201029150149509" style="zoom:50%;" />

接下来，我们定义由描述符0(标准输入)和描述符3(监听描述符)组成的读集合：

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201029150232943.png" alt="image-20201029150232943" style="zoom:50%;" />

开始服务器循环，不调用accept函数来等待一个连接请求，而是调用select函数，这个函数会一直阻塞，直到监听描述符或者标准输入准备好可以读，例如当用户按回车键，使得标准输入描述符变为可读时，select会返回的ready_set的值。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201029150512024.png" alt="image-20201029150512024" style="zoom:50%;" />

```c
#include "csapp.h"
void echo(int connfd);
void command(void);

int main(int argc, char **argv){
  int listenfd,connfd;
  socklen_t clientlen;
  struct sockaddr_storage clientaddr;
  fd_set read_set,ready_set;
  
  if(argc != 2){
    fprintf(stderr,"usage: %s<port>\n",argv[0]);
  }
  
  listenfd = Open_listenfd(argv[1]);
  
  FD_ZERO(&read_set); /*clear read set*/
  FD_SET(STDIN_FILENO,&read_set); /*Add stdin to read set*/
  FD_SET(listenfd,&read_set); /*Add listenfd to read set*/
  
  while(1){
    ready_set = read_set;
    Select(listenfd+1, &ready_set,NULL,NULL,NULL);
    if(FD_ISSET(STDIN_FILENO,&ready_set))
      command(); /*Read command line from stdin*/
    if(FD_ISSET(listenfd,&ready_set)){
      clientlen = sizeof(struct sockaddr_storage);
      connfd = Accept(listenfd,(SA *)&clientaddr,&clientlen);
      echo(connfd); /*Echo client input until EOF*/
      Close(connfd);
    }
  }
  
}


```

### IO多路复用技术的优劣

它比进程的设计给了程序员更多的对程序行为的控制。例如，我们可以设想编写一个事件驱动的并发服务器，为某些客户端提供它们需要的服务并且运行在单一进程上下文中，因此每个逻辑流都能访问该进程的全部地址空间。这使得在流之间共享数据变得很容易。最后事件驱动设计常常比急于进程的设计高效的多，因为它们不需要进程上下文切换来调度新的流。

事件驱动设计一个明显的缺点就是编码复杂。我们事件驱动的并发echo服务器需要的代码比基于进程的服务器多三倍，并且随着并发粒度的减小，复杂性还会上升。这里的粒度是指每个逻辑流每个时间片执行的指令数量。例如，在示例并发服务器中，并发粒度是读一个完整的文本行所需要的指令数量。只要某个逻辑流正忙于读一个文本行，其他逻辑流就不可能有进展。



