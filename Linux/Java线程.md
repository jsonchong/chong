### 线程的状态

线程创建之后，调用start()方法开始运行。当线程执行wait()方法之后，线程进入等待状态。进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而超时等待状态相当于在等待状态的基础上增加了超时限制，也就是超时时间到达时将会返回到运行状态。当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到阻塞状态。线程在执行Runnable的run()方法之后将会进入到终止状态。

### Daemon线程

Daemon属性需要在启动线程之前设置，不能在启动线程之后设置。

### 构造线程

在运行线程之前首先要构造一个线程对象，线程对象在构造的时候需要提供线程所需要的属性，如线程所属的线程组、线程优先级、是否是Daemon线程等信息。

```java
private void init(ThreadGroup g,Runnable target,String name,long stackSize,AccessControlContext acc){
  if(name == null){
    throw new NullPointException("name cannot be null");
  }
  Thread parent = currentThread();//当前线程就是该线程的父线程
  this.group = g;
  // 将daemon、priority属性设置为父线程的对应属性      
  this.daemon = parent.isDaemon();      
  this.priority = parent.getPriority();      
  this.name = name.toCharArray();      
  this.target = target;      
  setPriority(priority);
   // 将父线程的InheritableThreadLocal复制过来      
  if (parent.inheritableThreadLocals != null)       			               this.inheritableThreadLocals=
    ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);      		// 分配一个线程ID      
  tid = nextThreadID();
}
```

在上述过程中，一个新构造的线程对象是由其parent线程来进行空间分配的，而child线程继承了parent是否为Daemon、优先级和加载资源的contextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。至此，一个能够运行的线程对象就初始化好了，在堆内存中等待着运行。

### 启动线程

线程对象在初始化完成之后，调用start()方法就可以启动这个线程。线程start()方法的含义是：当前线程（即parent线程）同步告知Java虚拟机，只要线程规划器空闲，应立即启动调用start()方法的线程。

> 启动一个线程前，最好为这个线程设置线程名称，因为这样在使用jstack分析程序或者进行问题排查时，就会给开发人员提供一些提示，自定义的线程最好能够起个名字。

### 理解中断

中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。中断好比其他线程对该线程打了个招呼，其他线程通过调用该线程的interrupt()方法对其进行中断操作。

线程通过检查自身是否被中断来进行响应，线程通过方法isInterrupted()来进行判断是否被中断，**也可以调用静态方法Thread.interrupted()对当前线程的中断标识位进行复位**。如果该线程已经处于终结状态，即使该线程被中断过，在调用该线程对象的isInterrupted()时依旧会返回false。

从Java的API中可以看到，许多声明抛出InterruptedException的方法（例如Thread.sleep(long millis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedEx-ception，此时调用isInterrupted()方法将会返回false。

### 过期的suspend()、resume()和stop()

大家对于CD机肯定不会陌生，如果把它播放音乐比作一个线程的运作，那么对音乐播放做出的暂停、恢复和停止操作对应在线程Thread的API就是suspend()、resume()和stop()。

创建了一个线程PrintThread，它以1秒的频率进行打印，而主线程对其进行暂停、恢复和停止操作。

```java
public class Deprecated{
  	public static void main(String[] args){
      DateFormat format = new SimpleDateFormat("HH:mm:ss");
      Thread printThread = new Thread(new Runner(),"PrintThread");
      printThread.setDameon(true);
      printThread.start();
      TimeUnit.SECONDS.sleep(3);
      printThread.suspend();//将PrintThread进行暂停，输出工作停止
      System.out.Println("main suspend PrintThread at" + format.format(new Date()));
      
      TimeUnit.SECONDS.sleep(3);
      
      printThread.resume();//将printThread进行恢复，输出内容继续
      System.out.println("main resume PrintThread at" + format.format(new Date()));
      
      TimeUnit.SECONDS.sleep(3);
      
      printThread.stop();//将printThread进行终止，输出内容停止
      System.out.println("main stop PrintThread at" + format.format(new Date()));
      
      TimeUnit.SECONDS.sleep(3);
   
    }
  
  static class Runner implements Runnable{
    @Override
    public void run(){
      DateFormat format = new SimpleDateFormat("HH:mm:ss");
      while(true){
        System.out.println(Thread.currentThread().getName() + "Run at " + format.format(new Date()));
        SleepUtils.second(1);
      }
    }
  }
}
```

输出如下

PrintThread Run at 17:34:36

PrintThread Run at 17:34:37

PrintThread Run at 17:34:38

main suspend PrintThread at 17:34:39

main resume PrintThread at 17:34:42

PrintThread Run at 17:34:42

PrintThread Run at 17:34:43

PrintThread Run at 17:34:44

main stop PrintThread at 17:34:45

通过示例的输出可以看到，suspend()、resume()和stop()方法完成了线程的暂停、恢复和终止工作，而且非常“人性化”。但是这些API是过期的，也就是不建议使用的。

**不建议使用的原因主要有：以suspend()方法为例，在调用后，线程不会释放已经占有的资源（比如锁），而是占有着资源进入睡眠状态，这样容易引发死锁问题。同样，stop()方法在终结一个线程时不会保证线程的资源正常释放，通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定状态下。**

### volatile和synchronized关键字

Java支持多个线程同时访问一个对象或者对象的成员变量，**由于每个线程可以拥有这个变量的拷贝（虽然对象以及成员变量分配的内存是在共享内存中的，但是每个执行的线程还是可以拥有一份拷贝，这样做的目的是加速程序的执行，这是现代多核处理器的一个显著特性）**，所以程序在执行过程中，一个线程看到的变量并不一定是最新的。

关键字volatile可以用来修饰字段（成员变量），就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。

但是，过多地使用volatile是不必要的，因为它会降低程序执行的效率。

关键字synchronized可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。

synchronized本质是对一个对象的监视器（monitor）进行获取，而这个获取过程是排他的，也就是同一时刻只能有一个线程获取到由synchronized所保护对象的监视器。

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200923200853793.png" alt="image-20200923200853793" style="zoom:50%;" />

任意线程对Object（Object由synchronized保护）的访问，首先要获得Object的监视器。如果获取失败，线程进入同步队列，线程状态变为BLOCKED。当访问Object的前驱（获得了锁的线程）释放了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

### 等待/通知机制

1）使用wait()、notify()和notifyAll()时需要先对调用对象加锁。

2）调用wait()方法后，线程状态由RUNNING变为WAIT-ING，并将当前线程放置到对象的等待队列。

3）notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notifAll()的线程释放锁之后，等待线程才有机会从wait()返回。

4）**notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列**，被移动的线程状态由WAITING变为BLOCKED。

5）从wait()方法返回的前提是获得了调用对象的锁。

从上述细节中可以看到，等待/通知机制依托于同步机制，其**目的就是确保等待线程从wait()方法返回时能够感知到通知线程对变量做出的修改。**

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200924121015664.png" alt="image-20200924121015664" style="zoom:50%;" />

WaitThread首先获取了对象的锁，然后**调用对象的wait()方法，从而放弃了锁并进入了对象的等待队列WaitQueue中**，进入等待状态。由于WaitThread释放了对象的锁，NotifyThread随后获取了对象的锁，并调用对象的notify()方法，将WaitThread从WaitQueue移到SynchronizedQueue中，此时WaitThread的状态变为阻塞状态。NotifyThread释放了锁之后，WaitThread再次获取到锁并从wait()方法返回继续执行。

### 等待/通知的经典范式

该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。

1）获取对象的锁。

2）如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件。

3）条件满足则执行对应的逻辑。

对应的伪代码如下。

```java
synchronized(对象) {       
			while(条件不满足) {              
        对象.wait();       
      }       
  		对应的处理逻辑
}
```

通知方遵循如下原则。

1）获得对象的锁。

2）改变条件。

3）通知所有等待在对象上的线程。

```java
synchronized(对象){
  		改变条件
      对象.notifyAll();
}
```

### 一个简单的数据库连接池示例

我们使用等待超时模式来构造一个简单的数据库连接池,在示例中模拟从连接池中获取、使用和释放连接的过程，而客户端获取连接的过程被设定为等待超时的模式，也就是在1000毫秒内如果无法获取到可用连接，将会返回给客户端一个null。

调用方需要先调用fetchConnection(long)方法来指定在多少毫秒内超时获取连接，当连接使用完成后，需要调用releaseConnection(Connection)方法将连接放回线程池

```java
public class ConnectionPool {
  private LinkedList<Connection> pool = new LinkedList<Connection>();
  
  public ConnectionPool(int initialSize){
    if(initialSize > 0){
      for(int i = 0; i < initialSize; i++){
        pool.addLast(ConnectionDriver.createConnection());
      }
    }
  }
  
  public void releaseConnection(Connection connection){
    if(connection != null){
      synchronized(pool){
        //连接释放后需要进行通知，这样其他消费者能够感知到连接池中已经归还了一个连接
        pool.addLast(connection);
        pool.notifyAll();
      }
    }
  }
  
  public Connection fetchConnection(long mills) throws InterruptedException{
    synchronized (pool){
      if(mills <= 0){
        while(pool.isEmpty()){
          pool.wait();
        }
        return pool.removeFirst();
      }else{
        long future = System.currentTimeMills() + mills;
        long remaining = mills;
        while(pool.isEmpty() && remaining > 0){
          pool.wait(remaining);
          remaining = future - System.currentTimeMills();
        }
        Connection result  = null;
        if(!pool.isEmpty()){
          result = pool.removeFirst();
        }
        return result;
      }
    }
  }
}
```

考虑到只是个示例，我们通过动态代理构造了一个Connection，该Connection的代理实现仅仅是在commit()方法调用时休眠100毫秒

```java
public class ConnectionDriver {
  static class ConnectionHandler implements InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)throws Throwable{
      if(method.getName().equals("commit")){
        TimeUnit.MILLISECONDS.sleep(100);
      }
      return null;
    }
  }
  
  public static final Connection createConnection() {
    return (Connection)Proxy.newProxyInstance(ConnectionDriver.class.getClassLoader(),new Class<>[]{Connection.class},new ConnectionHandler());
  }
  
}
```

```java
public class ConnectionPoolTest {
  static ConnectionPool pool = new ConnectionPool(10);
  
  //保证所有ConnectionRunner能够同时开始
  static CountDownLatch start = new CountDownLatch(1);
  
  //main线程会等待所有ConnectionRunner结束后才能继续执行
  static CountDownLatch end;
  
  public static void main(String[] args) throws Exception{
    int threadCount = 10;
    end = new CountDownLatch(threadCount);
    int count = 20;
    AtomicInteger got = new AtomicInteger();       
    AtomicInteger notGot = new AtomicInteger();
    for(int i = 0;i < threadCount;i++){
      Thread thread = new Thread(new ConnectionRunner(count,got,notGot),"ConnectionRunnerThread");
      thread.start();
    }
    start.countDown(); //countDown()：使 CountDownLatch 初始值 N 减 1；
    end.await(); //调用该方法的线程等到构造方法传入的 N 减到 0 的时候，才能继续往下执行；
    System.out.println("total invoke: " + (threadCount * count));
    System.out.println("got connection:  " + got);        
    System.out.println("not got connection " + notGot);
  }
  
  static class ConnectionRunner implements Runnable {
    int count;
    AtomicInteger got;
    AtomicInteger notGot;
    
    public ConnectionRunner(int count,AtomicInteger got,AtomicInteger notGot){
      this.count = count;
      this.got = got;
      this.notGot = notGot;
    }
    public void run(){
      try{
        start.await();
      } catch(Exception ex){
        
      }
      while(count > 0){
        // 从线程池中获取连接，如果1000ms内无法获取到，将会返回null                   
        // 分别统计连接获取的数量got和未获取到的数量notGot
        try{
          Connection connection = pool.fetchConnection(1000);
          if(connection != null){
            try{
              connection.createStatement();
              connection.commit();
            }finally{
              pool.releaseConnection(connection);
              got.incrementAndGet();
            }
          }else{
            notGot.incrementAndGet();
          }
        }catch(Exception ex){
          
        }finally{
          count--;
        }
      }
      end.countDown();
    }
  }
}
```

上述示例中使用了CountDownLatch来确保Connection-RunnerThread能够同时开始执行

> CountDownLatch:在多线程协作完成业务功能时，有时候需要等待其他多个线程完成任务之后，主线程才能继续往下执行业务功能，在这种的业务场景下，通常可以使用 Thread 类的 join 方法，让主线程等待被 join 的线程执行完之后，主线程才能继续往下执行。当然，使用线程间消息通信机制也可以完成。其实，java 并发工具类中为我们提供了类似“倒计时”这样的工具类，可以十分方便的完成所说的这种业务场景。

客户端连接获取出现问题，是系统的一种自我保护机制。数据库连接池的设计也可以复用到其他的资源获取的场景，针对昂贵资源（比如数据库连接）的获取都应该加以超时限制。





