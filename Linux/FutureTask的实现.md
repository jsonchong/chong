### FutureTask的实现

FutureTask的实现基于AbstractQueuedSynchronizer（以下简称为AQS）。java.util.concurrent中的很多可阻塞类（比如ReentrantLock）都是基于AQS来实现的。AQS是一个同步框架，它提供通用机制来原子性管理同步状态、阻塞和唤醒线程，以及维护被阻塞线程的队列。

每一个基于AQS实现的同步器都会包含两种类型的操作，如下。

- 至少一个acquire操作。这个操作阻塞调用线程，直到AQS的状态允许这个线程继续执行。FutureTask的acquire操作为get()/get（long timeout，TimeUnit unit）方法调用。
- 至少一个release操作。这个操作改变AQS的状态，改变后的状态可允许一个或多个阻塞线程被解除阻塞。FutureTask的release操作包括run()方法和cancel（…）方法。

FutureTask声明了一个内部私有的继承于AQS的子类Sync，对FutureTask所有公有方法的调用都会委托给这个内部子类。

具体来说，Sync实现了AQS的tryAcquireShared（int）方法和tryReleaseShared（int）方法，Sync通过这两个方法来检查和更新同步状态。

FutureTask.get()方法会调用AQS.acquireSharedInterrupt-ibly（int arg）方法，这个方法的执行过程如下。

1）调用AQS.acquireSharedInterruptibly（int arg）方法，这个方法首先会回调在子类Sync中实现的tryAcquireShared()方法来判断acquire操作是否可以成功。acquire操作可以成功的条件为：state为执行完成状态RAN或已取消状态CAN-CELLED，且runner不为null。

2）如果成功则get()方法立即返回。如果失败则到线程等待队列中去等待其他线程执行release操作。

3）当其他线程执行release操作（比如FutureTask.run()或FutureTask.cancel（…））唤醒当前线程后，当前线程再次执行tryAcquireShared()将返回正值1，当前线程将离开线程等待队列并唤醒它的后继线程

4）最后返回计算的结果或抛出异常。

FutureTask.run()的执行过程如下。

1）执行在构造函数中指定的任务（Callable.call()）。

2）以原子方式来更新同步状态（调用AQS.compareAnd-SetState（int expect，int update），设置state为执行完成状态RAN）。如果这个原子操作成功，就设置代表计算结果的变量result的值为Callable.call()的返回值，然后调用AQS.release-Shared（int arg）。

3）AQS.releaseShared（int arg）首先会回调在子类Sync中实现的tryReleaseShared（arg）来执行release操作（设置运行任务的线程runner为null，然会返回true）；AQS.release-Shared（int arg），然后唤醒线程等待队列中的第一个线程。

4）调用FutureTask.done()。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201030195830785.png" alt="image-20201030195830785" style="zoom:50%;" />



