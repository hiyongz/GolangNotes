# Go语言并发编程：WaitGroup


我们知道，在并发编程中，主要线程需要等待子线程运行结束后才能退出，go语言中，主 goroutine 等待其他 goroutine 运行结束可以使用通道来解决，具体实现可以参考文章[Go语言并发编程：互斥锁](https://blog.csdn.net/u010698107/article/details/120248679)中的例子。使用通道可能不是很简洁，本文介绍另一种方法，也就是sync包中的WaitGroup类型来等待 goroutine执行完成。


sync.WaitGroup类型主要包括3个方法：
- Add：用于需要等待的 goroutine 的数量
- Done：对计数器的值进行减一操作，一般在需要等待的goroutine运行完成之前执行这一操作，可以通过defer语句调用它
- Wait：用于阻塞当前的 goroutine，直到其所属值中的计数器归零


下面直接修改[Go语言并发编程：互斥锁](https://blog.csdn.net/u010698107/article/details/120248679)中的例子，使用WaitGroup来等待goroutine：
```go
package main

import (
	"flag"
	"fmt"
	"sync"
)

var (
    mutex   sync.Mutex
    balance int
    protecting uint  // 是否加锁
    sign = make(chan struct{}, 10) //通道，用于等待所有goroutine
)

var wg sync.WaitGroup

// 存钱
func deposit(value int) {
    if protecting == 1 {
        mutex.Lock()
        defer mutex.Unlock()
    }

    fmt.Printf("余额: %d\n", balance)
    balance += value
    fmt.Printf("存 %d 后的余额: %d\n", value, balance)
    fmt.Println()

    wg.Done()
}

// 取钱
func withdraw(value int) {
    defer wg.Done()
    if protecting == 1 {
        mutex.Lock()
        defer mutex.Unlock()
    }
    fmt.Printf("余额: %d\n", balance)
    balance -= value
    fmt.Printf("取 %d 后的余额: %d\n", value, balance)
    fmt.Println()
}

func main() {
	wg.Add(10)
    for i:=0; i < 5; i++ {
        go withdraw(500) // 取500
        go deposit(500)  // 存500
    }
    wg.Wait()

    fmt.Printf("当前余额: %d\n", balance)
}

func init() {
    balance = 1000 // 初始账户余额为1000
    flag.UintVar(&protecting, "protecting", 1, "是否加锁，0表示不加锁，1表示加锁")
}

```

先声明了一个WaitGroup类型的全局变量wg。main方法中的wg.Add(10)表示有10个goroutine需要等待，wg.Wait()表示等待那10个goroutine执行结束。


另外，WaitGroup值是可以被复用的，wg归0后，可以继续使用：
```go
func main() {
	wg.Add(5)
    for i:=0; i < 5; i++ {
        go deposit(500)  // 存500
    }
    wg.Wait()

    time.Sleep(time.Duration(3) * time.Second)
    
    wg.Add(5)
    for i:=0; i < 5; i++ {
        go withdraw(500) // 取500
    }
    wg.Wait()

    fmt.Printf("当前余额: %d\n", balance)
}
```

如果你有多组任务，而这些任务需要串行执行，可以使用上面这种写法。

比如实现按顺序存钱：

```go
func main() {
    for i:=0; i < 5; i++ {
        wg.Add(1)
        go deposit(500+i)  // 存500
        wg.Wait()
    }

    fmt.Printf("当前余额: %d\n", balance)
}
```

执行结果：

```go
余额: 1000
存 500 后的余额: 1500

余额: 1500
存 501 后的余额: 2001

余额: 2001
存 502 后的余额: 2503

余额: 2503
存 503 后的余额: 3006

余额: 3006
存 504 后的余额: 3510

当前余额: 3510
```







