### 1. 概述

我们**最先**使用的就是 Netty 的 Bootstrap 和 ServerBootstrap 组这两个“**启动器**”组件。它们在 `transport` 模块的 `bootstrap` 包下实现

<img src="http://static2.iocoder.cn/images/Netty/2018_04_01/01.png" alt="`bootstrap` 包" style="zoom:50%;" />

我们可以看到三个以 Bootstrap 结尾的类，类图如下

<img src="http://static2.iocoder.cn/images/Netty/2018_04_01/02.png" alt="Bootstrap 类图" style="zoom:50%;" />

### 2. ServerBootstrap 示例

```java
/**
 * Echoes back any received data from a client.
 */
public final class EchoServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }

        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        //Create Handler object
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            //Create ServerBootStrap object
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup) //set EventLoopGrouop
             .channel(NioServerSocketChannel.class) //设置要被实例化的NioServerSocketChannel
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO)) //设置NioServerSocketChannel的处理器
             .childHandler(new ChannelInitializer<SocketChannel>() { //设置连入服务端的 Client 的 SocketChannel 的处理器
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });

            // Start the server.
            //绑定端口，并同步等待成功，即启动服务端
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

### 3. AbstractBootstrap

`#self()` 方法，返回自己。代码如下：

```java
private B self() {    return (B) this;}
```

`#group(EventLoopGroup group)` 方法，设置 EventLoopGroup 到 `group` 中。代码如下：

```java
public B group(EventLoopGroup group) {    
  if (group == null) {       
    throw new NullPointerException("group");    
  }    
  if (this.group != null) { // 不允许重复设置        
    throw new IllegalStateException("group set already");    
  }    
  this.group = group;    
  return self();
}
```

- 最终调用 `#self()` 方法，返回自己。实际上，AbstractBootstrap 整个方法的调用，基本都是链式调用。

`#channel(Class<? extends C> channelClass)` 方法，设置要被**实例化**的 Channel 的类。代码如下：

```java
public B channel(Class<? extends C> channelClass) {    
  if (channelClass == null) {        
    throw new NullPointerException("channelClass");    
  }    
  return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

`io.netty.channel.ChannelFactory` ，Channel 工厂**接口**，用于创建 Channel 对象。代码如下：

```java
public interface ChannelFactory<T extends Channel> extends io.netty.bootstrap.ChannelFactory<T> {

    /**
     * Creates a new channel.
     *
     * 创建 Channel 对象
     *
     */
    @Override
    T newChannel();

}
```

`io.netty.channel.ReflectiveChannelFactory` ，实现 ChannelFactory 接口，反射调用默认构造方法，创建 Channel 对象的工厂实现类。代码如下：

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    /**
     * Channel 对应的类
     */
    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            // 反射调用默认构造方法，创建 Channel 对象
            return clazz.getConstructor().newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

}
```

`#validate()` 方法，校验配置是否正确。代码如下

```java
public B validate() {
    if (group == null) {
        throw new IllegalStateException("group not set");
    }
    if (channelFactory == null) {
        throw new IllegalStateException("channel or channelFactory not set");
    }
    return self();
}
```

- 在 `#bind(...)` 方法中，绑定本地地址时，会调用该方法进行校验。

`#bind(...)` 方法，绑定端口，启动服务端。代码如下：

```java
public ChannelFuture bind() {
    // 校验服务启动需要的必要参数
    validate();
    SocketAddress localAddress = this.localAddress;
    if (localAddress == null) {
        throw new IllegalStateException("localAddress not set");
    }
    // 绑定本地地址( 包括端口 )
    return doBind(localAddress);
}

public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}

public ChannelFuture bind(String inetHost, int inetPort) {
    return bind(SocketUtils.socketAddress(inetHost, inetPort));
}

public ChannelFuture bind(InetAddress inetHost, int inetPort) {
    return bind(new InetSocketAddress(inetHost, inetPort));
}

public ChannelFuture bind(SocketAddress localAddress) {
    // 校验服务启动需要的必要参数
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    // 绑定本地地址( 包括端口 )
    return doBind(localAddress);
}
```

`#initAndRegister()` 方法，初始化并注册一个 Channel 对象，并返回一个 ChannelFuture 对象。代码如下：

```java
  final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

<img src="http://static2.iocoder.cn/images/Netty/2018_04_01/06.png" alt="Channel 类图" style="zoom:50%;" />

简单点来说，我们可以把 Netty Channel 和 Java 原生 Socket 对应，而 Netty NIO Channel 和 Java 原生 NIO SocketChannel 对象





