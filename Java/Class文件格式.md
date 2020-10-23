Class文件是Java源代码文件经Java编译器编译后得到的Java字节码文件。对比Linux、Windows上的可执行文件而言，Class文件可以看作是Java虚拟机的可执行文件

### Class文件格式总览

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200818110730118.png" alt="image-20200818110730118" style="zoom:50%;" />

- magic: 4个字节长，取值必须是0xCAFEBABE
- minor_version: 2个字节长，表示该class文件版本的小版本信息
- major_verion: 2个字节长，表示该class文件版本的大版本信息
- ·constant_pool_count: 表示常量池数组中元素的个数
- constant_pool: 存储cp_info信息的数组
- ·access_flags: 标明该类的访问权限，比如public、private之类的信息。
- ·this_class和super_class：存储的是指向常量池数组元素的索引。通过这两个索引和常量池对应元素的内容，我们可以知道本类和父类的类名（只是类名，不包含包名。类名最终用字符串描述）
- ·interfaces_count和interfaces: 这两个成员表示该类实现了多少个接口以及接口类的类名。和this_class一样，这两个成员也只是常量池数组里的索引号。真正的信息需要通过解析常量池的内容才能得到。
- ·fields_count和fields: 该类包含了成员变量的数量和它们的信息。成员变量信息由field_info结构体表示。
- ·methods_count和methods：该类包含了成员函数的数量和它们的信息。成员函数信息由method_info结构体表示。

### 常量池及相关内容

Java虚拟机规范中，常量池的英文叫Constant Pool，对应的**数据结构伪代码就是一个类型为cp_info的数组**。每一个cp_info对象存储了一个常量项。cp_info对应数据结构的伪代码如下所示。

```
cp_info {//u1表示该域对应一个字节长度，u表示unsigned
    u1 tag;//每一个cp_info的第一个字节表明该常量项的类型
    u1 info[];//常量项的具体内容
}
```

每一个常量项的第一个字节用于表明常量项的类型，紧接其后的才是具体的常量项内容了。常量项有哪些类型呢

​																		常量项的类型和tag取值

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200825175219769.png" alt="image-20200825175219769" style="zoom:50%;" />

**CONSTANT_Utf8：**该常量项真正存储了字符串的内容。以后我们将看到此类型常量项对应的数据结构中有一个字节数组，字符串就存储在这个字节数组中。
**CONSTANT_String：**代表了一个字符串，但是它本身不包含字符串的内容，而仅仅包含一个指向类型为CONSTANT_Utf8常量项的索引。

和String_info一样，Class_info里的name_index、MethodType_info里的descriptor_index、NameAndType_info里的name_index和descriptor_index都代表一个指向类型为Utf8_info元素的索引。

Double_info、Float_info、Integer_info和Long_info结构体内直接就能存储数据，这几个info之间没有引用关系。

**采用这种间接引用元素索引的方式就是为了节省Class文件的空间**

我们来看看几种常见常量项的内容

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200825181957422.png" alt="image-20200825181957422" style="zoom:50%;" />

### 数据类型描述规则

它讲述的是数据类型如何用对应的字符串来描述：

**原始数据类型对应的字符串描述为"B""C""D""F""I""J""S""Z"，它们分别对应的Java类型为byte、char、double、float、int、long、short、boolean。**

**·引用数据类型的格式为"LClassName；**"。此处的ClassName为对应类的全路径名，比如上例中的"Ljava/lang/String；"。全路径名的“.”号由“/”替代，并且最后必须带分号。

·数组也是一种引用类型，数组用"[其他类型的描述名"来表示，比如一个int数组的描述为"[I"，一个字符串数组的描述为"[Ljava/lang/String；"，一个二维int数组的描述为"[[I […]

### field_info和method_info

来看看Class文件格式里的field_info和method_info

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826175414584.png" alt="image-20200826175414584" style="zoom:50%;" />

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826180348598.png" alt="image-20200826180348598" style="zoom:50%;" />

​								filed_info和method_info数据结构伪代码

field_info和method_info对应的数据结构，二者有完全一样的成员变量。

·access_flags为访问标志

·name_index为指向成员变量或成员函数名字的Utf8_info常量项

·descriptor_index也指向Utf8_info常量项，其内容分别是描述成员变量的FieldDescriptor和描述成员函数的MethodDescriptor。

·attributes为属性信息，成员域和成员函数都包含若干属性。

### 属性介绍

```
attribute_info {
    u2 attribute_name_index;   // 属性名称，指向常量池中Utf8常量项的索引
    u4 attribute_length;       // 该属性具体内容的长度，即下面info数组的长度
    u1 info[attribute_length]; // 属性具体内容
}
```

属性是由其名称来区别的，即attribute_info中的attribute_name_index所指向的Utf8字符串

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826182529314.png" alt="image-20200826182529314" style="zoom:50%;" />

"Code"属性只能出现在method_info中。因为虚拟机在解析Class文件的时候是需要校验很多内容的，比如abstract的函数或native的函数就不能携带"Code"属性

### Code属性

一个函数的内容（也就是这个函数的源码转换后得到的Java字节码）就存储在Code属性中

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826182629160.png" alt="image-20200826182629160" style="zoom:50%;" />

- ·attribute_name_index指向内容为"Code"的Utf8_info常量项。attribute_length表示接下来内容的长度。

- ·max_stack：**JVM执行一个指令的时候，该指令的操作数存储在一个名叫“操作数栈（operand stack）**”的地方，每一个操作数占用一个或两个（long、double类型的操作数）栈项。stack就是一块只能进行先入后出的内存。**max_stack用于说明这个函数在执行过程中，需要最深多少栈空间（也就是多少栈项）。max_locals表示该函数包括最多几个局部变量。**注意，max_stack和max_locals都和JVM如何执行一个函数有关。**根据JVM官方规范，每一个函数执行的时候都会分配一个操作数栈和局部变量数组**。所以Code_attribute需要包含这些内容，这样JVM在执行函数前就可以分配相应的空间。

- code_length和code：**函数对应的指令内容也就是这个函数的源码经过编译器转换后得到的Java指令码存储在code数组中**，其长度由code_length表明。

- exception_table_length和exception_table：一个函数可以包含多个try/catch语句，一个try/catch语句对应exception_table数组中的一项。

  

介绍exception_table之前需要先了解pc（program counter）的概念。**JVM执行的时候，会维护一个变量来指向当前要执行的指令，这个变量就叫pc**。有了pc的概念，exception_table中各成员变量的含义就比较容易理解了。其中：

start_pc描述try/cath语句从哪条指令开始。注意，这个table中的各个pc变量的取值必须位于代表整个函数内容的Java字节码code[code_length]数组中。

end_pc表示这个try语句到哪条指令结束。注意，只包括try语句，不包括catch。

handler_pc表示catch语句的内容从哪条指令开始。

catch_type表示catch中截获的Exception或Error的名字，指向Utf8_info常量项。如果catch_type取值为0，则表示它是final{}语句块

### LineNumberTable属性

用于Java的调试，可指明某条指令对应于源码哪一行

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826191303536.png" alt="image-20200826191303536" style="zoom:50%;" />

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200826191314506.png" alt="image-20200826191314506" style="zoom:50%;" />

### Java指令码介绍

源码中一个类的每一个成员变量和成员函数都会相应在Class文件里存在一个field_info和method_info。JVM解析这个Class文件后无非是在内存里多创建了一个field_info和method_info对象。不过有了这些信息，JVM好像也无法做什么动作，因为这些信息并不能驱使JVM执行我们在源码中编写的函数。

我们知道Code_attribute中的code数组存储了一个函数源码经过编译后得到的Java字节码。根据Java虚拟机规范，code数组只能包括下面两种类型的信息。

Java指令码，即指示JVM该做什么动作，比如是进行加操作还是减操作，或者是new一个对象。在JVM规范中，**指令码的长度是1个字节。所以JVM规范中定义的Java指令码的个数不会超过255个（255的16进制表示为0xFF）。**

紧接指令码之后的是0个或多个操作数。JVM在执行某条Java指令码的时候，往往还需要其他一些参数。参数可以直接存储在code数组里，也可以存储在操作栈（Operand stack）中。由于不同指令码可能需要不同个数的参数，所以指令码后面的内容可以是参数（如果这条指令码需要参数）也可以是下一条指令（如果这条指令码无需参数）

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200827113941167.png" alt="image-20200827113941167" style="zoom:50%;" />

对invokevirtual来说，该指令后面需要两个字节的参数。比如图中的**indexbyte1<<8|indexbyte2共同组成一个指向常量池的索引，该索引对应的常量项是Methodref。Methodref描述了这个函数的信息（属于哪个类，以及该函数的Method Descriptor等），解析这个信息可得到函数调用时需要多少个参数。如此，JVM才能决定右边的操作数栈中有多少项（参数的个数由Methodref决定）需要在这次invokevirtual时用到。**

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200827121040975.png" alt="image-20200827121040975" style="zoom:50%;" />

**右边是操作数栈的变化情况。规范中，“操作数栈”这一部分描述的就是该指令执行前后操作数栈的变化情况。大多数指令执行的时候，JVM需要预先准备好操作数栈，里边存储了该指令需要的信息**

"objectref，[arg1，[arg2...]]→"中的箭头符号表示栈顶的增长方向，也就是栈顶在最右边，栈底在最左边

指令码的参数来源有两种：

1. 紧跟其后的内容可以是参数。
2. 操作数栈里的内容也是参数。

