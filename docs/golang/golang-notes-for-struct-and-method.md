# Go语言基础语法（三）：结构体及方法
结构体类型可以用来保存不同类型的数据，也可以通过方法的形式来声明它的行为。本文将介绍go语言中的结构体和方法，以及“继承”的实现方法。

<!--more-->

## 结构体类型

结构体类型（struct）在go语言中具有重要地位，它是实现go语言面向对象编程的重要工具。go语言中没有类的概念，可以使用结构体实现类似的功能，传统的OOP（Object-Oriented Programming）思想中的继承在go中可以通过嵌入字段的方式实现。

结构体的声明与定义：

```go
// 使用关键字 type 和 struct 定义名字为Person结构体
type Robot struct {
	name string
	height int
}
```

初始化及赋值：

```go
// 通过var声明
var r1 Robot
r1.name = "Optimus Prime"

// 字面量直接赋值
r2 := Robot{name: "Optimus Prime"}
r3 := Robot{"Optimus Prime", 100} //如果不加字段名，值必须按定义顺序给出

// new 函数
r4 := new(Robot)
r4.name = "Optimus Prime"
//或者
r5 := &Robot{}
r5.name = r1.name
```

## 方法

go语言中的函数和方法是有区别的，方法必须有名字，必须隶属于某一个类型，这个类型通过方法声明中的接收者（receiver）声明定义。

接收者声明位于关键字func和方法名称之间的圆括号中，必须包含确切的名称和类型字面量。

- 类型就是当前方法所属的类型
- 名称用于当前方法中引用它所属类型的值

```go
package main

import "fmt"

type Robot struct {
	name string
	height int
}

func (r Robot) String() string{	
	return fmt.Sprintf("name: %s, height: %d",r.name, r.height)
}

func main() {
    r1 := Robot{name: "Optimus Prime", height: 100}
	fmt.Println(r1)  // 结果： name: Optimus Prime, height: 100
}

```

从String()方法的接收者声明可以看出它隶属于Robot类型，接收者名称为r。

## 结构体内嵌：“继承”与“重写”

Go 语言中没有继承的概念，具体原因和理念可参考官网：[Why is there no type inheritance?](https://golang.org/doc/faq#inheritance)

go语言可以通过嵌入字段来实现类似继承的效果，来看下面的代码：

```go
package main

import "fmt"

type Skills struct {
	speak string 
}

func (s Skills) Speak()  {
	fmt.Println(s.speak)	
}

type Robot struct {
	name string // 姓名
	height int // 身高
	Skills
}

func main() {
	skill := Skills{speak: "hello !"}
	skill.Speak()
	
	robot := Robot{
		name: "Optimus Prime",
		Skills: skill,
	}
	robot.Speak()
}

```

嵌入字段的方法集合会被合并到被嵌入类型的方法集合中。上面代码中，`robot.Speak()` 会调用嵌入字段Skills的Speak()方法，类似于继承了Skills的Speak()方法。执行结果如下：

```go
hello !
hello !
```

下面添加一个Robot类型的Speak()方法：

```go
func (r Robot) Speak() {	
	fmt.Printf("My name is %s, ",r.name)
	r.Skills.Speak()
}
```

那么再次执行，会执行哪个Speak()方法呢？答案是Robot类型的Speak()方法，嵌入字段Skills的Speak()方法被“屏蔽”了，也就是说，被嵌入类型的方法覆盖了嵌入字段的同名方法，这与方法重写类似。

执行结果：

```go
hello !
My name is Optimus Prime, hello !
```

可以通过链式的选择表达式，选择到嵌入字段的字段或方法，`r.Skills.Speak()` 就调用了嵌入字段Skills的Speak()方法。

## 小结

需要注意的是Go 语言虽然支持面向对象编程，但是它没有继承的概念，可以通过嵌入字段的方式来实现类似继承的功能，这种组合方法相比多重继承更加简洁。







