# Go语言基础语法（一）
本文介绍一些Go语言的基础语法。
<!--more-->

## go简单小例子
先来看一个简单的go语言代码：
```go
package main
import "fmt"

// 加法运算
func add(x, y int) int {
	return x + y
}

func init() {	
	fmt.Println("main  init....")
}

func main() {
	var value1 int = 2
	var value2 = 3
	sum := add(value1,value2)
	fmt.Printf("%d + %d = %d",value1,value2,sum)
}
```
- `package main`：定义package 包名称为main，表示当前文件所属的包
- `import "fmt"`：导入Go标准库中的 fmt 模块，主要用于打印输出。go提供了很多标准库，具体可参考[Golang标准库文档](https://studygolang.com/pkgdoc)。
- `init()`：init()函数在main()函数之前执行。
- `main()`：main函数，是当前程序的入口，init()以及main()函数都无法被显式的调用。

go语言的注释方法：

- 单行注释：`//`
- 多行注释：`/*    */`

代码执行结果：

```bash
$ go run demo.go
main  init....
2 + 3 = 5
```

下面来进一步介绍go的基础语法。

## 格式化输出

go语言中格式化输出可以使用 fmt 和 log 这两个标准库，

### fmt

常用方法：

- `fmt.Printf`：格式化输出
- `fmt.Println`：仅打印，不能转义，会在输出结束后添加换行符。
- `fmt.Print`：和Println类似，但不添加换行符。
- `fmt.Sprintf`：格式化字符串并赋值给新的字符串

示例代码：

```go
package main

import (
	"fmt"
)

func main() {
	var age = 22
	fmt.Printf("I'm %d years old\n", age)

	str1 := "Hello world !"
	fmt.Printf("%s\n", str1)
	fmt.Printf(str1)
	fmt.Print("\n")
	str_hex := fmt.Sprintf("% 02x", str1)
	fmt.Printf("type of str_hex: %T\n", str_hex)
	fmt.Println(str_hex)
}

```

执行结果：

```go
I'm 22 years old
Hello world !
Hello world !
type of str_hex: string
48 65 6c 6c 6f 20 77 6f 72 6c 64 20 21
```

更多格式化方法可以访问[https://studygolang.com/pkgdoc](https://studygolang.com/pkgdoc)中的fmt包。

### log

log包实现了简单的日志服务，也提供了一些格式化输出的方法。
- `log.Printf`：格式化输出，和fmt.Printf类似
- `log.Println`：和fmt.Println类似
- `log.Print`：和fmt.Print类似

```go
package main

import (
	"log"
)

func main() {
	var age = 22
	log.Printf("I'm %d years old", age)

	str1 := "Hello world !"
	log.Println(str1)
	log.Print(str1)
	log.Printf("%s", str1)
}

```
执行结果：
```go
2021/08/12 16:52:12 I'm 22 years old
2021/08/12 16:52:12 Hello world !
2021/08/12 16:52:12 Hello world !
2021/08/12 16:52:12 Hello world !
```

下面来介绍一下go的数据类型

## 数据类型

下表列出了go语言的数据类型：

| 类型              | 说明                                                         | 示例                                  |
| :---------------- | :----------------------------------------------------------- | :------------------------------------ |
| 布尔类型 bool     | 可以是常量 true 或者 false                                   | `var flag bool = true`                |
| 数字类型          | 有符号整型：int8、int16、int32、int64<br />无符号整型：uint8、uint16、uint32、uint64<br />int和uint的具体长度取决于CPU位数<br />浮点型：float32、float64<br /> | `var num int = 2`                     |
| 字符串类型 string | 是UTF-8 编码的字符序列，只能使用双引号（""）或反引号（``）定义 | `var a = “test”`                      |
| 数组 array        | 数组类型是由**固定长度**的特定类型元素组成的序列，长度和容量相同 | `var myarr [5] int{1,2,3}`            |
| 切片 slice        | 切片类型是一种动态数组，是可变长的，长度和容量可相同也可以不相同 | `var myslice = []int{1, 2, 3}`        |
| 结构体 struct     | 结构体是多个任意类型的变量组合                               |                                       |
| 字典 map          | 存储键值对的集合                                             | `var mymap map[int]string`            |
| 通道 channel      | 相当于一个先进先出（FIFO）的队列，可以利用通道在多个 goroutine 之间传递数据 | `ch := make(chan int, 3)`             |
| 接口 interface    | 接口类型定义方法的集合，它无法实例化                         |                                       |
| 函数 func         | 函数类型是一种对一组输入、输出进行模板化的重要工具           | `type add func(a int, b int) (n int)` |
| 指针 pointer      | go语言的指针类型和C/C++中的指针类型类似，是指向某个确切的内存地址的值 |                                       |

int、float、bool、string、数组和struct属于值类型，这些类型的变量直接指向存在内存中的值；slice、map、chan、pointer等是引用类型，存储的是一个地址，这个地址存储最终的值。

## 常量声明

常量是在程序编译时就确定下来的值，程序运行时无法改变。

```go
package main

import (
	"fmt"
)

func main() {
	const name string = "zhangsan"
	fmt.Println(name)

	const course1, course2 = "math", "english"
	fmt.Println(course1, course2)

	const age = 20
	age = age + 1 // 不能改变age
	fmt.Println(age)
}
```

执行结果：

```go
# command-line-arguments
.\test_const.go:15:6: cannot assign to age (declared const)
```



## 变量声明

go的变量声明主要包括三种方法：

- 变量声明可指定变量类型，如果没有初始化，则变量默认为零值。

- 也可以不指定数据类型，由go自己判断。
- var可以省略，使用`:=`进行声明。注意：`:=` 左边的变量必须是没有声明新的变量，否则会编译错误。

```go
package main

import (
	"fmt"
)

func main() {
	var name string = "zhangsan"
	fmt.Println(name)

	var hight float32
	fmt.Println(hight)

	var course1, course2 = "math", "english"
	fmt.Println(course1, course2)

	age := 20
	age = age + 1
	fmt.Println(age)

	var (
		name1 string = "zhangsan"
		name2 string = "lishi"
	)
	fmt.Println(name1, name2)
}
```

执行结果：

```go
zhangsan
0
math english
21
zhangsan lishi
```

## 运算符

Go 语言的运算符主要包括算术运算符、关系运算符、逻辑运算符、位运算符、赋值运算符以及指针相关运算符。

算术运算符：

| 运算符 | 说明 |
| :----- | :--- |
| +      | 加   |
| -      | 减   |
| *      | 乘   |
| /      | 除   |
| %      | 求余 |
| ++     | 自增 |
| --     | 自减 |

关系运算符：

| 运算符 | 说明       |
| :----- | :--------- |
| ==     | 是否相等   |
| !=     | 是否不相等 |
| >      | 大于       |
| <      | 小于       |
| >=     | 大于等于   |
| <=     | 小于等于   |

逻辑运算符：

| 运算符 | 说明     |
| :----- | :------- |
| &&     | 逻辑 AND |
| \|\|   | 逻辑 OR  |
| !      | 逻辑 NOT |

位运算符：

| 运算符 | 说明                           |
| :----- | :----------------------------- |
| &      | 按位与，两个数对应的二进位相与 |
| \|     | 按位或                         |
| ^      | 按位异或                       |
| <<     | 左移，左移n位表示乘以2的n次方  |
| >>     | 右移，右移n位表示除以2的n次方  |

赋值运算符：

| 运算符 | 说明           |
| :----- | :------------- |
| =      | 简单赋值运算符 |
| +=     | 相加后再赋值   |
| -=     | 相减后再赋值   |
| *=     | 相乘后再赋值   |
| /=     | 相除后再赋值   |
| %=     | 求余后再赋值   |
| <<=    | 左移后赋值     |
| >>=    | 右移后赋值     |
| &=     | 按位与后赋值   |
| ^=     | 按位异或后赋值 |
| \|=    | 按位或后赋值   |

指针相关运算符：

| 运算符 | 说明                                                         |
| :----- | :----------------------------------------------------------- |
| &      | 返回变量在内存中的地址                                       |
| *      | 如果`*`后面是指针，表示取指针指向的值；如果`*`后面是类型，则表示一个指向该类型的指针。 |

## 条件语句

下面介绍一下go语言中的if语句和switch语句。另外还有一种控制语句叫select语句，通常与通道联用，这里不做介绍。

### if语句

if语法格式如下：

```go
if [布尔表达式] {
   // do something
}
```

if ... else ：

```go
if [布尔表达式] {
   // do something
} else {
   // do something
}
```

else if：

```go
if [布尔表达式] {
   // do something
} else if [布尔表达式] {
   // do something
} else {
   // do something
}
```

示例代码：
```go
package main

import "fmt"

func main() {	
	var grade = 70
	if grade >= 90 {
		fmt.Println("A" )
	} else if grade < 90 && grade >= 80 {
		fmt.Println("B" )
	} else if grade < 80 && grade > 60 {
		fmt.Println("C" )
	} else {
		fmt.Println("D" )
	}
}
```
### switch 语句
语法格式：
```go
switch var1 {
    case cond1:
        // do something
    case cond2:
        // do something
    default:
        // do something：条件都不满足时执行
}
```

另外，添加 fallthrough 会强制执行后面的 case 语句，不管下一条case语句是否为true。

示例代码：
```go
package main

import "fmt"

func main() {
	var grade = "B"
	switch grade {
	case "A":
		fmt.Println("优秀")
	case "B":
		fmt.Println("良好")
		fallthrough
	case "C":
		fmt.Println("中等")
	default:
		fmt.Println("不及格")
	}
}
```
执行结果：
```go
良好
中等
```

## 循环语句

下面介绍几种循环语句：

### for循环：使用分号

```go
package main

import "fmt"

func main() {
	sum := 0
	
    for i := 1; i < 5; i++ {
		sum += i
	}
	fmt.Println(sum) // 10 (1+2+3+4)
}
```

### 实现while效果

```go
package main

import "fmt"

func main() {
	sum := 0
	n := 0
    
	for n < 5 {
		sum += n
		n += 1
	}
	fmt.Println(sum) // 10 (1+2+3+4)
}
```

### 死循环

```go
package main

import "fmt"

func main() {
	sum := 0
	for {
		sum++		
	}
	fmt.Println(sum)
}
```

### for range 遍历

```go
package main

import "fmt"

func main() {	
	strings := []string{"hello", "world"}
	for index, str := range strings {
		fmt.Println(index, str)
	}
}
```

执行结果：

```go
0 hello
1 world
```



### 退出循环

- continue：结束当前迭代，进入下一次迭代

- break：结束当前for循环

```go
package main

import "fmt"

func main() {
	sum := 0
	for {
		sum++		
		if sum%2 != 0 {
			fmt.Println(sum)
			continue
		}
		if sum >= 10 {
			break
		}
	}
	fmt.Println(sum)
}
```

执行结果：

```go
1
3
5
7
9
10
```

也可以通过标记退出循环：

```go
package main

import "fmt"

func main() {
	sum := 0
	n := 0
	LOOP: for n <= 10 {
		if n == 8 {
			break LOOP
		}
		sum += n
		n++
	}
	fmt.Println(sum) // 28 (0+1+2+3+4+5+6+7)
}
```

### goto语句

```go
package main

import "fmt"

func main() {
	sum := 0
	sum2 := 0
	n := 0
    
	LOOP: for n <= 10 {
		if n%2 == 0 {
			sum += n
			n++
			goto LOOP
		}
		sum2 += n
		n++
	}
	fmt.Println(sum) // 30 (0+2+4+6+8+10)
	fmt.Println(sum2) // 25 (1+3+5+7+9)
}
```





