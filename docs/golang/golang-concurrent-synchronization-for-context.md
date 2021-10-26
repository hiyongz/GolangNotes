# Go语言并发编程：上下文Context

context.Context类型是在 Go 1.7 版本引入到标准库的，上下文Context主要用来在goroutine之间传递截止日期、停止信号等上下文信息，并且它是并发安全的，可以控制多个goroutine，因此它可以很方便的用于并发控制和超时控制，标准库中的一些代码包也引入了Context参数，比如os/exec包、net包、database/sql包，等等。下面来介绍Context类型的使用方法。


## Context介绍

Context类型的应用还是比较广的，比如http后台服务，多个客户端或者请求会导致启动多个goroutine来提供服务，通过Context，我们可以很方便的实现请求数据的共享，比如token值，超时时间等，可以让系统避免额外的资源消耗。

### Context类型

Context类型是一个接口类型，定义了4个方法：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}    
```

- `Deadline()` ：获取设置的截止日期，到截止日期时，Context会自动发起取消请求，ok为false表示没有设置截止日期；
- `Done()`：返回一个只读通道chan，在当前工作完成、超时或者context被取消后关闭；
- `Err()`：返回Context结束原因，取消时返回`Canceled` 错误，超时返回 `DeadlineExceeded` 错误。
- `Value`：获取 key 对应的 value值，可以用它来传递额外的信息和信号。



## Context 衍生

Context值是可以繁衍的，也就是可以通过一个Context值产生任意个子值，这些子值携带了父值的属性和数据，也可以响应通过其父值传达的信号。

Context根节点是一个已经在context包中预定义好的Context值，是全局唯一的，它既不可以被撤销，也不能携带任何数据，可以通过调用context.Background函数获取到它。

context包提供了4个用于繁衍Context值的函数：

- `WithCancel`：基于parent context 产生一个可撤销（cancel）的子context
- `WithDeadline`：产生可以定时撤销的子context，达到截止日期后，context会收到cancel通知。
- `WithTimeout`：与 `WithDeadline` 类似，产生可以定时撤销的子context
- `WithValue`：产生携带额外数据的子context

下面介绍这4个函数的使用示例。

### WithCancel

WithCancel返回两个结果值，第一个是可撤销的Context值，第二个则是用于触发撤销信号的函数。在撤销函数被执行后，先关闭内部的接收通道，然后向所有子Context发送cancel信号，最终断开与父Context之间的关联。其中cancel信号的传递采用的是深度优先搜索算法。

仍然是取钱的例子，要求是账户的钱大于10000后停止存钱：

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync/atomic"
	"time"
)

var (
	balance int32
)

// 存钱
func deposit(value int32, id int, deferFunc func()) {
	defer func() {
		deferFunc()
	}()

	for {
		currBalance := atomic.LoadInt32(&balance)
		newBalance := currBalance + value
		time.Sleep(time.Millisecond * 500)

		if atomic.CompareAndSwapInt32(&balance, currBalance, newBalance) {
			fmt.Printf("ID: %d, 存 %d 后的余额: %d\n", id, value, balance)
			break
		} else {
			// fmt.Printf("操作失败\n")
		}
	}
}

// 取钱
func withdraw(value int32) {	
	for {
		currBalance := atomic.LoadInt32(&balance)
		newBalance := currBalance - value
		if atomic.CompareAndSwapInt32(&balance, currBalance, newBalance) {
			fmt.Printf("取 %d 后的余额: %d\n", value, balance)
			break
		}
	}
}

func WithCancelDemo() {
	total := 10000
	ctx, cancelFunc := context.WithCancel(context.Background())
	for i := 1; i <= 100; i++ {
		num := rand.Intn(2000) // 随机数
		go deposit(int32(num), i, func() {
			if atomic.LoadInt32(&balance) >= int32(total) {
				cancelFunc()
			}
		})
	}
	<-ctx.Done()
	withdraw(10000)
	fmt.Println("退出")

}

func main() {	
	WithCancelDemo()
}

func init() {
	balance = 1000 // 初始账户余额为1000
}

```

执行结果：

```go
ID: 95, 存 1940 后的余额: 2940
ID: 19, 存 1237 后的余额: 4177
ID: 78, 存 1463 后的余额: 5640
ID: 17, 存 1211 后的余额: 6851
ID: 80, 存 420 后的余额: 7271
ID: 28, 存 888 后的余额: 8159
ID: 32, 存 408 后的余额: 8567
ID: 50, 存 1353 后的余额: 9920
ID: 38, 存 631 后的余额: 10551
取 10000 后的余额: 551
退出
```



### WithDeadline



设置截止日期，达到截止日期后停止存钱：

```go
func DeadlineDemo() {
	total := 10000
	deadline := time.Now().Add(2 * time.Second)
	ctx, cancelFunc := context.WithDeadline(context.Background(), deadline)
	for i := 1; i <= 100; i++ {
		num := rand.Intn(2000) // 随机数
		go deposit(int32(num), i, func() {
			if atomic.LoadInt32(&balance) >= int32(total) {
				cancelFunc()
			}
		})
	}
	select {
	case <-ctx.Done():
	    fmt.Println(ctx.Err())
	}
	fmt.Println("超时退出")	
}
```

截止日期参数deadline是一个时间对象：time.Time

执行结果：

```go
ID: 7, 存 1410 后的余额: 1961
ID: 5, 存 81 后的余额: 2042
ID: 69, 存 783 后的余额: 2825
context deadline exceeded
超时退出
```

WithDeadline和WithTimeout函数生成的Context值也是可撤销的，可以实现自动定时撤销，也可以在截止时间达到之前进行手动撤销（代码中的cancelFunc()操作）。

### WithTimeout

和WithDeadline不同之处在于，时间参数为持续时间：time.Duration：

```go
func WithTimeoutDemo() {
	total := 10000
	ctx, cancelFunc := context.WithTimeout(context.Background(), 2*time.Second)
	for i := 1; i <= 100; i++ {
		num := rand.Intn(2000) // 随机数
		go deposit(int32(num), i, func() {
			if atomic.LoadInt32(&balance) >= int32(total) {
				cancelFunc()
			}
		})
	}
	select {
	case <-ctx.Done():
	    fmt.Println(ctx.Err())
	}
	fmt.Println("超时退出")	
}
```

执行结果：

```go
ID: 36, 存 1356 后的余额: 4181
ID: 100, 存 1598 后的余额: 5779
ID: 25, 存 47 后的余额: 5826
ID: 10, 存 292 后的余额: 6118
context deadline exceeded
超时退出
```



### WithValue

WithValue函数产生的Context可以携带数据，和另外3种函数不同，它是不可撤销的。Value方法用来获取数据，没有提供改变数据的方法。

WithValue函数产生的Context携带的值可以在子Context中传递。

```go
func WithValueDemo() {

	rootNode := context.Background()
	ctx1, cancelFunc := context.WithCancel(rootNode)
	defer cancelFunc()

	ctx2 := context.WithValue(ctx1, "key2", "value2")
	ctx3 := context.WithValue(ctx2, "key3", "value3")
	fmt.Printf("ctx3: key2 %v\n", ctx3.Value("key2"))
	fmt.Printf("ctx3: key3 %v\n", ctx3.Value("key3"))

	fmt.Println()

	ctx4, _ := context.WithTimeout(ctx3, time.Hour)
	fmt.Printf("ctx4: key2 %v\n", ctx4.Value("key2"))
	fmt.Printf("ctx4: key3 %v\n", ctx4.Value("key3"))
	
}
```

执行结果：

```go
ctx3: key2 value2
ctx3: key3 value3

ctx4: key2 value2
ctx4: key3 value3
```



## 小结

Context类型是一个可以实现多 goroutine 并发控制的同步工具。Context类型主要分为三种，即：根Context、可撤销的Context和携带数据的Context。根Context和衍生的Context构成一颗Context树。需要注意的是，携带数据的Context不能被撤销，可撤销的Context无法携带数据。

Context比sync.WaitGroup更加灵活，在使用WaitGroup时，我们需要确定执行子任务的 goroutine 数量，如果不知道这个数量，使用WaitGroup就有风险了，采用Context就很容易解决了。



