# Go语言并发编程：互斥锁
在并发编程中，多个Goroutine访问同一块内存资源时可能会出现竞态条件，我们需要在临界区中使用适当的同步操作来以避免竞态条件。Go 语言中提供了很多同步工具，本文将介绍互斥锁Mutex和读写锁RWMutex的使用方法。

<!--more-->

## 互斥锁Mutex

### Mutex介绍

Go 语言的同步工具主要由 sync 包提供，互斥锁 (Mutex) 与读写锁 (RWMutex) 就是sync 包中的方法。

互斥锁可以用来保护一个临界区，保证同一时刻只有一个 goroutine 处于该临界区内。主要包括锁定（Lock方法）和解锁（Unlock方法）两个操作，首先对进入临界区的goroutine进行锁定，离开时进行解锁。

使用互斥锁 (Mutex)时要注意以下几点：

1. 不要重复锁定互斥锁，否则会阻塞，也可能会导致死锁（deadlock）；
2. 要对互斥锁进行解锁，这也是为了避免重复锁定；
3. 不要对未锁定或者已解锁的互斥锁解锁；
4. 不要在多个函数之间直接传递互斥锁，sync.Mutex类型属于值类型，将它传给一个函数时，会产生一个副本，在函数中对锁的操作不会影响原锁

总之，一个互斥锁只用来保护一个临界区，加锁后记得解锁，对于每一个锁定操作，都要有且只有一个对应的解锁操作，也就是加锁和解锁要成对出现，最保险的做法时使用**defer语句**解锁。

### Mutex使用实例

下面的代码模拟取钱和存钱操作：

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

// 存钱
func deposit(value int) {
    defer func() {
        sign <- struct{}{}
    }()

    if protecting == 1 {
        mutex.Lock()
        defer mutex.Unlock()
    }

    fmt.Printf("余额: %d\n", balance)
    balance += value
    fmt.Printf("存 %d 后的余额: %d\n", value, balance)
    fmt.Println()

}

// 取钱
func withdraw(value int) {
    defer func() {
        sign <- struct{}{}
    }()
    
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
    
    for i:=0; i < 5; i++ {
        go withdraw(500) // 取500
        go deposit(500)  // 存500
    }

    for i := 0; i < 10; i++ {
		<-sign
	}
    fmt.Printf("当前余额: %d\n", balance)
}

func init() {
    balance = 1000 // 初始账户余额为1000
    flag.UintVar(&protecting, "protecting", 0, "是否加锁，0表示不加锁，1表示加锁")
}

```

> 上面的代码中，使用了通道来让主 goroutine 等待其他 goroutine 运行结束，每个子goroutine在运行结束之前向通道发送一个元素，主 goroutine 在最后从这个通道接收元素，接收次数与子goroutine个数相同。接收完后就会退出主goroutine。

代码使用协程实现多次（5次）对一个账户进行存钱和取钱的操作，先来看不加锁的情况：


```go
余额: 1000
存 500 后的余额: 1500

余额: 1000
取 500 后的余额: 1000

余额: 1000
存 500 后的余额: 1500

余额: 1000
取 500 后的余额: 1000

余额: 1000
存 500 后的余额: 1500

余额: 1000
取 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 1000
存 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 1000
存 500 后的余额: 1000

当前余额: 1000
```

可以看到出现了混乱，比如第二次1000的余额取500后还是1000，这种对同一资源的竞争出现了竞态条件(Race Condition)。

下面来看加锁的执行结果：

```go
余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

余额: 1000
存 500 后的余额: 1500

余额: 1500
取 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

当前余额: 1000
```

加锁后就正常了。

下面介绍更细化的互斥锁：读/写互斥锁RWMutex。

## 读写锁RWMutex

### RWMutex介绍

读/写互斥锁RWMutex包含了读锁和写锁，分别对共享资源的“读操作”和“写操作”进行保护。sync.RWMutex类型中的Lock方法和Unlock方法分别用于对写锁进行锁定和解锁，而它的RLock方法和RUnlock方法则分别用于对读锁进行锁定和解锁。

有了互斥锁Mutex，为什么还需要读写锁呢？因为在很多并发操作中，并发读取占比很大，写操作相对较少，读写锁可以并发读取，这样可以提供服务性能。读写锁具有以下特征：

| 读写锁 | 读锁 | 写锁 |
| ------ | ---- | ---- |
| 读锁   | Yes  | No   |
| 写锁   | No   | No   |

也就是说，

- 如果某个共享资源受到读锁和写锁保护时，其它goroutine不能进行写操作。换句话说就是读写操作和写写操作不能并行执行，也就是读写互斥；
- 受读锁保护时，可以同时进行多个读操作。

在使用读写锁时，还需要注意：

1. 不要对未锁定的读写锁解锁；
2. 对读锁不能使用写锁解锁
3. 对写锁不能使用读锁解锁

### RWMutex使用实例

改写前面的取钱和存钱操作，添加查询余额的方法：

```go
package main

import (
	"fmt"
	"sync"
)

// account 代表计数器。
type account struct {
	num uint         // 操作次数
	balance int		 // 余额
	rwMu  *sync.RWMutex // 读写锁
}

var sign = make(chan struct{}, 15) //通道，用于等待所有goroutine

// 查看余额：使用读锁
func (c *account) check() {
	defer func() {
        sign <- struct{}{}
    }()
	c.rwMu.RLock()
	defer c.rwMu.RUnlock()
	fmt.Printf("%d 次操作后的余额: %d\n", c.num, c.balance)
}

// 存钱：写锁
func (c *account) deposit(value int) {
	defer func() {
        sign <- struct{}{}
    }()
    c.rwMu.Lock()
	defer c.rwMu.Unlock()	

	fmt.Printf("余额: %d\n", c.balance)   
	c.num += 1
    c.balance += value
    fmt.Printf("存 %d 后的余额: %d\n", value, c.balance)
    fmt.Println() 
}

// 取钱：写锁
func (c *account) withdraw(value int) {
    defer func() {
        sign <- struct{}{}
    }()
	c.rwMu.Lock()
	defer c.rwMu.Unlock()	  
	fmt.Printf("余额: %d\n", c.balance)     
	c.num += 1
    c.balance -= value
	fmt.Printf("取 %d 后的余额: %d\n", value, c.balance)
    fmt.Println() 	
}


func main() {
	c := account{0, 1000, new(sync.RWMutex)}

	for i:=0; i < 5; i++ {
        go c.withdraw(500) // 取500
        go c.deposit(500)  // 存500
		go c.check()
    }

    for i := 0; i < 15; i++ {
		<-sign
	}
	fmt.Printf("%d 次操作后的余额: %d\n", c.num, c.balance)

}
```

执行结果：

```go
余额: 1000
取 500 后的余额: 500

1 次操作后的余额: 500
1 次操作后的余额: 500
1 次操作后的余额: 500
1 次操作后的余额: 500
1 次操作后的余额: 500
余额: 500
存 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

余额: 1000
存 500 后的余额: 1500

余额: 1500
取 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

余额: 1000
取 500 后的余额: 500

余额: 500
存 500 后的余额: 1000

10 次操作后的余额: 1000
```

读写锁和互斥锁的不同之处在于读写锁把对共享资源的读操作和写操作分开了，可以实现更复杂的访问控制。

## 小结

读写锁也是一种互斥锁，它是互斥锁的扩展。在使用时需要注意：

1. 加锁后一定要解锁
2. 不要重复加锁或者解锁
3. 不解锁未锁定的锁
4. 不要传递互斥锁







