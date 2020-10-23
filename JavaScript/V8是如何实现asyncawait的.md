### 什么是回调地狱？

代码逻辑的不连贯、不线性，非常不符合人的直觉。

因此，异步回调模式影响到我们的编码方式，如果在代码中过多地使用异步回调函数，会将你的整个代码逻辑打乱，从而让代码变得难以理解，这也就是我们经常所说的回调地狱问题。

### 使用 Promise 解决回调地狱问题

比如最新的 fetch 就使用 Promise 的技术

```javascript

fetch(id_url)

.then((response) => {

return response.text()

})

.then((response) => {

let new_name_url = name_url + "?id=" + response

return fetch(new_name_url)

}).then((response) => {

return response.text()

}).then((response) => {

console.log(response)//输出最终的结果

})  
```

### 使用 Generator 函数实现更加线性化逻辑

虽然使用 Promise 可以解决回调地狱中编码不线性的问题，但这种方式充满了 Promise 的 then() 方法，如果处理流程比较复杂的话，那么整段代码将充斥着大量的 then，异步逻辑之间依然被 then 方法打断了，因此这种方式的语义化不明显，代码不能很好地表示执行流程。

能不能更进一步，像编写同步代码的方式来编写异步代码，比如：

```javascript

 function getResult(){
   let id = getUserID()
   let name = getUserName(id)
   return name
 }
```

由于 getUserID() 和 getUserName() 都是异步请求，如果要实现这种线性的编码方式，**那么一个可行的方案就是执行到异步请求的时候，暂停当前函数，等异步请求返回了结果，再恢复该函数。**

具体地讲，执行到 getUserID() 时暂停 GetResult 函数，然后浏览器在后台处理实际的请求过程，待 ID 数据返回时，再来恢复 GetResult 函数。接下来再执行 getUserName 来获取到用户名，由于 getUserName() 也是一个异步请求，所以在使用 getUserName() 的同时，依然需要暂停 GetResult 函数的执行，等到 getUserName() 返回了用户名数据，再恢复 GetResult 函数的执行，最终 getUserName() 函数返回了 name 信息。

<img src="https://static001.geekbang.org/resource/image/48/65/485d9de2097107f818c622a83a953665.jpg" alt="img" style="zoom:50%;" />

这个模型的关键就是实现函数暂停执行和函数恢复执行，而生成器就是为了实现暂停函数和恢复函数而设计的。

**生成器函数是一个带星号函数，配合 yield 就可以实现函数的暂停和恢复**，我们看看生成器的具体使用方式：

```javascript

function* getResult() {

yield 'getUserID'

yield 'getUserName'

return 'name'

}

let result = getResult()

console.log(result.next().value)

console.log(result.next().value)
console.log(result.next().value)
```

执行上面这段代码，观察输出结果，你会发现函数 getResult 并不是一次执行完的，而是全局代码和 getResult 函数交替执行

其实这就是生成器函数的特性，在生成器内部，**如果遇到 yield 关键字，那么 V8 将返回关键字后面的内容给外部，并暂停该生成器函数的执行。生成器暂停执行后，外部的代码便开始执行，外部代码如果想要恢复生成器的执行，可以使用 result.next 方法。**

<img src="https://static001.geekbang.org/resource/image/f0/41/f0e0800ab9ec2f559fbb58ecabc76c41.jpg" alt="img" style="zoom:33%;" />

因为生成器可以暂停函数的执行，所以，我们将所有异步调用的方式，写成同步调用的方式，比如我们使用生成器来实现上面的需求，代码如下所示：

```javascript

function* getResult() {
    let id_res = yield fetch(id_url);
    console.log(id_res)
    let id_text = yield id_res.text();
    console.log(id_text)


    let new_name_url = name_url + "?id=" + id_text
    console.log(new_name_url)


    let name_res = yield fetch(new_name_url)
    console.log(name_res)
    let name_text = yield name_res.text()
    console.log(name_text)
}


let result = getResult()
result.next().value.then((response) => {
    return result.next(response).value
}).then((response) => {
    return result.next(response).value
}).then((response) => {
    return result.next(response).value
}).then((response) => {
    return result.next(response).value
```



们可以将同步、异步逻辑全部写进生成器函数 getResult 的内部，然后，我们在外面依次使用一段代码来控制生成器的暂停和恢复执行。以上，就是协程和 Promise 相互配合执行的大致流程。

通常，我们把执行生成器的代码封装成一个函数，这个函数驱动了 getResult 函数继续往下执行，**我们把这个执行生成器代码的函数称为执行器（可参考著名的 co 框架）**，如下面这种方式：

```javascript

function* getResult() {
    let id_res = yield fetch(id_url);
    console.log(id_res)
    let id_text = yield id_res.text();
    console.log(id_text)


    let new_name_url = name_url + "?id=" + id_text
    console.log(new_name_url)


    let name_res = yield fetch(new_name_url)
    console.log(name_res)
    let name_text = yield name_res.text()
    console.log(name_text)
}
co(getResult())
```

### async/await：异步编程的“终极”方案

由于生成器函数可以暂停，因此我们可以在生成器内部编写完整的异步逻辑代码，不过生成器依然需要使用额外的 co 函数来驱动生成器函数的执行，这一点非常不友好。

基于这个原因，**ES7 引入了 async/await**，这是 JavaScript 异步编程的一个重大改进，它改进了生成器的缺点，提供了在不阻塞主线程的情况下使用同步代码实现异步访问资源的能力。你可以参考下面这段使用 async/await 改造后的代码：

```javascript

async function getResult() {
    try {
        let id_res = await fetch(id_url)
        let id_text = await id_res.text()
        console.log(id_text)
  
        let new_name_url = name_url+"?id="+id_text
        console.log(new_name_url)


        let name_res = await fetch(new_name_url)
        let name_text = await name_res.text()
        console.log(name_text)
    } catch (err) {
        console.error(err)
    }
}
getResult()
```

也就是说，在执行到 await fetch 的时候，整个函数会暂停等待 fetch 的执行结果，等到函数返回时，再恢复该函数，然后继续往下执行。

其实 async/await 技术背后的秘密就是 Promise 和生成器应用，往底层说，就是微任务和协程应用。要搞清楚 async 和 await 的工作原理，我们就得对 async 和 await 分开分析。

我们先来看看 async 到底是什么。根据 MDN 定义，**async 是一个通过异步执行并隐式返回 Promise 作为结果的函数**。

这里需要重点关注异步执行这个词，简单地理解，如果在 async 函数里面使用了 await，那么此时 async 函数就会暂停执行，并等待合适的时机来恢复执行，所以说 async 是一个异步执行的函数。

那么暂停之后，什么时机恢复 async 函数的执行呢？

要解释这个问题，我们先来看看，V8 是如何处理 await 后面的内容的。

通常，await 可以等待两种类型的表达式：

1. 可以是任何普通表达式 ;
2. 也可以是一个 Promise 对象的表达式。

**如果 await 等待的是一个 Promise 对象，它就会暂停执行生成器函数，直到 Promise 对象的状态变成 resolve，才会恢复执行**，然后得到 resolve 的值，作为 await 表达式的运算结果。

```javascript

function NeverResolvePromise(){
    return new Promise((resolve, reject) => {})
}
async function getResult() {
    let a = await NeverResolvePromise()
    console.log(a)
}
getResult()
console.log(0)
```

这一段代码，我们使用 await 等待一个没有 resolve 的 Promise，那么这也就意味着，getResult 函数会一直等待下去。

和生成器函数一样，使用了 async 声明的函数在执行时，也是一个单独的协程，我们可以使用 await 来暂停该协程，由于 await 等待的是一个 Promise 对象，我们可以 resolve 来恢复该协程。

<img src="https://static001.geekbang.org/resource/image/5f/f6/5fd7a95e6dd2ee6dda588641e2eecaf6.jpg" alt="img" style="zoom:33%;" />

如果 await 等待的对象已经变成了 resolve 状态，那么 V8 就会恢复该协程的执行，我们可以修改下上面的代码，来证明下这个过程：

```javascript

function HaveResolvePromise(){
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(100)
          }, 0);
      })
}
async function getResult() {
    console.log(1)
    let a = await HaveResolvePromise()
    console.log(a)
    console.log(2)
}
console.log(0)
getResult()
console.log(3)
```

<img src="https://static001.geekbang.org/resource/image/1c/7f/1c7bc077282dfa996746e5c403c42f7f.jpg" alt="img" style="zoom:33%;" />

如果 await 等待的是一个非 Promise 对象，比如 await 100，那么通用 V8 会**隐式地**将 await 后面的 100 包装成一个已经 resolve 的对象，其效果等价于下面这段代码：

```javascript

function ResolvePromise(){
    return new Promise((resolve, reject) => {
            resolve(100)
      })
}
async function getResult() {
    let a = await ResolvePromise()
    console.log(a)
}
getResult()
console.log(3)
```



