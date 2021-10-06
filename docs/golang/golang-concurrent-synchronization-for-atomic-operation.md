# Go语言并发编程：原子操作
在程序执行过程中，操作系统会进行线程调度，同一时刻能同时执行的程序数量跟CPU的内核线程数有关，比如4核CPU，同时最多只能有4个线程。Go 语言中的运行时系统也会对goroutine进行调度，调度器会频繁地让goroutine处于中断或者运行状态，这就不能保证代码执行的原子性（atomicity），即使使用互斥锁也不能保证原子性操作。Go语言中的atomic包提供了原子操作方法，下面来介绍它的使用方法。

<!--more-->

原子操作过程中是不允许中断的，是绝对并发安全的。由于原子操作不允许中断，所以它非常影响系统执行效率，因此，Go 语言的sync/atomic包只针对少数数据类型提供了原子操作函数。

## atomic原子操作类型和方法

支持的数据类型主要有7个：int32、int64、uint32、uint64、uintptr，Pointer（unsafe包）以及Value类型，Value类型可以用来存储任意类型的值。

对这些类型的操作函数包括：

- 增加 (Add)：`atomic.AddInt32(addr *int32, delta int32)`
- 加载（Load）：`atomic.LoadInt32(addr *int32)`
- 存储（Store）：`atomic.LoadInt32(addr *int32)`
- 交换（Swap）：`atomic.SwapInt32(addr *int32, new int32)`
- 比较并交换（CompareAndSwap）: `atomic.CompareAndSwapInt32(addr *int32, old int32, new int32)`


其中，unsafe.Pointer类型没有add操作，Value类型只要Load和Store两个方法。

注意，第一个参数值为被操作值的指针，原子操作根据指针定位到该值的内存地址，操作这个内存地址上的数据。



## Add 增加

Add可以用于增加操作：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	num := int32(20)
	atomic.AddInt32(&num, 3)
	fmt.Println(num)  // 23
}
```

Add也可以做减法操作，其中AddInt32的第二个参数int32是有符号整型，所以delta值设置为负整数就是减法操作了。

```go
num := int32(18)
atomic.AddInt32(&num, -3)
fmt.Println(num)
```

而uint32和uint64是无符号的，如果想对这两种类型做减法操作需要做一下转换，比如先把delta值转换为有符号类型，然后再转换为无符号类型：

```go
num := uint32(18)
delta := int32(-3)
atomic.AddUint32(&num, uint32(delta))
fmt.Println(num)
```

也可以使用如下方式：

```go
atomic.AddUint32(&num, ^uint32(-(-3)-1))
```



## Load 加载

Load可以实现对值的原子读取：

```go
num := int32(20)
atomic.LoadInt32(&num)
fmt.Println(atomic.LoadInt32(&num))
```

## Store 存储

原子的存储某个值：

```go
num := int32(20)
atomic.StoreInt32(&num, 30)
fmt.Println(num) // 30
```

## Swap 交换

将新的值赋给被操作的旧值，并返回旧值

```go
num := int32(20)
old := atomic.SwapInt32(&num, 60)
fmt.Println(num) // 60
fmt.Println(old) // 20
```


## CompareAndSwap 比较并交换

比较并交换（Compare And Swap，CAS操作 ）和交换（Swap）不同，会先进行比较，满足条件后再进行交换操作，将新值赋给变量。返回值为true或者false，true表示执行了交换操作。

```go
num:= int32(18)
atomic.CompareAndSwapInt32(&num, 20, 0)
fmt.Printf("The number: %d\n", num)
atomic.CompareAndSwapInt32(&num, 18, 0)
fmt.Printf("The number: %d\n", num)
```

执行结果：

```go
The number: 18
The number: 0
```

CAS操作可以用来实现自旋锁（spinlock），下面先来介绍一下什么是自旋锁，自旋锁和互斥锁都可以用来保护共享资源，它们的区别在于，资源被互斥锁锁定时，其它要操作资源的线程会进入睡眠状态；如果是自旋锁，线程将循环等待，不会释放cpu，直到获取到锁才会退出循环。由于自旋锁的这种特性，一般会对等待时间或者尝试次数进行一定的限制。

由于自旋锁不需要进行上下文切换，它的效率比互斥锁高，适用于保持锁的时间比较短，并且不会频繁操作共享资源的场景。

下面的代码实现一个简单的自旋锁，存满10000后全部取出：


```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
	"sync"
)

var (
    balance int32
	wg sync.WaitGroup
)

// 存钱
func deposit(value int32) {
	for {
		
		fmt.Printf("余额: %d\n", balance)
		atomic.AddInt32(&balance, value)
		fmt.Printf("存 %d 后的余额: %d\n", value, balance)
		fmt.Println()
		if balance == 10000 {
			break
		}
		time.Sleep(time.Millisecond * 500)
	}
    wg.Done()
}

// 取钱
func withdrawAll(value int32) {
    defer wg.Done()
	
	for {
		if atomic.CompareAndSwapInt32(&balance, value, 0) {
			break
		}
		time.Sleep(time.Millisecond * 500)
	}
	fmt.Printf("余额: %d\n", value)
    fmt.Printf("取 %d 后的余额: %d\n", value, balance)
    fmt.Println()    
}

func main() {
	wg.Add(2)	
	go deposit(1000)  // 每次存1000
	go withdrawAll(10000)
    wg.Wait()

    fmt.Printf("当前余额: %d\n", balance)
}

func init() {
    balance = 1000 // 初始账户余额为1000    
}
```


## atomic.Value

Value类型可以被用来“原子地”存储（Store）和加载（Load）任意的值。

```go
var valu atomic.Value
valu := [...]int{1, 2, 3}
box.Store(valu)
fmt.Println(valu.Load())
```

使用Value类型时需要注意以下事项：

1、Value不能用来存储nil值。

2、一个Value变量不能存储不同类型的值，存储的类型只能是第一个存储值的类型。

```go
var box atomic.Value
v1 := "123"
box.Store(v1)
v2 := 123	
box.Store(v2)
```

上面的写法会引发一个panic：`panic: sync/atomic: store of inconsistently typed value into Value`

3、尽量不要使用Value存储引用类型的值。

先来看下面的例子：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
    var valu atomic.Value
    v1 := []int{1, 2, 3}
    valu.Store(v1)
    fmt.Println(valu.Load())
    v1[1] = 6
    fmt.Println(valu.Load())
}
```

执行结果：

```go
[1 2 3]
[1 6 3]
```

修改引用类型的值相当于修改了valu中存储的值，可以使用深拷贝copy方法来解决这个漏洞：

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
    var valu atomic.Value
    v1 := []int{1, 2, 3}
	store := func(v []int) {
		replica := make([]int, len(v))
		copy(replica, v)
		valu.Store(replica)
	}
	fmt.Printf("Store %v to box6.\n", v6)
	store(v1)
	fmt.Println(valu.Load())
    v1[1] = 6
    fmt.Println(valu.Load())
}
```
执行结果：

```go
[1 2 3]
[1 2 3]
```




## 小结

原子操作函数支持的数据类型有限，互斥锁可能使用的场景更多一些，在可以使用原子操作的情况下还是建议使用它，因为相对来说原子操作函数的执行速度比互斥锁快，且使用简单。另外在使用 CAS 操作时，要防止进入死循环，导致“阻塞”流程。

在使用Value类型时要注意尽量不要存储引用类型的值，是非并发安全的。











