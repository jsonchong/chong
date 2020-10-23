### **什么是注解处理器？**

注解处理器是（Annotation Processor）是javac的一个工具，用来在编译时扫描和编译和处理注解（Annotation）。

一个注解处理器它以Java代码或者（编译过的字节码）作为输入，生成文件（通常是java文件）。这些生成的java文件不能修改，并且会同其手动编写的java代码一样会被javac编译。

### **处理器AbstractProcessor**

处理器的写法有固定的套路，继承AbstractProcessor。如下：

```java
public class MyProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public Set getSupportedAnnotationTypes() {
        return null;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set annotations, RoundEnvironment roundEnv) {
        return true;
    }
}
```

- init(ProcessingEnvironment processingEnv) 被注解处理工具调用，参数ProcessingEnvironment 提供了Element，Filer，Messager等工具
- getSupportedAnnotationTypes() 指定注解处理器是注册给那一个注解的，它是一个字符串的集合，意味着可以支持多个类型的注解，并且字符串是合法全名。
- getSupportedSourceVersion 指定Java版本
- process(Set annotations, RoundEnvironment roundEnv) 这个也是最主要的，在这里扫描和处理你的注解并生成Java代码，信息都在参数RoundEnvironment 里了

### **注册注解处理器**

打包注解处理器的时候需要一个特殊的文件 javax.annotation.processing.Processor 在 META-INF/services 路径下

```
－－myprcessor.jar
－－－－com
－－－－－－example
－－－－－－－－MyProcessor.class
－－－－META-INF
－－－－－－services
－－－－－－－－javax.annotation.processing.Processor
```

打包进javax.annotation.processing.Processor的内容是处理器的合法全称，多个处理器之间换行。

```
com.example.myprocess.MyProcessorA
com.example.myprocess.MyProcessorB
```

google提供了一个注册处理器的库

```
compile 'com.google.auto.service:auto-service:1.0-rc2'
```

一个注解搞定：

```java
@AutoService(Processor.class)
public class MyProcessor extends AbstractProcessor {
      ...
}
```

BufferKnife使用：

```java
public class MainActivity extends AppCompatActivity {

    @Bind(R.id.rxjava_demo)
    Button mRxJavaDemo;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        mRxJavaDemo.setText("Text");
    }

}
```

### **项目结构**

```
－－apt-demo
－－－－bindview-annotation(Java Library)
－－－－bindview-api(Android Library)
－－－－bindview-compiler(Java Library)
－－－－app(Android App)
```

- bindview-annotation 注解声明
- bindview-api 调用Android SDK API
- bindview-compiler 注解处理器相关
- app 测试App

1. 在 bindview-annotation 下创建一个@BindView注解，该注解返回一个值，整型，名字为value，用来表示控件ID。

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface BindView {
    /**
     * 用来装id
     *
     * @return
     */
    int value();
}
```

2. 在 bindview-compiler 中创建注解处理器 BindViewProcessor 并注册，做基本的初始化工作。

```java
@AutoService(Processor.class)
public class BindViewProcessor extends AbstractProcessor {
    /**
     * 文件相关的辅助类
     */
    private Filer mFiler;
    /**
     * 元素相关的辅助类
     */
    private Elements mElementUtils;
    /**
     * 日志相关的辅助类
     */
    private Messager mMessager;
    /**
     * 解析的目标注解集合
     */
    private Map mAnnotatedClassMap = new HashMap<>();

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
        mFiler = processingEnv.getFiler();
    }

    @Override
    public Set getSupportedAnnotationTypes() {
        Set types = new LinkedHashSet<>();
        types.add(BindView.class.getCanonicalName());//返回该注解处理器支持的注解集合
        return types;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set annotations, RoundEnvironment roundEnv) {
        return true;
    }
}
```

是不是注意到了里面有个**Map容器**，而且类型是**AnnotatedClass**，我们在解析XML，解析Json的时候数据解析完之后要以对象的形式表示出来，这里也一样，@BindView用来标记类成员，一个类下可以有多个成员，好比一个Activity中可以有多个控件，一个容器下有多个控件等。如下

```java
/**
 * Created by mingwei on 12/10/16.
 * CSDN:    http://blog.csdn.net/u013045971
 * Github:  https://github.com/gumingwei
 */
public class AnnotatedClass {
    /**
     * 类名
     */
    public TypeElement mClassElement;
    /**
     * 成员变量集合
     */
    public List mFiled;
    /**
     * 元素辅助类
     */
    public Elements mElementUtils;

    public AnnotatedClass(TypeElement classElement, Elements elementUtils) {
        this.mClassElement = classElement;
        this.mElementUtils = elementUtils;
        this.mFiled = new ArrayList<>();
    }
    /**
     * 获取当前这个类的全名
     */
    public String getFullClassName() {
        return mClassElement.getQualifiedName().toString();
    }
    /**
     * 添加一个成员
     */
    public void addField(BindViewField field) {
        mFiled.add(field);
    }
    /**
     * 输出Java
     */
    public JavaFile generateFinder() {
        return null;
    }
    /**
     * 包名
     */
    public String getPackageName(TypeElement type) {
        return mElementUtils.getPackageOf(type).getQualifiedName().toString();
    }
    /**
     * 类名
     */
    private static String getClassName(TypeElement type, String packageName) {
        int packageLen = packageName.length() + 1;
        return type.getQualifiedName().toString().substring(packageLen).replace('.',''); 
    }
}
```

成员用BindViewField表示，没什么复杂的逻辑，在构造函数判断类型和初始化，简单的get函数

```java
/**
 * 被BindView注解标记的字段的模型类
 */
public class BindViewField {

    private VariableElement mFieldElement;

    private int mResId;

    public BindViewField(Element element) throws IllegalArgumentException {
        if (element.getKind() != ElementKind.FIELD) {//判断是否是类成员
            throw new IllegalArgumentException(String.format("Only field can be annotated with @%s", BindView.class.getSimpleName()));
        }
        mFieldElement = (VariableElement) element;
        //获取注解和值
        BindView bindView = mFieldElement.getAnnotation(BindView.class);
        mResId = bindView.value();
        if (mResId < 0) {
            throw new IllegalArgumentException(String.format("value() in %s for field % is not valid",
                    BindView.class.getSimpleName(), mFieldElement.getSimpleName()));
        }
    }

    public Name getFieldName() {
        return mFieldElement.getSimpleName();
    }

    public int getResId() {
        return mResId;
    }

    public TypeMirror getFieldType() {
        return mFieldElement.asType();
    }
}
```

接下来就是在处理器的process中解析注解了

每次解析前都要清空，因为process方法可能不止走一次。

拿到注解模型之后遍历调用生成Java代码

```java
@Override
    public boolean process(Set annotations, RoundEnvironment roundEnv) {
        mAnnotatedClassMap.clear();
        try {
            processBindView(roundEnv);
        } catch (IllegalArgumentException e) {
            error(e.getMessage());
            return true;
        }

        try {
            for (AnnotatedClass annotatedClass : mAnnotatedClassMap.values()) {
                info("generating file for %s", annotatedClass.getFullClassName());
                annotatedClass.generateFinder().writeTo(mFiler);
            }
        } catch (Exception e) {
            e.printStackTrace();
            error("Generate file failed,reason:%s", e.getMessage());
        }
        return true;
    }
```

processBindView 和 getAnnotatedClass

```java
/**
     * 遍历目标RoundEnviroment
     * @param roundEnv
     */
    private void processBindView(RoundEnvironment roundEnv) {
        for (Element element : roundEnv.getElementsAnnotatedWith(BindView.class)) {
            AnnotatedClass annotatedClass = getAnnotatedClass(element);
            BindViewField field = new BindViewField(element);
            annotatedClass.addField(field);
        }
    }
    /**
     * 如果在map中存在就直接用，不存在就new出来放在map里
     * @param element
     */
    private AnnotatedClass getAnnotatedClass(Element element) {
        TypeElement encloseElement = (TypeElement) element.getEnclosingElement();
        String fullClassName = encloseElement.getQualifiedName().toString();
        AnnotatedClass annotatedClass = mAnnotatedClassMap.get(fullClassName);
        if (annotatedClass == null) {
            annotatedClass = new AnnotatedClass(encloseElement, mElementUtils);
            mAnnotatedClassMap.put(fullClassName, annotatedClass);
        }
        return annotatedClass;
    
```

3. 在生成Java之前 我们要在bindview-api 中创建一些类，配合 bindview-compiler 一起使用。
4. 在AnnotatedClass中生成Java代码

生成代码使用了一个很好用的库 Javapoet 。类，方法，都可以使用构建器构建出来，很好上手，再也不用拼接字符串了

```java
public JavaFile generateFinder() {
        //构建方法
        MethodSpec.Builder injectMethodBuilder = MethodSpec.methodBuilder("inject")
                .addModifiers(Modifier.PUBLIC)//添加描述
                .addAnnotation(Override.class)//添加注解
                .addParameter(TypeName.get(mClassElement.asType()), "host", Modifier.FINAL)//添加参数
                .addParameter(TypeName.OBJECT, "source")//添加参数
                .addParameter(TypeUtil.FINDER, "finder");//添加参数

        for (BindViewField field : mFiled) {
            //添加一行
            injectMethodBuilder.addStatement("host.$N=($T)finder.findView(source,$L)", field.getFieldName()
                    , ClassName.get(field.getFieldType()), field.getResId());
        }

        String packageName = getPackageName(mClassElement);
        String className = getClassName(mClassElement, packageName);
        ClassName bindClassName = ClassName.get(packageName, className);
        //构建类
        TypeSpec finderClass = TypeSpec.classBuilder(bindClassName.simpleName() + "$Injector")//类名
                .addModifiers(Modifier.PUBLIC)//添加描述
                .addSuperinterface(ParameterizedTypeName.get(TypeUtil.INJECTOR, TypeName.get(mClassElement.asType())))//添加接口（类／接口，范型）
                .addMethod(injectMethodBuilder.build())//添加方法
                .build();

        return JavaFile.builder(packageName, finderClass).build();
    }

    public String getPackageName(TypeElement type) {
        return mElementUtils.getPackageOf(type).getQualifiedName().toString();
    }

    private static String getClassName(TypeElement type, String packageName) {
        int packageLen = packageName.length() + 1;
        return type.getQualifiedName().toString().substring(packageLen).replace('.',''); 
    }
```

可以在代码里System.out调试注解处理器的代码。

bindview-complier 引用 bindview-annotation

app 引用了剩下的三个module，在引用 bindview-complier 的时候用的apt的方式







