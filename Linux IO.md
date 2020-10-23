### Unix I/O

所有的的I/O设备(例如网络，磁盘和终端)都被模型化为文件，而所有的输入输出都被当作对相应文件的读和写来执行。这种将设备优雅地映射为文件方式，允许Linux内核引出一个相对简单，低级的应用接口，为Unix I/O，使得所有的输入输出都能以一种统一且一致的方式来执行。

### 读和写文件

```c
#include <unistd.h>
 ssize_t read(int fd, void *buf, size_t n); //若成功返回读取的字节数

 ssize_t write(int fd, const void *buf, size_t n);//若成功返回写的字节数
```

read函数从描述符fd的当前文件位置复制最多n个字节到内存位置buf。返回值-1表示一个错误,返回值0表示EOF

write函数从内存位置buf复制至多n个字节到描述符fd的当前文件位置

下例展示了使用read和write调用一次一个字节从标准输入复制到标准输出

```c
#include "csapp.h"

int main(void){
  char c;
  
  while(Read(STDIN_FILENO, &c, 1) != 0)
    Write(STDOUT_FILENO, &c, 1);
  
  exit(0);
}
```

实际上，除了EOF, 当你在读磁盘文件时，将不会遇到不足值，而且在写磁盘文件时，也不会遇到不足值，然而，如果想创建健壮的(可靠的)诸如Web服务器这样的网络应用，就必须反复调用read和write处理不足值，直到所有需要的字节都传送完毕。

### I/O缓冲的类型

在使用`stdio`库中文件写操作相关的函数（如:`printf`, `fputc`, `fputs`, `fwrite`）时，待写入数据从用户空间内存到内核空间内存、再到磁盘会经过以下3类缓冲

1. stdio库的缓冲区
2. 文件I/O的内核缓冲区的高速缓存
3. 磁盘驱动器内置高速缓存

![stdio buffer](https://www.litreily.top/assets/linux/stdio_buffer.png)

如上图所示，`stdio`库实现的缓冲位于用户空间内存当中，该缓冲区A会缓冲大块的文件数据以减少系统调用（如: `read`, `write`）。

需要知道的是，`stdio`库函数内部会调用底层的系统调用，如`fgets`调用`read`，`fputs`调用`write`。但是在调用之前，

- 对于读操作，库函数会先检查缓冲区A内是否已有所需数据，如果有则直接从缓冲区A读取；否则先执行系统调用`read`，从内核缓冲区B中读取数据到缓冲区A，然后从缓冲区A读取数据
- 对于写操作，库函数会先检查缓冲区A是否还有空闲，如果有则先存入缓冲区A；否则先执行库函数`fflush`，将缓冲区A中数据刷新至内核缓冲区B，然后将当前待写入数据写入缓冲区A

对于`stdio`库的缓冲数据，在执行库函数之后的某一时刻，系统会通过`fflush`函数将数据刷新至内核缓冲区。当然，我们也可以手动执行`fflush`函数强制刷新数据至内核缓冲区。

### 文件I/O的内核缓冲

不管使不使用`stdio`库函数，最终都会直接或间接的调用`open`, `read`, `write`, `lseek`等系统调用读写文件I/O，那么系统就会在写操作后将数据存入内核缓冲区，但此时还并未存入磁盘。

也就是说，在执行`write`后，函数直接返回，但数据只是存在内核缓冲区中。当有新的读取请求时，会先在内核缓冲区中查找，如果有则直接返回；如果没有则先从磁盘读入大块数据至内核缓冲区，这样可以减少磁盘读写操作。毕竟，相比于系统调用和用户空间与内核空间之间的数据传输，磁盘读写所花费的时间要长得多。

### RIO函数

带缓冲版本的read函数

```c
#define RIO_BUFSIZE 8192

typedef struct {
  int rio_fd;        //Descriptor for this internal buf
  int rio_cnt;       // Unread bytes in internal buf
  char *rio_bufptr;  // Next unread byte in internal buf
  char rio_buf[RIO_BUFSIZE];  //Internal buffer
}
```

```c
void rio_readinitb(rio_t *rp, int fd){
	rp->rio_fd = fd;
  rp->rio_cnt = 0;
  rp->rio_bufptr = rp->rio_buf;
}
```

```c
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n){
  int cnt;
  
  while(rp->rio_cnt <= 0){  //refill if buf is empty
    rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf));
    
    if(rp->rio_cnt < 0){
      if(errno != EINTR)
        return -1;
    }
    
    else if(rp->rio_cnt == 0)  //EOF
    		return 0;
    else
      rp->rio_bufptr = rp->rio_buf;// reset buffer ptr
  }
  
  cnt = n;
  if(rp->rio_cnt < n)
    cnt = rp->rio_cnt;
  memcpy(usrbuf, rp->rio_bufptr, cnt);
  rp->rio_bufptr += cnt;
  rp->rio_cnt -= cnt;
  return cnt;
}
```



























