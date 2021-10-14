# Go语言并发编程：sync.Once

sync.Once用于保证某个动作只被执行一次，可用于单例模式中，比如初始化配置。我们知道init()函数也只会执行一次，不过它是在main()函数之前执行，如果想要在代码执行过程中只运行某个动作一次，可以使用sync.Once，下面来介绍一下它的使用方法。


先来看下面的代码：

```go
package main

import (
	"fmt"
	"sync"
)


func main() {
	var num = 6
	var once sync.Once

	add_one := func() {
		num = num + 1
	}

	minus_one := func() {
		num = num - 1
	}	

	once.Do(add_one)
	fmt.Printf("The num: %d\n", num)
	once.Do(minus_one)
	fmt.Printf("The num: %d\n", num)
}
```

执行结果：

```go
The num: 7
The num: 7
```

sync.Once类型提供了一个Do方法，Do方法只接受一个参数，且参数类型必须是func() ，也就是没有参数声明和结果声明的函数。

Do方法只会执行首次被调用时传入的那个函数，只执行一次，也不会执行其它函数。上面的例子中，即使传入的函数不同，也只会执行第一次传入的那个函数。如果有多个只执行一次的函数，需要为每一个函数分配一个sync.Once类型的值：
```go
func main() {
	var num = 6
	var once1 sync.Once
	var once2 sync.Once

	add_one := func() {
		num = num + 1
	}

	minus_one := func() {
		num = num - 1
	}	

	once1.Do(add_one)
	fmt.Printf("The num: %d\n", num)
	once2.Do(minus_one)
	fmt.Printf("The num: %d\n", num)
}
```

sync.Once类型是一个结构体类型，一个是名为done的uint32类型字段，还有一个互斥锁m。

```go
type Once struct {
	done uint32
	m    Mutex
}
```

done字段的值只可能是0或者1，Do方法首次调用完成后，done的值就变为了1。done的值使用四个字节的uint32类型的原因是为了保证对它的操作是“原子操作”，通过调用atomic.LoadUint32函数获取它的值，如果为1，直接返回，不会执行函数。

如果为0，Do方法会立即锁定字段m，如果这里不加锁，多个goroutine 同时执行到Do方法时判断都为0，则都会执行函数，所以Once是并发安全的。

加锁之后，会再次检查done字段的值，如果满足条件，执行传入的函数，并用原子操作函数atomic.StoreUint32将done的值设置为1。

下面是Once的源码：

```go
func (o *Once) Do(f func()) {

	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

源码非常简洁，和GoF 设计模式中的单例模式非常相似。
