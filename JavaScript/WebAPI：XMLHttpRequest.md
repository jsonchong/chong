深入讲解 XMLHttpRequest 之前，我们得先介绍下同步回调和异步回调这两个概念，这会帮助你更加深刻地理解 WebAPI 是怎么工作的

### 回调函数 VS 系统调用栈

```javascript

let callback = function(){
    console.log('i am do homework')
}
function doWork(cb) {
    console.log('start do work')
    cb()
    console.log('end do work')
}
doWork(callback)
```

上面的回调方法有个特点，就是回调函数 callback 是在主函数 doWork 返回之前执行的，我们把这个回调过程称为**同步回调。**

```javascript

let callback = function(){
    console.log('i am do homework')
}
function doWork(cb) {
    console.log('start do work')
    setTimeout(cb,1000)   
    console.log('end do work')
}
doWork(callback)
```









