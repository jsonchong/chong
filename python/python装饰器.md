### 简单的装饰器

```python

def my_decorator(func):
    def wrapper():
        print('wrapper of decorator')
        func()
    return wrapper

def greet():
    print('hello world')

greet = my_decorator(greet)
greet()

# 输出
wrapper of decorator
hello world
```

这里的函数 my_decorator() 就是一个装饰器，它把真正需要执行的函数 greet() 包裹在其中，并且改变了它的行为，但是原函数 greet() 不变。

事实上，上述代码在 Python 中有更简单、更优雅的表示

```python

def my_decorator(func):
    def wrapper():
        print('wrapper of decorator')
        func()
    return wrapper

@my_decorator
def greet():
    print('hello world')

greet()
```

### 带有参数的装饰器

一个简单的办法，是可以在对应的装饰器函数 wrapper() 上，加上相应的参数

```python

def my_decorator(func):
    def wrapper(message):
        print('wrapper of decorator')
        func(message)
    return wrapper


@my_decorator
def greet(message):
    print(message)


greet('hello world')

# 输出
wrapper of decorator
hello world
```

### 类装饰器

类装饰器主要依赖于函数`__call__()`，每当你调用一个类的示例时，函数`__call__()`就会被执行一次

```python

class Count:
    def __init__(self, func):
        self.func = func
        self.num_calls = 0

    def __call__(self, *args, **kwargs):
        self.num_calls += 1
        print('num of calls is: {}'.format(self.num_calls))
        return self.func(*args, **kwargs)

@Count
def example():
    print("hello world")

example()

# 输出
num of calls is: 1
hello world

example()

# 输出
num of calls is: 2
hello world

...
```

我们定义了类 Count，初始化时传入原函数 func()

### 装饰器用法实例

### 身份认证

```python

import functools

def authenticate(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        request = args[0]
        if check_user_logged_in(request): # 如果用户处于登录状态
            return func(*args, **kwargs) # 执行函数post_comment() 
        else:
            raise Exception('Authentication failed')
    return wrapper
    
@authenticate
def post_comment(request, ...)
    ...
 
```

我们定义了装饰器 authenticate；而函数 post_comment()，则表示发表用户对某篇文章的评论。每次调用这个函数前，都会先检查用户是否处于登录状态，如果是登录状态，则允许这项操作；如果没有登录，则不允许。

### 日志记录

日志记录同样是很常见的一个案例。在实际工作中，如果你怀疑某些函数的耗时过长，导致整个系统的 latency（延迟）增加，所以想在线上测试某些函数的执行时间，那么，装饰器就是一种很常用的手段。

```python

import time
import functools

def log_execution_time(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        res = func(*args, **kwargs)
        end = time.perf_counter()
        print('{} took {} ms'.format(func.__name__, (end - start) * 1000))
        return res
    return wrapper
    
@log_execution_time
def calculate_similarity(items):
    ...
```

### 缓存

LRU cache，在 Python 中的表示形式是@lru_cache。@lru_cache会缓存进程中的函数参数和结果，当缓存满了以后，会删除 least recenly used 的数据。

这样一来，我们通常使用缓存装饰器，来包裹这些检查函数，避免其被反复调用，进而提高程序运行效率，比如写成下面这样：

```python
@lru_cache
def check(param1, param2, ...) # 检查用户设备类型，版本号等等
    ...
```

















