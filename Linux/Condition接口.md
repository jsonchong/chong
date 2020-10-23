### Condition接口与示例

Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condi-tion对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的，换句话说，Condition是依赖Lock对象的。

Condition的使用方式比较简单，需要注意在调用方法前获取锁

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();

public void conditionWait() throws InterruptedException {
  lock.lock();
  try{
    condition.await();
  }finally{
    lock.unlock();
  }
}

public void conditionSignal() throws InterruptedException{
  lock.lock();
  try{
    condition.signal();
  }finally{
    lock.unlock();
  }
}

```

一般都会将Condition对象作为成员变量。当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition对象的signal()方法，通知当前线程后，当前线程才从await()方法返回，并且在返回前已经获取了锁

获取一个Condition必须通过Lock的newCondition()方法。

有界队列是一种特殊的队列，当队列为空时，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”

```java
public class BoundedQueue<T> {
  private Object[] items;
  private int addIndex,removeIndex,count;
  private Lock lock = new ReentrantLock();
  private Condition notEmpty = lock.newCondition();
  private Condition notFull = lock.newCondition();
  
  public BoundedQueue(int size){
    items = new Object[size];
  }
  
  //添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
  public void add(T t)throws InterruptedException{
    lock.lock();
    try{
      while(count == items.length)
        notFull.await();
      items[addIndex] = t;
      if(++addIndex == items.length)
        addIndex = 0;
      ++count;
      notEmpty.signal();
    }finally{
      lock.unlock();
    }
  }
  
  //由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
  public T remove() throws InterruptedException {
    lock.lock();
    try{
      while(count == 0)
        notEmpty.await();
      Object x = items[remove];
      if(++removeIndex == items.length)
        removeIndex = 0;
      --count;
      notFull.signal();
      return (T)x;
    }finally{
      lock.unlock();
    }
  }
  
  
}
```

首先需要获得锁，目的是确保数组修改的可见性和排他性。当数组数量等于数组长度时，表示数组已满，则调用not-Full.await()，当前线程随之释放锁并进入等待状态。如果数组数量不等于数组长度，表示数组未满，则添加元素到数组中，同时通知等待在notEmpty上的线程，数组中已经有新元素可以获取。

### Condition的实现分析

ConditionObject是同步器AbstractQueuedSynchronizer的内部类，因为Condition的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理。每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。

**等待队列**

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。事实上，节点的定义复用了同步器中节点的定义，也就是说，同步队列和等待队列中节点类型都是同步器的静态内部类AbstractQueuedSyn-chronizer.Node。

一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201010183605623.png" alt="image-20201010183605623" style="zoom:50%;" />

Condition拥有首尾节点的引用，而新增节点只需要将原有的尾节点nextWaiter指向它，并且更新尾节点即可。上述节点引用更新的过程并没有使用CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的

在Object的监视器模型上，一个对象拥有一个同步队列和等待队列，而并发包中的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201010183926318.png" alt="image-20201010183926318" style="zoom:50%;" />

### 等待

调用Condition的await()方法会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态。当从await()方法返回时，当前线程一定获取了Condition相关联的锁。

如果从队列（同步队列和等待队列）的角度看await()方法，**当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。**

```java
public final void await() throws InterrupteedException {
  if(Thread.interupted())
    		throw new InterruptedException();
  //当前线程加入等待队列
  Node node = addConditionWaiter();
  //释放同步状态，也就是释放锁
  int savedState = fullyRelease(node);
  int interruptNode = 0;
  
  while(!isOnSyncQueue(node)){
    LockSupport.park(this);
    if((interruptMode = checkInteruptWhileWaiting(node)) != 0 )
      break;
  }
  if(acquireQueued(node, savedState) && interruptMode != THROW_IE)
    	interruptMode = REINTERRUPT;
  if(node.nextWaiter != null)
    unlinkCancelledWaiters();
  if(interruptMode != 0){
    reportInterruptAfterWait(interruptMode);
  }
}
```

调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，**该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。**

**当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态**。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedExcep-tion。

### 通知

调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20201019144012861.png" alt="image-20201019144012861" style="zoom:50%;" />

被唤醒后的线程，**将从await()方法中的while循环中退出（isOnSyncQueue(Node node)方法返回true，节点已经在同步队列中），进而调用同步器的acquireQueued()方法加入到获取同步状态的竞争中。**

成功获取同步状态（或者说锁）之后，被唤醒的线程将从先前调用的await()方法返回，此时该线程已经成功地获取了锁。













