### Binder驱动程序

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
        int min_priority : 8；
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





