# Go语言基础语法（二）：函数
函数是一等（first-class）公民，可用来封装代码。在[Go语言基础语法（一）](https://blog.csdn.net/u010698107/article/details/119741912)中介绍了函数也是一种数据类型，函数的值也可以在其他函数间传递、赋予变量、做类型判断和转换等。下面来介绍Go语言中的函数定义和使用方法。

<!--more-->

## 普通函数声明与使用

下面先来介绍函数的简单使用方法。

函数定义语法：

```go
func function_name( parameter-list ) ( return-types ) {
   // 函数体
}
```

Go函数使用 `func` 关键字进行声明，输入参数和返回值都是可选的，可以没有参数，也可以没有返回值，函数体实现函数的功能逻辑。

除法运算例子：

```go
package main

import (
	"fmt"
	"errors"
)

func add(x int, y int) (float64, error) {	
	if y == 0 {
		return 0, errors.New("can't divide by zero!!")
	}
	res := float64(x) / float64(y)	
	return res, nil
}

func main() {
	value1 := 3
	value2 := 2
    value3 := 0
	res, err := add(value1, value2)
	fmt.Printf("%d / %d = %f (error: %v)\n", value1, value2, res, err)
	res, err = add(value1, value3)
	fmt.Printf("%d / %d = %f (error: %v)\n", value1, value3, res, err)
}

```

执行结果：

```go
3 / 2 = 1.500000 (error: <nil>)
3 / 0 = 0.000000 (error: can't divide by zero!!)
```

## 函数类型

前面说了函数也是一种数据类型，函数类型的声明语法如下：

```go
type function_name func(parameter-list) (return-types)
```

函数类型的函数签名（参数列表和结果列表）方法与函数声明一致，只要两个函数的函数签名一致（元素顺序和类型相同），它们就是相同的函数类型。

在前面除法运算例子中声明一个名为calculate的函数类型：

```go\
type calculate func(x int, y int) (float64, error)
```

函数签名和add函数一样，所以add和calculate是相同的函数类型。

```go
var cal calculate
cal = add
res, err = cal(3,2)
fmt.Printf("The result: %f (error: %v)\n", res, err)
```

执行结果：

```go
The result: 1.500000 (error: <nil>)
```

## 高阶函数

高阶函数和普通函数的区别在于高阶函数的形参或者返回参数列表中存在函数类型，也就是接收函数作为参数输入或者返回一个函数。

下面使用高阶函数实现加减乘除运算。

```go
package main

import (
	"errors"
	"fmt"
)

type operate func(x, y int) int

func calculate(x int, y int, op operate) (int, error) {
	if op == nil {
		return 0, errors.New("invalid operation")
	}
	return op(x, y), nil
}

func add(x, y int) int {
	return x + y
}

func sub(x, y int) int {
	return x - y
}

func multiply(x, y int) int {
	return x * y	
}

func divide(x, y int) int {
	return x / y	
}

func main() {
	x, y := 36, 6
    
	result, _ := calculate(x, y, add)
	fmt.Println("The result: ",result)

	result, _ = calculate(x, y, sub)
	fmt.Println("The result: ",result)

	result, _ = calculate(x, y, multiply)
	fmt.Println("The result: ",result)

	result, _ = calculate(x, y, divide)
	fmt.Println("The result: ",result)
	
	result, _ = calculate(x, y, nil)
	fmt.Println("The result: ",result)	
}

```

执行结果：

```go
The result:  42
The result:  30
The result:  216
The result:  6
The result:  0
```

## 闭包函数

闭包函数是引用了自由变量的代码块，闭包可以作为函数对象或者匿名函数。下面用闭包实现计算一个数的 n 次幂：

```go
package main

import (
	"fmt"
)

type exponent func(uint64) uint64

func nth_power(exp uint64) exponent {
	return func(base uint64) uint64 {
		result := uint64(1)
		for i := exp ; i > 0; i >>= 1 {
			if i&1 != 0 {
				result *= base
			}
			base *= base
		}
		return result
	}
}

func main() {
	square := nth_power(2) // 平方
	cube := nth_power(3) // 立方	
	fmt.Println(square(5))
	fmt.Println(cube(5))
}

```

执行结果：

```go
25
125
```

从代码中可以看出闭包返回的是一个函数，不是具体的值，使用闭包可以根据需要生成功能不同的函数。

## 参数传递

我在[Python函数的参数类型](https://blog.csdn.net/u010698107/article/details/118280135)中介绍过Python函数中的参数传递，Python中的参数传递属于对象的引用传递，而Go语言中均为**值传递**。

```go
package main

import "fmt"


func modifyArray(a [3]int) [3]int {
	a[1] = 0
	return a
}

func modifySlice(a []int) []int {
	a[1] = 0
	return a
}

func main() {
	l1 := [3]int{1, 2, 3}
	fmt.Println("value of l1:  ",l1)
	fmt.Printf("address of l1: %p\n",&l1)

	l2 := modifyArray(l1)
	fmt.Printf("address of l2: %p\n",&l2)
	fmt.Println("value of l1:  ",l1)
	fmt.Println("value of l2:  ",l2)

	slice1 := []int{1, 2, 3}
	fmt.Println("value of slice1:  ",slice1)
	fmt.Printf("address of slice1: %p\n",&slice1)

	slice2 := modifySlice(slice1)
	fmt.Printf("address of slice2: %p\n",&slice2)
	fmt.Println("value of slice1:  ",slice1)
	fmt.Println("value of slice2:  ",slice2)

	slice2[2] = 6
	fmt.Println("value of slice1:  ",slice1)
	fmt.Println("value of slice2:  ",slice2)

}

```

执行结果：

```go
value of l1:   [1 2 3]
address of l1: 0xc000016198
address of l2: 0xc0000161c8
value of l1:   [1 2 3]
value of l2:   [1 0 3]
value of slice1:   [1 2 3]
address of slice1: 0xc000004078
address of slice2: 0xc0000040a8
value of slice1:   [1 0 3]
value of slice2:   [1 0 3]
value of slice1:   [1 0 6]
value of slice2:   [1 0 6]
```

由于数组是值类型，传给函数的参数值都会被复制，所以使用modifyArray对原数组进行修改时原数组不会改变，只是修改了它的副本而已，这和Python中的list不一样。

而对于引用类型，比如：切片、字典、通道，使用上面代码中的方式修改时，不会拷贝它们引用的底层数据，只是进行了浅表复制。所以上面例子中的原切片slice1也会跟着改变。

对于引用类型可以使用copy函数进行拷贝：

```go
package main

import "fmt"

func main() {
	slice1 = []int{1, 2, 3}
	slice3 := make([]int, len(slice1))
	copy(slice3, slice1)
	slice3[1] = 6
	fmt.Printf("address of slice1: %p\n",&slice1)
	fmt.Printf("address of slice3: %p\n",&slice3)
	fmt.Println("value of slice1:  ",slice1)
	fmt.Println("value of slice3:  ",slice3)
}
```

执行结果：

```go
address of slice1: 0xc000098060
address of slice3: 0xc000098108
value of slice1:   [1 2 3]
value of slice3:   [1 6 3]
```





