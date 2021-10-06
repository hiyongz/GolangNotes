# Go语言中的通道
通道（channel）是Go 语言中一种特殊的数据类型，通道本身就是并发安全的，可以通过它在多个 goroutine 之间传递数据。通道是Go 语言编程理念：“*Do not communicate by sharing memory; instead, share memory by communicating*”（不要通过共享数据来通信，而应该通过通信来共享数据。）的完美实现，在并发编程中经常会遇到它。下面来介绍一下通道的使用方法。

<!--more-->

## 通道的发送和接收

通道包括双向通道和单向通道，这里双向通道只的是支持发送和接收的通道，而单向通道是只能发送或者只能接收的通道。

### 双向通道

使用make函数声明并初始化一个通道：

```go
ch1 := make(chan string, 3)
```

- `chan` 是表示通道类型的关键字
- `string` 表示该通道类型的元素类型
- `3` 表示该通道的容量为3，最多可以缓存3个元素值。

一个通道相当于一个先进先出（FIFO）的队列，使用操作符 `<-` 进行元素值的发送和接收：

```go
ch1 <- "1"  //向通道ch1发送数据 "1"
```

接收元素值：

```go
elem1 := <- ch1 // 接收通道中的元素值
```

首先接收到的元素为先存入通道中的元素值，也就是先进先出：

```go
package main

import "fmt"

func main() {
	str1 := []string{"hello","world", "!"}
	ch1 := make(chan string, len(str1))

	for _, str := range str1 {
		ch1 <- str
	}
	
	for i := 0; i < len(str1); i++ {
		elem := <- ch1
		fmt.Println(elem)
	}
}
```

执行结果：

```go
hello
world
!
```

### 单向通道

单向通道包括只能发送的通道和只能接收的通道：

```go
var WriteChan = make(chan<- interface{}, 1) // 只能发送不能接收的通道
var ReadChan = make(<-chan interface{}, 1) // 只能接收不能发送的通道
```

单向通道的这种特性可以用来约束函数的输入类型或者输出类型，比如下面的例子约束了只能从通道中接收元素值：

```go
package main

import (
	"fmt"
)

func OnlyReadChan(num int) <-chan int {
	ch := make(chan int, 1)
	ch <- num
	close(ch)
	return ch
}

func main() {

	Chan1 := OnlyReadChan(6)
	num := <- Chan1
	fmt.Println(num)
}
```

执行结果：

```go
6
```

## 通道阻塞

通道操作是**并发安全**的，在同一时刻，只会执行对同一个通道的任意个发送操作中的某一个，直到这个元素值被完全复制进该通道之后，其他针对该通道的发送操作才可能被执行。接收操作也一样。另外，对于通道中的同一个元素值来说，发送操作和接收操作之间也是互斥的。

发送操作和接收操作是原子操作，也就是说，发送操作绝不会出现只复制了一部分的情况，要么还没有复制，要么已经复制完毕。接收操作在准备好元素值的副本之后，一定会删除掉通道中的原值，绝不会出现通道中仍有残留的情况。在进行发送操作和接收操作时，代码会一直阻塞在那里，完成操作后才会继续执行后面的代码。通道的发送操作和接收操作是很快的，那么什么情况下会出现长时间的阻塞呢？下面介绍几种情况。

### 缓冲通道的阻塞

缓冲通道是容量大于0的通道，也就是可以缓存数据的通道。

**1、发送阻塞**

如果缓冲通道已经填满，如果有goroutine继续向该通道发送数据就会阻塞。请看下面的例子：

```go
package main

func main() {
	ch1 := make(chan int, 1)
	ch1 <- 1
	ch1 <- 2
}
```
执行结果：
```go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
...........
```

如果通道可以接收数据（有元素被接收），通道会通知最先等待发送操作的 goroutine再次执行发送操作。

**2、接收阻塞**

类似的，如果通道已空，如果继续进行接收操作就会被阻塞。

```go
package main

func main() {
	ch1 := make(chan int, 1)
	<- ch1
}
```
执行结果：
```go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
...........
```
### 非缓冲通道
非缓冲通道是容量为0的通道，不能缓存数据。

非缓冲通道的数据传递是同步的，发送操作或者接收操作在执行后就会阻塞，需要对应的接收操作或者发送操作执行才会继续传递。由此可以看出缓冲通道使用的是异步方式进行数据传递。

```go
package main

import (
    "fmt"
)

func main() {
    str1 := []string{"hello","world", "!"}
	ch1 := make(chan string, 0)

    go func() {
        for _, str := range str1 {
            ch1 <- str
        }
    }()

	for i := 0; i < len(str1); i++ {
		elem := <- ch1
		fmt.Println(elem)
	}

}
```

执行结果：

```go
hello
world
!  
```

上面的代码中3个goroutine向通道写了三次数据，必须有三次接收，不然会阻塞。

对值为nil的通道进行发送操作和接收操作也会发生阻塞：

```go
var ch1 chan int
ch1 <- 1 // 阻塞
<-ch1 // 阻塞
```



## 通道关闭

可以使用close()方法来关闭通道，通道关闭后，不能再对通道进行发送操作，可以进行接收操作。

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int, 1)
	ch1 <- 1
	close(ch1)
    
    ele := <-ch1
	fmt.Println(ele)  
    
	ch1 <- 2
}
```

执行结果：

```go
1
panic: send on closed channel

goroutine 1 [running]:
.....
```

如果通道关闭时，里面还有元素，进行接收操作时，返回的通道关闭标志仍然为true：

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int, 1)
	ch1 <- 1
	close(ch1)

	ele1, statu1 := <-ch1
	fmt.Println(ele1, statu1)
	ele2, statu2 := <-ch1
	fmt.Println(ele2, statu2)
}
```

执行结果：

```go
1 true
0 false
```

由于通道的这种特性，可以让发送方来关闭通道。前面的例子可以这样写：

```go
package main

import (
    "fmt"
)

func main() {
    str1 := []string{"hello","world", "!"}
	ch1 := make(chan string, 0)

    go func() {
        for _, str := range str1 {
            ch1 <- str
        }
        close(ch1)
    }()

	for i := 0; i < len(str1); i++ {
		elem := <- ch1
		fmt.Println(elem)
	}

}
```

另外，不能对关闭的通道再次关闭：

```go
package main

// import "fmt"

func main() {
	ch1 := make(chan int, 1)
	ch1 <- 1
	close(ch1)
	close(ch1)

}
```

执行结果：

```go
panic: close of closed channel
```

## select语句与通道

select语句通常与通道联用，它是专为通道而设计的。select语句执行时，一般只有一个case表达式或者default语句会被运行。

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int, 1)
	num := 2
	
	select {
		case data := <-ch1:
			fmt.Println("Read data: ", data)
		case ch1 <- num:
			fmt.Println("Write data: ", num)	
		default:
			fmt.Println("No candidate case is selected!")
	}
}
```

执行结果：

```go
Write data:  2
```

需要注意的是，如果没有default默认分支，case表达式都没有满足条件，那么select语句就会被阻塞，直到至少有一个case表达式满足条件为止。

如果同时有多个分支满足条件，会随机选择一个分支执行

for语句与select语句联用时，分支中的break语句只能结束当前select语句的执行，而不会退出for循环。下面的代码永远不会退出循环：

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int, 1)
	for {
		select {
		case ch1 <- 6:
			fmt.Println("Write data: 6")
		case data := <-ch1:
			fmt.Println(data)
			break
		}
	}
}
```

解决方案是使用goto语句和标签。

方法1：

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int, 1)
	num := 6
	for {
		select {
		case ch1 <- num:
			fmt.Println("Write data: ", num)
		case data := <-ch1:
			fmt.Println("Read data: ", data)
			goto loop
		}
	}	
	loop:
	fmt.Println(ch1)
}
```

执行结果：

```go
Write data:  6
Read data:  6
0xc00000e0e0
```

方法2：

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int, 1)
	num := 6
	loop:
	for {
		select {
		case ch1 <- num:
			fmt.Println("Write data: ", num)
		case data := <-ch1:
			fmt.Println("Read data: ", data)
			break loop
		}
	}
	fmt.Println(ch1)
}
```

执行结果：

```go
Write data:  6
Read data:  6
0xc0000e4000
```



## 小结

本文主要介绍了通道的基本操作：初始化、发送、接收和关闭，要注意在什么情况下会引起通道阻塞。select语句通常与通道联用，介绍了分支的选择规则以及for语句与select语句联用时如何退出循环。

通道是 Go 语言并发编程的重要实现基础，还是有必要掌握的。





