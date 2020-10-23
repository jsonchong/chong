### 基本性质

对象(Object Instance)并没有原型，而构造器(Constructor)有原型，对象只有"构造自某个原型"的问题，并不存在"持有某个原型"的问题

原型其实也是一个对象实例，如果构造器有一个原型对象A,则由该构造器创建的实例(Instance)都必然复制自A

由于实例复制自对象A，所以必然继承了A的所有属性，方法和其他性质

**原型也是对象实例** 是一个关键的性质，这与**类继承体系**在本质上有不同，对于类继承来说，类不必是对象，如类可以是一个内存块，也可以是一段描述文本，而不必是一个有对象特性(例如可以调用方法或存取属性)的结构

### 空对象是所有对象的基础

如下两行代码都是自Object.prototype上复制出一个对象的映像来

```javascript
obj1 = new Object();

obj2 = {};
```

因此对象的构建过程可以被简单地映射为复制

### 复制策略

如果都从原型中复制出一个实例，新的实例与原型占用了相同的内存空间，就是非常的不经济，内存空间的消耗会急剧增加

- **写时复制**：在读取的时候顺着读原型即可，当需要写对象(obj2)的属性的时候，我们就复制一个原型的映像出来，并使以后的操作指向该映像就行了，不过对于经常进行写操作来说，这种法子并不比上一种经济
- **写时复制粒度变细**:把写复制的粒度从原型变成了成员，仅当写某个实例的成员时，将成员的信息复制到实例映像中，这样一来在初始构造该对象时仍与之前写时复制的原型继承一样，但写对象属性(如：obj2.value =10)时，会产生一个名为value的属性值，放在obj2对象的成员列表中

存取实例中的属性比存取原型中的属性效率要高，obj2.value比obj1.value要少一个指针访问

### 构造过程：从函数到构造器

上面的话题只讲述了对象的形成过程，而没有说明对象的构造过程，那函数作为一个构造器时，都做了些什么

其实构造函数首先是函数，尽管有一个prototype成员，但如果每声明一个函数都先创建一个对象实例，并使prototype成员指向它，那么也并不经济，所以可以认为prototype在函数初始时根本是无值的，实现上可能是如下逻辑：

```javascript
var __proto__ = null;

function get_prototype(){
    if(!__proto__){
        __proto__ = new Object();
        __proto__.constructor = this;
    }
    return __proto__;
}
```

所以，函数只有在需要引用到原型时，才具有构造器的性质，而且函数的原型总是一个系统内置的Object()构造器的实例，不过该实例创建后constructor属性总先被赋值为当前函数

```javascript
function MyObject(){
    
}

alert(MyObject.prototype.constructor == MyObject);
//显示 true 表明原型的构造器总是指向函数自身 同时也说明了MyObject构造
//的实例对象的constructor属性也指向MyObject
```

当一个函数的prototype有意义之后，它就摇身一变成了一个"构造器"，如果试图用new运算符创建一个它的实例，那么引擎就再构造一个对象，并使该对象的原型链接指向这个prototype属性就可以了，因此函数与构造器没有明显的界限，区别在于原型prototype是不是一个有意义的值

### constructor属性的维护

一个构造器产生的实例，其constructor属性默认总是指向该构造器

```javascript
function MyObject(){

}

var obj = new MyObject();
alert(obj.constructor == MyObject);
```

究其根源在于构造器函数的原型的constructor属性指向了构造器本身，所以

```javascript
alert(MyObject.prototype.constructor == MyObject)
```

所以一般情况下可以通过实例constructor属性来找到构造器，进而找到原型

```javascript
alert(obj.constructor.prototype)
```

下面的代码导致了问题

```javascript
function MyObject(){

}

function MyObjectEx(){

}
//构建原型链
MyObjectEx.prototype = new MyObject()

//等效于 proto = new MyObject()
// MyObjectEx.prototype = proto
```

因为我们重置了MyObjectEx的原型属性，使其指向了MyObject的实例，这样的话，如果由MyObjectEx构造的实例的constructor属性就需要从MyObjectEx新的prototype的constructor属性找，而prototype的constructor属性即为MyObject实例的constructor的属性，所以导致

```javascript
var obj1 = new MyObject();

var obj2 = new MyObjectEx();

//结果为true 都是MyObject
alert(obj1.constructor == obj2.constructor);
```

问题在于我们给MyObjectEx的赋予一个原型时，应该"修正"该原型的构造器值，这个修正的建议出自《JavaScript权威指南》中的处理：

```javascript
//方法一
MyObjectEx.prototype = new MyObject();
MyObjectEx.prototype.constructor = MyObjectEx;
```

也就是说重置原型后，再**修改原型的constructor属性，使之指向正确的构造器函数**，这样MyObject和MyObjectEx构造的实例就能指向对应正确的构造器函数了，但是有个问题是由于**丢掉了原型的constructor属性**，就**切断了与原型父类的关系，不能通过constructor追溯正确的原型链了**

另一个思路是保持原型的构造器属性，在子类构造器函数内初始化实例的constructor属性

```javascript
//方法2

function MyObjectEx(){
    this.constructor = MyObjectEx;
}

MyObjectEx.prototype = new MyObject();
```

这样一来，MyObjectEx()构造器的实例的constructor属性都正确地指向MyObjectEx().而原型的constructor则指向MyObject







