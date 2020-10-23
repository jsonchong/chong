### 什么是Goroutine

goroutine与线程相比创建成本非常小，可以认为goroutine就是一小段代码，我们使用goroutine往往是执行某一个特定的任务，也就是函数或者方法。

使用go关键字调用这个函数开启一个goroutine时候，即使这个函数有返回值也会忽略的。所以不需要接收这个函数的返回值。

### Goroutine是如何执行的

在函数或者方法前面加上关键字go，就会同时运行一个新的goroutine。

**与函数不同的是goroutine调用之后会立即返回，不会等待goroutine的执行结果，所以goroutine不会接收返回值**。 把封装main函数的goroutine叫做主goroutine，main函数作为主goroutine执行，如果main函数中goroutine终止了，程序也将终止，其他的goroutine都不会再执行。

使用匿名函数创建goroutine时候在匿名函数后加上(),直接调用

```go
package main

import (
	"fmt"
)

func main() {
	go func() {
		fmt.Println("匿名函数创建goroutine执行")
	}()

	fmt.Println("主函数执行")
}
```

### runtime包

虽然说Go编译器将Go的代码编译成本地可执行代码。不需要像java或者.net那样的语言需要一个虚拟机来运行，但其实go是运行在runtime调度器上的，它主要负责内存管理、垃圾回收、栈处理等等。也包含了Go运行时系统交互的操作，控制goroutine的操作，Go程序的调度器可以很合理的分配CPU资源给每一个任务。

Go1.5版本之前默认是单核执行的。从1.5之后使用可以通过`runtime.GOMAXPROCS()`来设置让程序并发执行，提高CPU的利用率。

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    //获取当前GOROOT目录
    fmt.Println("GOROOT:", runtime.GOROOT())
    //获取当前操作系统
    fmt.Println("操作系统:", runtime.GOOS)
    //获取当前逻辑CPU数量
    fmt.Println("逻辑CPU数量：", runtime.NumCPU())

    //设置最大的可同时使用的CPU核数  取逻辑cpu数量
    n := runtime.GOMAXPROCS(runtime.NumCPU())
    fmt.Println(n) //一般在使用之前就将cpu数量设置好 所以最好放在init函数内执行

    //goexit 终止当前goroutine
    //创建一个goroutine
    go func() {
        fmt.Println("start...")
        runtime.Goexit() //终止当前goroutine
        fmt.Println("end...")
    }()
    time.Sleep(3 * time.Second) //主goroutine 休眠3秒 让子goroutine执行完
    fmt.Println("main_end...")
}
```

### Go语言临界资源安全

指并发环境中多个协程之间的共享资源，如果对临界资源处理不当，往往会导致数据不一致的情况。例如：多个goroutine在访问同一个数据资源的时候，其中一个修改了数据，另一个goroutine在使用的时候就不对了。

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

//定义全局变量 表示救济粮食总量
var food = 10

func main() {
	//开启4个协程抢粮食
	go Relief("灾民好家伙1")
	go Relief("灾民好家伙2")
	go Relief("灾民老李头1")
	go Relief("灾民老李头2")

	//让程序休息5秒等待所有子协程执行完毕
	time.Sleep(5 * time.Second)
}

//定义一个发放的方法
func Relief(name string) {
	for {
		if food > 0 { //此时有可能第二个goroutine访问的时候 第一个goroutine还未执行完 所以条件也成立
			time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond) //随机休眠时间
			food--
			fmt.Println(name, "抢到救济粮 ，还剩下", food, "个")
		} else {
			fmt.Println(name, "别抢了 没有粮食了。")
			break
		}
	}
}
```

以上代码出现负数的情况，也是因为Go语言的并发走的太快了，当有一个协程进入执行的时候还没来得及取出数据，另外一个协程也进来了，所以会出现负数的情况，那么如何解决这样的问题，我们不能用休眠的方法让程序等待，因为你并不知道程序会多久执行结束，到底应该让程序休眠多长时间。下面看看如何控制goroutine协程在执行过程中保证数据的安全。

### sync同步包

sync同步包，是Go语言提供的内置同步操作，保证数据统一的一些方法，**WaitGroup 等待一个goroutine的集合执行完成，也叫同步等待组，使用`Add()`方法，来设置要等待一组goroutine 要执行的数量**。**用`Done()`方法来减去执行goroutine集合的数量。使用`Wait()` 方法让`主goroutine`也就是`main`函数进入阻塞状态，等待其他的子goroutine执行结束后，main函数才会解除阻塞状态**

### 互斥锁

互斥锁，当一个goroutine获得锁之后其他的就只能等待当前goroutine执行完成之后解锁后才能访问资源。对应的方法有`上锁Lock()`和`解锁Unlock()`。



```go
package main

import (
	"fmt"
	"sync"
)

//定义全局变量 表示救济粮食总量
var food = 10

//同步等到组对象
var wg sync.WaitGroup

//创建一把锁
var mutex sync.Mutex

func main() {
	wg.Add(4)
	//开启4个协程抢粮食
	go Relief("灾民好家伙")
	go Relief("灾民好家伙2")
	go Relief("灾民老李头")
	go Relief("灾民老李头2")
	wg.Wait() //阻塞主协程，等待子协程执行结束
}

//定义一个发放的方法
func Relief(name string) {
	defer wg.Done()
	for {
		//上锁
		mutex.Lock()
		if food > 0 { //加锁控制之后每次只允许一个协程进来，就会避免争抢
			food--
			fmt.Println(name, "抢到救济粮 ，还剩下", food, "个")
		} else {
			mutex.Unlock() //条件不满足也需要解锁 否则就会造成死锁其他不能执行
			fmt.Println(name, "别抢了 没有粮食了。")
			break
		}
		//执行结束解锁，让其他协程也能够进来执行
		mutex.Unlock()
	}
}
```







