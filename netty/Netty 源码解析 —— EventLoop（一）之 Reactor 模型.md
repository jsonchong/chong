### 1. 概述

我们来看看 Reactor 模型的**核心思想**：

将关注的 I/O 事件注册到多路复用器上，一旦有 I/O 事件触发，将事件分发到事件处理器中，执行就绪 I/O 事件对应的处理函数中。模型中有三个重要的组件：

- 多路复用器：由操作系统提供接口，Linux 提供的 I/O 复用接口有select、poll、epoll 。
- 事件分离器：将多路复用器返回的就绪事件分发到事件处理器中。
- 事件处理器：处理就绪事件处理函数。

Reactor 示例代码如下：

```java
/**
* 等待事件到来，分发事件处理
*/
class Reactor implements Runnable {
  private Reactor() throws Exception {
      SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
      // attach Acceptor 处理新连接
      sk.attach(new Acceptor());
  }

	@Override
  public void run() {
      try {
          while (!Thread.interrupted()) {
              selector.select();
              Set selected = selector.selectedKeys();
              Iterator it = selected.iterator();
              while (it.hasNext()) {
                  it.remove();
                  //分发事件处理
                  dispatch((SelectionKey) (it.next()));
              }
          }
      } catch (IOException ex) {
          //do something
      }
  }

  void dispatch(SelectionKey k) {
      // 若是连接事件获取是acceptor
      // 若是IO读写事件获取是handler
      Runnable runnable = (Runnable) (k.attachment());
      if (runnable != null) {
          runnable.run();
      }
  }

}
```

- 有新连接到来触发 `OP_ACCEPT` 事件之后， 交由 Acceptor 进行处理。
- 有 IO 读写事件之后，交给 Handler 处理。

- 在获取到 Client 相关的 SocketChannel 之后，绑定到相应的 Handler 上。
- 对应的 SocketChannel 有读写事件之后，基于 Reactor 分发，Handler 就可以处理了。

该模型适用于处理器链中业务处理组件能快速完成的场景。不过，这种单线程模型不能充分利用多核资源，**所以实际使用的不多**

### 单 Reactor 多线程模型

相对于第一种单线程的模式来说，在处理业务逻辑，也就是获取到 IO 的读写事件之后，交由线程池来处理，这样可以减小主 Reactor 的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。

```java

class MultiThreadHandler implements Runnable {
  public static final int READING = 0, WRITING = 1;
  int state;
  final SocketChannel socket;
  final SelectionKey sk;

  //多线程处理业务逻辑
  ExecutorService executorService =      Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());


  public MultiThreadHandler(SocketChannel socket, Selector sl) throws Exception {
      this.state = READING;
      this.socket = socket;
      sk = socket.register(selector, SelectionKey.OP_READ);
      sk.attach(this);
      socket.configureBlocking(false);
  }

  @Override
  public void run() {
      if (state == READING) {
          read();
      } else if (state == WRITING) {
          write();
      }
  }

  private void read() {
      //任务异步处理
      executorService.submit(() -> process());

      //下一步处理写事件
      sk.interestOps(SelectionKey.OP_WRITE);
      this.state = WRITING;
  }

  private void write() {
      //任务异步处理
      executorService.submit(() -> process());

      //下一步处理读事件
      sk.interestOps(SelectionKey.OP_READ);
      this.state = READING;
  }

  /**
    * task 业务处理
    */
  public void process() {
      //do IO ,task,queue something
  }
}
```

### 多 Reactor 多线程模型

第三种模型比起第二种模型，是将 Reactor 分成两部分：

1. mainReactor 负责监听 ServerSocketChannel ，用来处理客户端新连接的建立，并将建立的客户端的 SocketChannel 指定注册给 subReactor 。
2. subReactor 维护自己的 Selector ，基于 mainReactor 建立的客户端的 SocketChannel 多路分离 IO 读写事件，读写网络数据。对于业务处理的功能，另外扔给 worker 线程池来完成。

```java
/**
* 多work 连接事件Acceptor,处理连接事件
*/
class MultiWorkThreadAcceptor implements Runnable {

  // cpu线程数相同多work线程
  int workCount = Runtime.getRuntime().availableProcessors();
  SubReactor[] workThreadHandlers = new SubReactor[workCount];
  volatile int nextHandler = 0;

  public MultiWorkThreadAcceptor() {
      this.init();
  }

  public void init() {
      nextHandler = 0;
      for (int i = 0; i < workThreadHandlers.length; i++) {
          try {
              workThreadHandlers[i] = new SubReactor();
          } catch (Exception e) {
          }
      }
  }

  @Override
  public void run() {
      try {
          SocketChannel c = serverSocket.accept();
          if (c != null) {// 注册读写
              synchronized (c) {
                  // 顺序获取SubReactor，然后注册channel 
                  SubReactor work = workThreadHandlers[nextHandler];
                  work.registerChannel(c);
                  nextHandler++;
                  if (nextHandler >= workThreadHandlers.length) {
                      nextHandler = 0;
                  }
              }
          }
      } catch (Exception e) {
      }
  }
}
```

```java
/**
* 多work线程处理读写业务逻辑
*/
class SubReactor implements Runnable {
  final Selector mySelector;

  //多线程处理业务逻辑
  int workCount =Runtime.getRuntime().availableProcessors();
  ExecutorService executorService = Executors.newFixedThreadPool(workCount);


  public SubReactor() throws Exception {
      // 每个SubReactor 一个selector 
      this.mySelector = SelectorProvider.provider().openSelector();
  }

  /**
    * 注册chanel
    *
    * @param sc
    * @throws Exception
    */
  public void registerChannel(SocketChannel sc) throws Exception {
      sc.register(mySelector, SelectionKey.OP_READ | SelectionKey.OP_CONNECT);
  }

  @Override
  public void run() {
      while (true) {
          try {
          //每个SubReactor 自己做事件分派处理读写事件
              selector.select();
              Set<SelectionKey> keys = selector.selectedKeys();
              Iterator<SelectionKey> iterator = keys.iterator();
              while (iterator.hasNext()) {
                  SelectionKey key = iterator.next();
                  iterator.remove();
                  if (key.isReadable()) {
                      read();
                  } else if (key.isWritable()) {
                      write();
                  }
              }

          } catch (Exception e) {

          }
      }
  }

  private void read() {
      //任务异步处理
      executorService.submit(() -> process());
  }

  private void write() {
      //任务异步处理
      executorService.submit(() -> process());
  }

  /**
    * task 业务处理
    */
  public void process() {
      //do IO ,task,queue something
  }
}
```

从代码中，我们可以看到：

1. mainReactor 主要用来处理网络 IO 连接建立操作，通常，mainReactor 只需要一个，因为它一个线程就可以处理。
2. subReactor 主要和建立起来的客户端的 SocketChannel 做数据交互和事件业务处理操作。通常，subReactor 的个数和 CPU 个数**相等**，每个 subReactor **独占**一个线程来处理。



