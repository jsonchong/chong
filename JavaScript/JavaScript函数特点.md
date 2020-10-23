### 函数的本质

在 JavaScript 中，函数是一种特殊的对象，它和对象一样可以拥有属性和值，但是函数和普通对象不同的是，函数可以被调用。

先来看一段 JavaScript 代码，在这段代码中，我们定义了一个函数 foo，接下来我们给 foo 函数设置了 myName 和 uName 的属性

```javascript
function foo(){
    var test = 1
}
foo.myName = 1
foo.uName = 2
console.log(foo.myName)
```

既然是函数，那么它也可以被调用。比如你定义了一个函数，便可以通过函数名称加小括号来实现函数的调用，代码如下所示：

```javascript
function foo(){
    var test = 1
    console.log(test)
}
foo()
```

除了使用函数名称来实现函数的调用，还可以直接调用一个匿名函数，代码如下所示：

```javascript
(function (){
    var test = 1
    console.log(test)
})()
```

其实在 V8 内部，会为函数对象添加了两个隐藏属性，具体属性如下图所示：

<img src="https://static001.geekbang.org/resource/image/9e/e2/9e274227d637ce8abc4a098587613de2.jpg" alt="img" style="zoom:50%;" />

也就是说，函数除了可以拥有常用类型的属性值之外，还拥有两个隐藏属性，分别是 name 属性和 code 属性。

隐藏 name 属性的值就是函数名称，如果某个函数没有设置函数名，如下面这段函数：

```javascript
(function (){
    var test = 1
    console.log(test)
})()
```

该函数对象的默认的 name 属性值就是 anonymous，表示该函数对象没有被设置名称。另外一个隐藏属性是 code 属性，其值表示函数代码，以字符串的形式存储在内存中。当执行到一个函数调用语句时，V8 便会从函数对象中取出 code 属性值，也就是函数代码，然后再解释执行这段函数代码。

### 函数是一等公民

**如果某个编程语言的函数，可以和这个语言的数据类型做一样的事情，我们就把这个语言中的函数称为一等公民**

我们知道，在执行 JavaScript 函数的过程中，为了实现变量的查找，V8 会为其维护一个作用域链，如果函数中使用了某个变量，但是在函数内部又没有定义该变量，那么函数就会沿着作用域链去外部的作用域中查找该变量

```javascript
function foo(){
    var number = 1
    function bar(){
        number++
        console.log(number)
    }
    return bar
}
var mybar = foo()
mybar()
```

观察上段代码可以看到，我们在 foo 函数中定义了一个新的 bar 函数，并且 bar 函数引用了 foo 函数中的变量 number，当调用 foo 函数的时候，它会返回 bar 函数。那么所谓的“函数是一等公民”就体现在，如果要返回函数 bar 给外部，那么即便 foo 函数执行结束了，其内部定义的 number 变量也不能被销毁，因为 bar 函数依然引用了该变量。









