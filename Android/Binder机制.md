### Binder驱动程序

我们知道，Android系统是基于Linux内核开发的。Linux内核提供了丰富的进程间通讯机制，如管道(Pipe)、信号(Signal)、消息队列(Message)、共享内存(Share Memory)和Socket等。然而，Android系统并没有采用这些传统的进程间通讯机制，而是开发了一套新的进程间通信机制-Binder。与传统的进程间通讯机制相比，Binder进程间通信机制在进程间传输数据时，只需要执行一次拷贝操作，因此不仅提高了效率，而且节省了内存空间。

Binder进程间通讯采用CS通信方式，其中提供服务的进程称为Server进程，而访问服务的进程称为Client进程。同一个Server进程可以同时运行多个组件来向Client进程提供服务，这些组件称为Service组件。同时一个Client进程也可以同时向多个Service组件请求服务，每一个请求都对应有一个Client组件，或者称为Service代理对象。Server进程和Client进程的通信要依靠运行在内核空间的Binder驱动程序来进行。Binder驱动程序向用户空间暴露了一个设备文件/dev/binder。

Client和Service和ServiceManager均是通过系统调用open,mmap和ioctl来访问设备文件/dev/binder，从而实现与Binder驱动程序的交互，而交互的目的就是为了能够间接地执行进程间通信。

Binder驱动程序实现在内核空间中，它的目录结构如下：

```
~/Android/kernel/goldfish
----drivers
    ----staging
        ----android
            ---- binder.h
            ---- binder.c
```

**struct binder_work**

**kernel/goldfish/drivers/staging/android/binder.c**

```c
struct binder_work { //用来描述待处理的工作项
        struct list_head entry； //entry表示将该结构体嵌入到一个宿主结构中
        enum {
                BINDER_WORK_TRANSACTION = 1，
                BINDER_WORK_TRANSACTION_COMPLETE，
                BINDER_WORK_NODE，
                BINDER_WORK_DEAD_BINDER，
                BINDER_WORK_DEAD_BINDER_AND_CLEAR，
                BINDER_WORK_CLEAR_DEATH_NOTIFICATION，
        } type； //成员变量type用来描述工作项的类型
          //根据成员变量type的取值，Binder 驱动程序就可以判断出一个binder_work结构体嵌入到什么类型的宿主结构中。
}；
```

**struct binder_node**

**kernel/goldfish/drivers/staging/android/binder.c**

```c
//结构体binder_node用来描述一个Binder实体对象。
//每一个Service组件在Binder驱动程序中都对应有一个Binder实体对象,用来描述它在内核中的状态
//Binder驱动程序通过强引用计数和弱引用计数技术来维护它们的生命周期。
struct binder_node { 
        int debug_id；
        struct binder_work work；
        union {
                struct rb_node rb_node；
                struct hlist_node dead_node；
        }；
        //指向一个Binder实体对象的宿主进程 
        //这些宿主进程通过一个binder_proc结构体来描述
        struct binder_proc *proc； 
        struct hlist_head refs；
        int internal_strong_refs；
        int local_weak_refs；
        int local_strong_refs；
        void __user *ptr；
        void __user *cookie；
        unsigned has_strong_ref : 1；
        unsigned pending_strong_ref : 1；
        unsigned has_weak_ref : 1；
        unsigned pending_weak_ref : 1；
        unsigned has_async_transaction : 1；
        unsigned accept_fds : 1；
        int min_priority : 8； //一个Binder实体对象在处理一个来自Client进程的请求时，所要求的处理线程，即Server进程中的一个线程
        struct list_head async_todo；
}；
```

宿主进程使用一个**红黑树来维护它内部所有的Binder实体对象**

如果一个Binder实体对象的宿主进程已经死亡了，那么这个Binder实体对象就会通过它的成员变量dead_node保存在一个全局的hash列表中。

由于一个Binder实体对象可能会同时被多个Client组件引用，因此，**Binder驱动程序就使用结构体binder_ref来描述这些引用关系，并且将引用了同一个Binder实体对象的所有引用都保存在一个hash列表中。这个hash列表通过Binder实体对象的成员变量refs来描述**，而Binder驱动程序通过这个成员变量就可以知道有哪些Client组件引用了同一个Binder实体对象。

成员变量internal_strong_refs和local_strong_refs均是用来描述一个Binder实体对象的强引用计数

而成员变量**local_weak_refs用来描述一个Binder实体对象的弱引用计数**。**当一个Binder实体对象请求一个Service组件来执行某一个操作时，会增加该Service组件的强引用计数或者弱引用计数，相应地，Binder实体对象会将其成员变量has_strong_ref和has_weak_ref的值设置为1**。当一个Service组件完成一个Binder实体对象所请求的操作之后，Binder实体对象就会请求减少该Service组件的强用计数或者弱引用计数。

成员变量ptr和cookie分别指向一个用户空间地址，它们用来描述用户空间中的一个Service组件，其中，成员变量cookie指向该Service组件的地址，而成员变量ptr指向该Service组件内部的一个引用计数对象(类型为weakref_impl)的地址。

成员变量has_async_transaction用来描述一个Binder实体对象是否正在处理一个异步事务,如果是，它的值就等于1，否则等于0。

每一个事务都关联着一个Binder实体对象，表示该事务的目标处理对象，即要求与该Binder实体对象对应的Service组件在指定的线程中处理该事务

```c
//struct binder_ref
//结构体binder_ref用来描述一个Binder引用对象，每一个Client组件在Binder驱动程序中都对应有一个Binder引用对象，用来描述它在内核中的状态。Binder驱动程序通过强引用计数和弱引用计数来维护它们的生命周期
struct binder_ref{ 
  //desc是一个句柄值，用来描述一个Binder引用对象的，在Client进程的用户空间中，一个Binder引用对象是使用一个句柄值来描述的，因此当client进程的通过Binder驱动程序来访问Service组件时，只需要指定一个句柄值，Binder驱动程序就可以通过该句柄值找到对应的Binder引用对象，然后再根据该Binder引用对象的成员变量node找到对应的Binder实体对象
  struct rb_node rb_node_desc;
  struct rb_mode rb_node_node;
  struct binder_proc *proc;
  //node用来描述一个Binder引用对象所引用的Binder实体对象，之前提到Binder实体对象都有一个hash列表，用来保存那些引用了它的Binder引用对象
  struct binder_node *node;
  uint32_t desc;
  int strong;
  int weak;
  struct binder_ref_death *death;
};
```

成员变量proc指向一个Binder引用对象的宿主进程。一个宿主进程使用两个红黑树来保存它内部所有的Binder引用对象，它们分别以句柄值和对应的Binder实体对象的地址来作为关键字保存这些Binder引用对象。

```c
//用来描述一个正在使用Binder进程间通信机制的进程。当一个进程调用函数open来打开设备文件/dev/binder时，Binder驱动程序就会为它创建一个binder_node结构体，并且保存在一个全局的hash列表中，而成员变量proc_node是hash列表中的一个节点
struct binder_proc{
  
};
```



### Binder设备文件打开过程

一个进程在使用Binder进程间通信机制之前，首先要调用函数open打开设备文件/dev/binder来获取一个文件描述符，然后才能通过这个文件描述符来和Binder驱动程序交互。首先为进程创建一个binder_proc结构体proc,Binder驱动程序将所有打开了设备文件/dev/binder的进程都加入到全局hash队列binder_procs中，通过遍历这个hash队列就可以知道系统当前有多少个进程使用Binder进程间通信机制。

### Binder设备文件的内存映射过程

进程打开了设备文件/dev/binder之后，还必须要调用函数mmap把这个设备文件映射到进程的地址空间，然后才可以使用Binder进程间通信机制。设备文件/dev/binder对应的是一个虚拟设备，将它映射到进程的地址空间目的是为进程分配内核缓冲区，以便它可以用来传输进程间通信数据。























