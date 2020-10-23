### ServiceManager

Service Manager是Binder进程间通讯的核心组件，**它扮演着Binder上下文管理者(Context Manager)的角色，同时负责管理系统中的Service组件，并且向Client组件提供获取Service代理对象的服务。**

Service Manager运行在一个独立的进程中，因此，Service组件和Client组件也需要通过进程间通信机制来和它交互，是一个特殊的Service组件。

Service Manager是由init进程负责启动的，而init进程是在系统启动时启动的。

```shell
system/core/rootdir/init.rc
  service	servicemanager /system/bin/servicemanager # service表明servicemanager是以服务形式启动的
  		user system  # user表明Service Manager是以系统用户身份system的身份运行的
  		critical #critical表明是ServiceManager是一个关键服务，关键服务不退出的，一旦退出系统重启
  		onrestart restart zygote  #一旦Service Manager被系统重启，zygote和media进程也需要重启
  		onrestart restart media
```

该程序的入口函数main的实现在文件service_manager.c中

```c
~Android/frameworks/base/cmd
----servicemanager
		----binder.h
		----binder.c
		----service_manager.c

frameworks/base/cmds/servicemanager/service_manager.c
void *svcmgr_handle;

int main(int argc, char **argv){
  struct binder_state *bs;
  void *svcmgr = BINDER_SERVICE_MANAGER;
  
  bs = binder_open(128 * 1024);
  
  if(binder_become_context_manager(bs)){
    LOGE("cannot become context manager (%s)\n",strerror(errno));
    return -1;
  }
  
  svcmgr_handle = svcmgr;
  binder_loop(bs, svmgr_handler);
  return 0;
  
}
```

启动过程分为三步：

1. 调用函数binder_open打开设备文件/dev/binder,以及将它映射到本进程的地址空间；
2. 调用函数binder_become_context_manager将自己注册为Binder进程间通信机制的上下文管理者
3. 调用函数binder_loop来循环等待和处理Client进程的通信请求

结构体binder_state

```c
frameworks/base/cmds/servicemanager/binder.c
struct binder_state{
  int fd;
  void *mapped;
  unsigned mapsize;
};
```

Service Manager打开设备文件之后/dev/binder之后，将得到的文件描述符保存在一个binder_state结构体的成员变量fd中，以便后面可以通过它来和Binder驱动程序交互。

> 一个进程如果要和Binder驱动程序交互，除了要打开设备文件/dev/binder之外，还需要将该设备文件映射到进程的地址空间，以便Binder驱动程序可以为它分配内核缓冲区来保存进程通信数据。

Service Manager需要将设备文件/dev/binder映射到自己的进程地址空间，并且将映射后得到的地址空间大小和起始地址保存在一个binder_state结构体的成员变量mapsize和mapped中。

宏BINDER_SERVICE_MANAGER的定义如下：

### 打开和映射Binder设备文件

函数binder_open用来打开设备文件/dev/binder,并且将它映射到进程的地址空间

```c
frameworks/base/cmds/servicemanager/binder.c
struct binder_state *binder_open(unsigned mapsize){
  struct binder_state *bs;
  
  bs = malloc(sizeof(*bs));
  
  if(!bs){
    errno = ENOMEM;
    return 0;
  }
  
  bs->fd = open("/dev/binder",O_RDWR);
  if (bs->fd < 0){
    fprintf(stderr,"binder: cannot open device (%s)\n",strerror(errno));
    goto fail_open;
  }
  
  bs->mapsize = mapsize;
  bs->mapped = mmap(NULL,mapsize,PROT_READ,MAP_PRIVATE,bs->fd,0);
  
  if(bs->mapped == MAP_FAILED) {
    fprintf(stderr,"binder: cannot map device (%s)\n",strerror(errno));
    goto fail_map;
  }
  
  return bs;
  
 fail_map:
  	close(bs->fd);
  
 fail_open:
  	free(bs);
  	return 0;
}
```



