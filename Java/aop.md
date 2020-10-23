### AOP探索

#### 引入

假设我想不依赖任何 AOP 方法，在特定方法的执行前后加上日志打印。

##### 第一种方式：写死代码

定义一个目标类接口

```java
public interface Status{
  void execute(String name)
}
```

```java
public class NoAop{
  public static void main(String args[]){
    Status status = new StatusImpl();
    status.execute("execute no Aop");
  }
}

class StatusImpl implements Status{
  @Override
  public void execute(String name){
    before();
    System.out.println("hello! " + name);
    after();
  }
  
  private void before(){System.out.println("Before");}
  
  private void after(){ System.out.println("After");}
}
```

把 before() 和 after() 方法写死在 execute() 方法体中，非常不优雅，我们改进一下。

##### 第二种方式：静态代理

```java
public class StaticProxy{
  public static void main(String args[]){
    Status status = new StatusProxy(new StaticProxyStatusImpl());
    status.execute("static proxy");
  }
}

class StatusProxy implements Status{
 	private StaticProxyStatusImpl status;
  
  public StatusProxy(StaticProxyStatusImpl status){
    this.status = status;
  }
  
  @Override
  public void execute(String name){
    before();
    status.execute(name);
    after();
  }
  
  private void before(){
    System.out.println("Before");
  }
  
  private void after(){
    System.out.println("After");
  }
  
}
```

但是存在一个问题，随着打印日志的需求增多，Proxy 类越来越多，我们能不能保持只有一个代理呢？这时候我们就需要用到 JDK 动态代理了。

##### 第三种方式：动态代理

新建动态代理类

```java
class JdkDynamicProxy implements InvocationHandler{
  private Object target;
  
  public JdkDynamicProxy(Object target){
    this.target = target;
  }
  
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
    before();
    Object result  = method.invoke(target, args);
    after();
    return result;
  } 
  
  public <T> T getProxy(){
    return (T)Proxy.newProxyInstance(target.getClass().getClassLoader(), 						   target.getClass().getInterfaces(),this);
  }
  
  private void before(){
    System.out.println("Before");
  }
  
  private void after(){
    System.out.println("After");
  }
}
```

客户端调用

```Java
public class DynamicProxy{
  public static void main(String args[]){
    Status status = new JdkDynamicProxy(new DynamicProxyStatusImpl()).getProxy();
    status.execute("DynamicProxy");
  }
}

class DynamicProxyStatusImpl implements Status{
  @Override
  public void execute(String name){
    System.out.println("hello " + name);
  }
}
```

##### AspectJ示例

比方说，我们要测量一个方法的性能（执行这个方法需要多长时间）。为此我们用一个 **@DebugTrace** 的注解标记我们的这个方法，并且无需在每个注解过的方法中编写代码，就可以通过 logcat 输出结果。我们的方法是使用 AspectJ 达到这个目的。

##### 工程结构

我们会把一个简单的示例应用拆分成两个 modules，第一个包含我们的 Android App 代码，第二个是一个 Android Library 工程，使用 AspectJ 织入代码（代码注入）。

你可能会想知道为什么我们用一个 Android Library 工程，而不是用一个纯的 Java Library：原因是为了使 AspectJ 能在 Android 上运行，我们必须在编译时做一些 hook。这只能使用 andorid-library gradle 插件完成

##### 创建注解

首先我们创建我们的Java注解。这个注解周期声明在 class 文件上（RetentionPolicy.CLASS），可以注解构造函数和方法（ElementType.CONSTRUCTOR 和 ElementType.METHOD）。因此，我们的 DebugTrace.java 文件看上是这样的：

```java
@Retention(RetentionPolicy.CLASS)
@Target({ ElementType.CONSTRUCTOR, ElementType.METHOD })
public @interface DebugTrace {}
```

##### 性能监控计时类

我已经创建了一个简单的计时类，包含 `start/stop` 方法。下面是 StopWatch.java 文件

```java
public class StopWatch {
  private long startTime;
  private long endTime;
  private long elapsedTime;

  public StopWatch() {
    //empty
  }

  private void reset() {
    startTime = 0;
    endTime = 0;
    elapsedTime = 0;
  }

  public void start() {
    reset();
    startTime = System.nanoTime();
  }

  public void stop() {
    if (startTime != 0) {
      endTime = System.nanoTime();
      elapsedTime = endTime - startTime;
    } else {
      reset();
    }
  }

  public long getTotalTimeMillis() {
    return (elapsedTime != 0) ? TimeUnit.NANOSECONDS.toMillis(endTime - startTime) : 0;
  }
}
```

##### Aspect 类

```java
/**
 * Aspect representing the cross cutting-concern: Method and Constructor Tracing.
 */
@Aspect
public class TraceAspect {

  private static final String POINTCUT_METHOD =
      "execution(@org.android10.gintonic.annotation.DebugTrace * *(..))";

  private static final String POINTCUT_CONSTRUCTOR =
      "execution(@org.android10.gintonic.annotation.DebugTrace *.new(..))";

  @Pointcut(POINTCUT_METHOD)
  public void methodAnnotatedWithDebugTrace() {}

  @Pointcut(POINTCUT_CONSTRUCTOR)
  public void constructorAnnotatedDebugTrace() {}

  @Around("methodAnnotatedWithDebugTrace() || constructorAnnotatedDebugTrace()")
  public Object weaveJoinPoint(ProceedingJoinPoint joinPoint) throws Throwable {
    MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
    String className = methodSignature.getDeclaringType().getSimpleName();
    String methodName = methodSignature.getName();

    final StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    Object result = joinPoint.proceed();
    stopWatch.stop();

    DebugLog.log(className, buildLogMessage(methodName, stopWatch.getTotalTimeMillis()));

    return result;
  }

  /**
   * Create a log message.
   *
   * @param methodName A string with the method name.
   * @param methodDuration Duration of the method in milliseconds.
   * @return A string representing message.
   */
  private static String buildLogMessage(String methodName, long methodDuration) {
    StringBuilder message = new StringBuilder();
    message.append("Gintonic --> ");
    message.append(methodName);
    message.append(" --> ");
    message.append("[");
    message.append(methodDuration);
    message.append("ms");
    message.append("]");

    return message.toString();
  }
}
```

##### 使 AspectJ 运行在 Anroid 上

现在，所有代码都可以正常工作了，但是，如果我们编译我们的例子，我们并没有看到任何事情发生。原因是我们必须使用 AspectJ 的编译器（ajc，一个java编译器的扩展）对所有受 aspect 影响的类进行织入。这就是为什么，我之前提到的，我们需要在 gradle 的编译 task 中增加一些额外配置，使之能正确编译运行

```groovy
import com.android.build.gradle.LibraryPlugin
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

buildscript {
  repositories {
    mavenCentral()
  }
  dependencies {
    classpath 'com.android.tools.build:gradle:0.12.+'
    classpath 'org.aspectj:aspectjtools:1.8.1'
  }
}

apply plugin: 'android-library'

repositories {
  mavenCentral()
}

dependencies {
  compile 'org.aspectj:aspectjrt:1.8.1'
}

android {
  compileSdkVersion 19
  buildToolsVersion '19.1.0'

  lintOptions {
    abortOnError false
  }
}

android.libraryVariants.all { variant ->
  LibraryPlugin plugin = project.plugins.getPlugin(LibraryPlugin)
  JavaCompile javaCompile = variant.javaCompile
  javaCompile.doLast {
    String[] args = ["-showWeaveInfo",
                     "-1.5",
                     "-inpath", javaCompile.destinationDir.toString(),
                     "-aspectpath", javaCompile.classpath.asPath,
                     "-d", javaCompile.destinationDir.toString(),
                     "-classpath", javaCompile.classpath.asPath,
                     "-bootclasspath", plugin.project.android.bootClasspath.join(
        File.pathSeparator)]

    MessageHandler handler = new MessageHandler(true);
    new Main().run(args, handler)

    def log = project.logger
    for (IMessage message : handler.getMessages(null, true)) {
      switch (message.getKind()) {
        case IMessage.ABORT:
        case IMessage.ERROR:
        case IMessage.FAIL:
          log.error message.message, message.thrown
          break;
        case IMessage.WARNING:
        case IMessage.INFO:
          log.info message.message, message.thrown
          break;
        case IMessage.DEBUG:
          log.debug message.message, message.thrown
          break;
      }
    }
  }
}
```

##### 测试方法

我们添加一个测试方法，来使用我们炫酷的 aspect 注解。我已经在主 Activity 类中增加了一个方法用来测试。看下代码：

```java
  @DebugTrace
  private void testAnnotatedMethod() {
    try {
      Thread.sleep(10);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
```

