### Guava操作文件工具类

读写文件是一个很核心的编程任务。java 提供了一个丰富和健壮的库来进行 I/O 操作，但是执行一些基本操作非常繁琐。Guava 提供了一些简单的工具类。

###### 复制文件

Files 提供了一些有用的方法来操作 File 对象。对于 java 开发人员来说，从一个文件复制到另外一个文件是有挑战性的。但是，让我们考虑如何使用 Guava 来实现相同的功能

```java
File original = new File("path/to/original");
File copy = new File("path/to/copy");
Files.copy(original, copy);
```

###### 移动/重命名文件

使用 Guava 进行移动文件是非常简单的：

```java
public class GuavaMoveFileExample {
    public static void main(String[] args) {
        File original = new File("src/main/resources/copy.txt");
        File newFile = new File("src/main/resources/newFile.txt");
        try{
            Files.move(original, newFile);
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

###### 把文件当中字符串来操作

有时候我们需要操作文件里面的字符。Files 类有一个方法可以按行来读取文件并且把每行的信息存放到 list 当中，并且返回这个 list 。

```java
@Test
public void readFileIntoListOfStringsTest() throws Exception{
    File file = new File("src/main/resources/lines.txt");
    List<String> expectedLines = Lists.newArrayList("The quickbrown","fox jumps over","the lazy dog");
    List<String> readLines = Files.readLines(file,Charsets.UTF_8);
    assertThat(expectedLines,is(readLines));
}
```

还有一个重载的 readLines 方法，接受一个 LineProcessor 实例，读取的每一行都会调用 LineProcessor.processLine 并且把这行的数据传入进去，这个方法会返回一个布尔值。当文件读到末尾或者这个 LineProcessor.processLine 返回 false 的时候将停止读取文件。

看下面的例子:

```java
public class MyLineProcessor implements LineProcessor<List<String>>
{
    private List<String> result = Lists.newArrayList();

    @Override
    public boolean processLine(String line) throws IOException
    {
        result.add(line);
        return true;
    }

    @Override
    public List<String> getResult()
    {
        return result;
    }
}

@Test
public void readLinesTest2() throws Exception
{
    LineProcessor<List<String>> my = new MyLineProcessor();
    List<String> list = Files.readLines(new File("a.txt"), Charsets.UTF_8, my);
    System.out.println(my.getResult());
}
```



###### 哈希文件

可以使用下面的方法进行哈希文件：

```java
public class HashFileExample {
    public static void main(String[] args) throws IOException {
    File file = new File("src/main/resources/sampleTextFileOne.txt");
    HashCode hashCode = Files.hash(file, Hashing.md5());
    System.out.println(hashCode);
	}
}
```

###### 写文件

当我们操作输入/输出流的时候，我们需要一下几步：

1. 打开输入/输出流
2. 读取或者输出字节到流当中
3. 当做完操作后在 finally 块当中关闭流

我们不得不一次次的重复，很容易出错。Files 类提供便利的方法写文件或者从文件读取内容到字节数组中。

```java
@Test
public void appendingWritingToFileTest() throws IOException {
    File file = new File("src/test/resources/quote.txt");
    file.deleteOnExit();
    String hamletQuoteStart = "To be, or not to be";
    Files.write(hamletQuoteStart,file, Charsets.UTF_8);
    assertThat(Files.toString(file,Charsets.UTF_8),is(hamletQuoteStart));
    String hamletQuoteEnd = ",that is the question";
    Files.append(hamletQuoteEnd,file,Charsets.UTF_8);
    assertThat(Files.toString(file, Charsets.UTF_8),is(hamletQuoteStart + hamletQuoteEnd));
    String overwrite = "Overwriting the file";
    Files.write(overwrite, file, Charsets.UTF_8);
    assertThat(Files.toString(file, Charsets.UTF_8),is(overwrite));
}
```

使用 Files.write 方法往文件中写入数据，使用 Files.append 方法往文件中追加数据。

###### Closer

Closer 类是用于确保所有注册可关闭的对象在调用 Closer.close 之后都是关闭的。看下面的例子：

```java
public class CloserExample {
    public static void main(String[] args) throws IOException {
        Closer closer = Closer.create();
        try {
            File destination = new File("src/main/resources/copy.txt");
            destination.deleteOnExit();
            BufferedReader reader = new BufferedReader(new
            	FileReader("src/main/resources/sampleTextFileOne.txt"));
            BufferedWriter writer = new BufferedWriter(new FileWriter(destination));
            closer.register(reader);
            closer.register(writer);
            String line;
            while((line = reader.readLine())!=null){
            	writer.write(line);
            }
        } catch (Throwable t) {
        	throw closer.rethrow(t);
        } finally {
        	closer.close();
        }
    }
}
```

