### JVM类加载流程

java语言系统内置了众多类加载器，从一定程度上讲，只存在两种不同的类加载器：一种是启动类加载器，此类加载由C++实现，是JVM的一部分；另一种就是所有其他的类加载器，这些类加载器均由java实现，且全部继承自`java.lang.ClassLoader`

- Bootstrap ClassLoader 启动类加载器，最顶层的加载类，由C++实现，负责加载%JAVA_HOME%/lib目录中或-Xbootclasspath中参数指定的路径中的，并且是虚拟机识别的（按名称）类库

- Extention ClassLoader 扩展类加载器，由启动类加载器加载，实现为`sun.misc.Launcher$ExtClassLoader`，负责加载目录%JRE_HOME%/lib/ext目录中或-Djava.ext.dirs中参数指定的路径中的jar包和class文件

- Application ClassLoader 应用类加载器，也称为系统类加载器(System ClassLoader，可由`java.lang.ClassLoader.getSystemClassLoader()`获取)，实现为`sun.misc.Launcher$AppClassLoader`，由启动类加载器加载，负责加载当前应用classpath下的所有类

  

### 双亲委派模型

java语言系统有众多类加载器，包括用户自定义类加载器，各加载器之间的加载顺序如何？首先从JVM入口应用`sun.misc.Launcher`聊起

### Launcher

```java
public Launcher() {
    ExtClassLoader localExtClassLoader;
    try {
        // 加载扩展类加载器
        localExtClassLoader = ExtClassLoader.getExtClassLoader();
    } catch (IOException localIOException1) {
        throw new InternalError("Could not create extension class loader", localIOException1);
    }
    try {
        // 加载应用类加载器
        this.loader = AppClassLoader.getAppClassLoader(localExtClassLoader);
    } catch (IOException localIOException2) {
        throw new InternalError("Could not create application class loader", localIOException2);
    }
    // 设置AppClassLoader为线程上下文类加载器
    Thread.currentThread().setContextClassLoader(this.loader);
    // ...
    
    static class ExtClassLoader extends java.net.URLClassLoader
    static class AppClassLoader extends java.net.URLClassLoader
}
```

`ExtClassLoader`和`AppClassLoader`都继承自`URLClassLoader`，而最终的父类则为`ClassLoader`。

<img src="https://segmentfault.com/img/bV4ERs?w=651&amp;h=521" alt="classloader" style="zoom:50%;" />

### 父类加载器

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println("ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader());
        System.out.println("DNSNameService's ClassLoader is " + DNSNameService.class.getClassLoader());
        System.out.println("String's ClassLoader is " + String.class.getClassLoader());
    }
}

```

```
ClassLodarDemo's ClassLoader is sun.misc.Launcher$AppClassLoader@135fbaa4
DNSNameService's ClassLoader is sun.misc.Launcher$ExtClassLoader@6e0be858
String's ClassLoader is null
```

ClassLodarDemo为我们自己创建的类，其类加载器为AppClassLoader

DNSNameService为%JRE_HOME%/lib/ext目录下的类，其类加载器为ExtClassLoader

String存在于rt.jar中，但其类加载器为null，rt.jar由Bootstrap ClassLoader加载，而Bootstrap ClassLoader是由C++编写，属于JVM的一部分



### 双亲委派

这种模型除了顶层的BootStrap ClassLoader除外，其余的类加载器都应当有自己的父类加载器，如果一个类加载器收到了类加载请求，首先会把这个请求委派给父类加载器加载，只有父类加载器无法完成类加载请求时，子类加载器才会尝试自己去加载。

查看ClassLoader.loadClass方法

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 检查是否已经加载过
        Class<?> c = findLoadedClass(name);
        if (c == null) { // 没有被加载过
            long t0 = System.nanoTime();
            // 首先委派给父类加载器加载
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // 如果父类加载器无法加载，才尝试加载
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 自定义ClassLoader

不论是AppClassLoader还是ExtClassLoader还是启动类加载器，其加载类的路径都是固定的，如果我们需要加载外部类或者资源，如某路径下或网络上，这样便需要自定义类加载器
自定义类加载器，只需要继承ClassLoader类，复写findClass方法，在findClass方法中调用defineClass方法即可

> 一个ClassLoader创建时如果没有指定parent，那么它的parent默认就是AppClassLoader
> 如果需要制定一个ClassLoader的父类加载器为启动类加载器，只需要将其parent指定为null即可

首先，编写一个测试用的类文件

```java
public class BeLoadedClass {
    public void say() {
        System.out.println("I'm Loaded by " + this.getClass().getClassLoader());
    }
}
```

将其编译，放入/data/classloader目录下,接下来，编写DiskClassLoader

```java
public class DiskClassLoader extends URLClassLoader {

    public DiskClassLoader(URL path) throws MalformedURLException {
        super(new URL[]{path});
    }

    public DiskClassLoader(URL path, ClassLoader parent) throws MalformedURLException {
        super(new URL[]{path}, parent);
    }
}
```

```java
protected Class<?> findClass(final String name) throws ClassNotFoundException {
    final Class<?> result;
    try {
        result = AccessController.doPrivileged(
            new PrivilegedExceptionAction<Class<?>>() {
                public Class<?> run() throws ClassNotFoundException {
                    // 类文件全路径
                    String path = name.replace('.', '/').concat(".class");
                    // 指定资源目录下查找
                    Resource res = ucp.getResource(path, false);
                    if (res != null) {
                        try {
                            // 调用defineClass生成类
                            return defineClass(name, res);
                        } catch (IOException e) {
                            throw new ClassNotFoundException(name, e);
                        }
                    } else {
                        return null;
                    }
                }
            }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (ClassNotFoundException) pae.getException();
    }
    if (result == null) {
        throw new ClassNotFoundException(name);
    }
    return result;
}
```

编写测试程序

```java
public class ClassLoaderDemo {
    public static void main(String[] args) throws Exception {
        URL path = new File("/data/classloader").toURI().toURL();

        DiskClassLoader diskClassLoaderA = new DiskClassLoader(path);
        Class<?> clazzA = diskClassLoaderA.loadClass("BeLoadedClass");
        Method sayA = clazzA.getMethod("say");
        Object instanceA = clazzA.newInstance();
        sayA.invoke(instanceA);
        System.out.println(diskClassLoaderA);
        System.out.println("clazzA@" + clazzA.hashCode());

        System.out.println("====");

        DiskClassLoader diskClassLoaderB = new DiskClassLoader(path, diskClassLoaderA);
        Class<?> clazzB = diskClassLoaderB.loadClass("BeLoadedClass");
        Method sayB = clazzB.getMethod("say");
        Object instanceB = clazzA.newInstance();
        sayB.invoke(instanceB);
        System.out.println(diskClassLoaderB);
        System.out.println("clazzB@" + clazzB.hashCode());

        System.out.println("====");

        DiskClassLoader diskClassLoaderC = new DiskClassLoader(path);
        Class<?> clazzC = diskClassLoaderC.loadClass("BeLoadedClass");
        Method sayC = clazzC.getMethod("say");
        Object instanceC = clazzC.newInstance();
        sayC.invoke(instanceC);
        System.out.println(diskClassLoaderC);
        System.out.println("clazzC@" + clazzC.hashCode());

        System.out.println("====");

        System.out.println("clazzA == clazzB " + (clazzA == clazzB));
        System.out.println("clazzC == clazzB " + (clazzC == clazzB));
    }
}
```

输出为：

```
I'm Loaded by com.manerfan.jvm.oom.DiskClassLoader@4b67cf4d
com.manerfan.jvm.DiskClassLoader@4b67cf4d
clazzA@312714112
====
I'm Loaded by com.manerfan.jvm.oom.DiskClassLoader@4b67cf4d
com.manerfan.jvm.DiskClassLoader@29453f44
clazzB@312714112
====
I'm Loaded by com.manerfan.jvm.oom.DiskClassLoader@5cad8086
com.manerfan.jvm.DiskClassLoader@5cad8086
clazzC@1639705018
====
clazzA == clazzB true
clazzC == clazzB false
```

### 类的唯一性

对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间

