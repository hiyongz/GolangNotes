# Go语言中的字符串拼接方法介绍
本文介绍Go语言中的string类型、strings包和bytes.Buffer类型，介绍几种字符串拼接方法。

<!--more-->


## string类型

string类型的值可以拆分为一个包含多个字符（rune类型）的序列，也可以被拆分为一个包含多个字节 (byte类型) 的序列。其中一个rune类型值代表一个Unicode 字符，一个rune类型值占用四个字节，底层就是一个 UTF-8 编码值，它其实是int32类型的一个别名类型。

```go
package main

import (
	"fmt"
)

func main() {
	str := "你好world"
	fmt.Printf("The string: %q\n", str)
	fmt.Printf("runes(char): %q\n", []rune(str))
	fmt.Printf("runes(hex): %x\n", []rune(str))
	fmt.Printf("bytes(hex): [% x]\n", []byte(str))
}
```

执行结果：

```go
The string: "你好world"
runes(char): ['你' '好' 'w' 'o' 'r' 'l' 'd']
runes(hex): [4f60 597d 77 6f 72 6c 64]
bytes(hex): e4 bd a0 e5 a5 bd 77 6f 72 6c 64
```

可以看到，英文字符使用一个字节，而中文字符需要三个字节。下面使用 `for range` 语句对上面的字符串进行遍历：

```go
for index, value := range str {
    fmt.Printf("%d: %q [% x]\n", index, value, []byte(string(value)))
}
```

执行结果如下：

```go
0: '你' [e4 bd a0]
3: '好' [e5 a5 bd]
6: 'w' [77]
7: 'o' [6f]
8: 'r' [72]
9: 'l' [6c]
10: 'd' [64]
```

index索引值不是0-6，相邻Unicode 字符的索引值不一定是连续的，因为中文字符占用了3个字节，宽度为3。



## strings包



### strings.Builder类型

strings.Builder的优势主要体现在字符串拼接上，相比使用`+`拼接，效率更高。

- strings.Builder已存在的值不可改变，只能重置（Reset()方法）或者拼接更多的内容。
- 一旦调用了Builder值，就不能再以任何方式对其进行复制，比如函数间值传递、通道传递值、把值赋予变量等。
- 在进行拼接时，Builder值会自动地对自身的内容容器进行扩容，也可以使用Grow方法进行手动扩容。

```go
package main

import (
	"fmt"
	"strings"
)
func main() {
	var builder1 strings.Builder
	builder1.WriteString("hello")
	builder1.WriteByte(' ')
	builder1.WriteString("world")
	builder1.Write([]byte{' ', '!'})

	fmt.Println(builder1.String())	

	f1 := func(b strings.Builder) {
		// b.WriteString("world !")  //会报错
	}
	f1(builder1)

	builder1.Reset()
	fmt.Printf("The length 0f builder1: %d\n", builder1.Len())

}
```
执行结果：
```go
hello world !
The length 0f builder1: 0
```

### strings.Reader类型

strings.Reader类型可以用于高效地读取字符串，它通过使用**已读计数**机制来实现了高效读取，已读计数保存了已读取的字节数，也代表了下一次读取的起始索引位置。

```go
package main

import (
	"fmt"
	"strings"
)
func main() {	
	reader1 := strings.NewReader("hello world!")
	buf1 := make([]byte, 6)
    fmt.Printf("reading index: %d\n", reader1.Size()-int64(reader1.Len()))
	
    reader1.Read(buf1)
	fmt.Println(string(buf1))
    fmt.Printf("reading index: %d\n", reader1.Size()-int64(reader1.Len()))
    
	reader1.Read(buf1)
	fmt.Println(string(buf1))
    fmt.Printf("reading index: %d\n", reader1.Size()-int64(reader1.Len()))
}
```

执行结果：

```go
reading index: 0
hello
reading index: 6
world!
reading index: 12
```

可以看到，每读取一次之后，已读计数就会增加。

strings包的**ReadAt方法**不会依据已读计数进行读取，也不会更新已读计数。它可以根据偏移量来自由地读取Reader值中的内容。

```go
package main

import (
	"fmt"
	"strings"
)
func main() {
    reader1 := strings.NewReader("hello world!")
    buf1 := make([]byte, 6)
	offset1 := int64(6)
	n, _ := reader1.ReadAt(buf1, offset1)	
	fmt.Println(string(buf2))
}
```

执行结果：

```go
world!
```

也可以使用**Seek方法**来指定下一次读取的起始索引位置。

```go
package main

import (
	"fmt"
	"strings"
    "io"
)
func main() {
    reader1 := strings.NewReader("hello world!")
    buf1 := make([]byte, 6)
	offset1 := int64(6)
	readingIndex, _ := reader2.Seek(offset1, io.SeekCurrent)
	fmt.Printf("reading index: %d\n", readingIndex)

	reader1.Read(buf1)
	fmt.Printf("reading index: %d\n", reader1.Size()-int64(reader1.Len()))
	fmt.Println(string(buf1))
}
```

执行结果：

```go
reading index: 6
reading index: 12
world!
```

## bytes.Buffer

bytes包和strings包类似，strings包主要面向的是 Unicode 字符和经过 UTF-8 编码的字符串，而bytes包面对的则主要是字节和字节切片，主要作为字节序列的缓冲区。bytes.Buffer数据的读写都使用到了已读计数。

bytes.Buffer具有读和写功能，下面分别介绍他们的简单使用方法。

### bytes.Buffer：写数据

和strings.Builder一样，bytes.Buffer可以用于拼接字符串，strings.Builder也会自动对内容容器进行扩容。请看下面的代码：

```go
package main

import (
	"bytes"
	"fmt"
)

func DemoBytes() {
	var buffer bytes.Buffer
	buffer.WriteString("hello ")
	buffer.WriteString("world !")
	fmt.Println(buffer.String())
}
```

执行结果：

```go
hello world !
```

### bytes.Buffer：读数据

bytes.Buffer读数据也使用了已读计数，需要注意的是，进行读取操作后，Len方法返回的是未读内容的长度。下面直接来看代码：

```go
package main

import (
	"bytes"
	"fmt"
)

func DemoBytes() {
	var buffer bytes.Buffer
	buffer.WriteString("hello ")
	buffer.WriteString("world !")
    
    p1 := make([]byte, 5)
	n, _ := buffer.Read(p1)
    
	fmt.Println(string(p1))
	fmt.Println(buffer.String())
    fmt.Printf("The length of buffer: %d\n", buffer.Len())
}
```

执行结果：

```go
hello
 world !
The length of buffer: 8
```



## 字符串拼接
简单了解了string类型、strings包和bytes.Buffer类型后，下面来介绍golang中的字符串拼接方法。

https://zhuanlan.zhihu.com/p/349672248

go test -bench=. -run=^BenchmarkDemoBytes$

### 直接相加

最简单的方法是直接相加，由于string类型的值是不可变的，进行字符串拼接时会生成新的字符串，将拼接的字符串依次拷贝到一个新的连续内存空间中。如果存在大量字符串拼接操作，使用这种方法非常消耗内存。

```go
package main

import (
	"bytes"
	"fmt"
	"time"
)

func main() {
	str1 := "hello "
	str2 := "world !"
    str3 := str1 + str2
    fmt.Println(str3)	
}
```



### strings.Builder

前面介绍了strings.Builder可以用于拼接字符串：

```go
var builder1 strings.Builder
builder1.WriteString("hello ")
builder1.WriteString("world !")
```



### strings.Join()

也可以使用strings.Join方法，其实Join()调用了WriteString方法；

```go
str1 := "hello "
str2 := "world !"
str3 := ""

str3 = strings.Join([]string{str3,str1},"")
str3 = strings.Join([]string{str3,str2},"")
```



### bytes.Buffer

bytes.Buffer也可以用于拼接：

```go\
var buffer bytes.Buffer

buffer.WriteString("hello ")
buffer.WriteString("world !")
```



### append方法

也可以使用Go内置函数append方法，用于拼接切片：

```go
package main

import (
	"fmt"
)

func DemoAppend(n int) {
	str1 := "hello "
	str2 := "world !"
	var str3 []byte

    str3 = append(str3, []byte(str1)...)
    str3 = append(str3, []byte(str2)...)
	fmt.Println(string(str3))
}
```

执行结果：

```go
hello world !
```

### fmt.Sprintf

fmt包中的Sprintf方法也可以用来拼接字符串：

```go
str1 := "hello "
str2 := "world !"
str3 := fmt.Sprintf("%s%s", str1, str2)
```

## 字符串拼接性能测试

下面来测试一下这6种方法的性能，编写测试源码文件strcat_test.go：

```go
package benchmark

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

func DemoBytesBuffer(n int) {
	var buffer bytes.Buffer

	for i := 0; i < n; i++ {
		buffer.WriteString("hello ")
		buffer.WriteString("world !")
	}
}

func DemoWriteString(n int) {
	var builder1 strings.Builder
	for i := 0; i < n; i++ {
		builder1.WriteString("hello ")
		builder1.WriteString("world !")
	}
}

func DemoStringsJoin(n int) {
	str1 := "hello "
	str2 := "world !"
	str3 := ""
	for i := 0; i < n; i++ {
		str3 = strings.Join([]string{str3, str1}, "")
		str3 = strings.Join([]string{str3, str2}, "")
	}

}

func DemoPlus(n int) {

	str1 := "hello "
	str2 := "world !"
	str3 := ""
	for i := 0; i < n; i++ {
		str3 += str1
		str3 += str2
	}
}

func DemoAppend(n int) {

	str1 := "hello "
	str2 := "world !"
	var str3 []byte
	for i := 0; i < n; i++ {
		str3 = append(str3, []byte(str1)...)
		str3 = append(str3, []byte(str2)...)
	}
}

func DemoSprintf(n int) {
	str1 := "hello "
	str2 := "world !"
	str3 := ""
	for i := 0; i < n; i++ {
		str3 = fmt.Sprintf("%s%s", str3, str1)
		str3 = fmt.Sprintf("%s%s", str3, str2)
	}
}

func BenchmarkBytesBuffer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoBytesBuffer(10000)
	}
}

func BenchmarkWriteString(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoWriteString(10000)
	}
}

func BenchmarkStringsJoin(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoStringsJoin(10000)
	}
}

func BenchmarkAppend(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoAppend(10000)
	}
}

func BenchmarkPlus(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoPlus(10000)
	}
}

func BenchmarkSprintf(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DemoSprintf(10000)
	}
}

```

执行性能测试：

```bash
$ go test -bench=. -run=^$
goos: windows
goarch: amd64
pkg: testGo/benchmark
cpu: Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz
BenchmarkBytesBuffer-8              3436            326846 ns/op
BenchmarkWriteString-8              4148            271453 ns/op
BenchmarkStringsJoin-8                 3         402266267 ns/op
BenchmarkAppend-8                   1923            618489 ns/op
BenchmarkPlus-8                        3         345087467 ns/op
BenchmarkSprintf-8                     2         628330850 ns/op
PASS
ok      testGo/benchmark        9.279s

```

通过平均耗时可以看到WriteString方法执行效率最高。Sprintf方法效率最低。

1. 我们看到Strings.Join方法效率也比较低，在上面的场景下它的效率比较低，它在合并已有字符串数组的场合效率是很高的。

2. 如果要连续拼接大量字符串推荐使用WriteString方法，如果是少量字符串拼接，也可以直接使用`+`。

3. append方法的效率也是很高的，它主要用于切片的拼接。

4. fmt.Sprintf方法虽然效率低，但在少量数据拼接中，如果你想拼接其它数据类型，使用它可以完美的解决：

   ```go
   name := "zhangsan"
   age := 20
   str4 := fmt.Sprintf("%s is %d years old", name, age)
   fmt.Println(str4)  // zhangsan is 20 years old
   ```

   













