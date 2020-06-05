# a-tour-of-go

## 接口

```Go
package main

import (
	"fmt"
	"math"
)

type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}
	a = f  // a MyFloat 实现了 Abser
	fmt.Println(a.Abs())
	fmt.Printf("%v  %T\n",a,a)
	a = &v // a *Vertex 实现了 Abser
	fmt.Println(a.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

```

### 错误

`error`类型是一个内建接口

```Go
type error interface{
    Error() string
}
```



### type Reader

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

Reader 接口包装了基本的 Read 方法。

Read 将 len(p) 个字节读取到 p 中。它返回读取的字节数 n（0 <= n <= len(p)） 以及任何遇到的错误。即使 Read 返回的 n < len(p)，它也会在调用过程中使用 p 的全部作为暂存空间。若一些数据可用但不到 len(p) 个字节，Read 会照例返回可用的东西， 而不是等待更多。

当 Read 在成功读取 n > 0 个字节后遇到一个错误或 EOF 情况，它就会返回读取的字节数。 它会从相同的调用中返回（非nil的）错误或从随后的调用中返回错误（和 n == 0）。 这种一般情况的一个例子就是 Reader 在输入流结束时会返回一个非零的字节数， 可能的返回不是 err == EOF 就是 err == nil。无论如何，下一个 Read 都应当返回 0, EOF。

调用者应当总在考虑到错误 err 前处理 n > 0 的字节。这样做可以在读取一些字节， 以及允许的 EOF 行为后正确地处理I/O错误。

Read 的实现在 len(p) == 0 以外的情况下会阻止返回零字节的计数和 nil 错误， 调用者应将返回 0 和 nil 视作什么也没有发生；特别是它并不表示 EOF。

实现必须不保留 p。



### Reader

`io.Reader` 接口有一个 `Read` 方法：

```
func (T) Read(b []byte) (n int, err error)
```

`Read` 用数据填充给定的字节切片并返回填充的字节数和错误值。在遇到数据流的结尾时，它会返回一个 `io.EOF` 错误。



```Go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("Hello, Reader!")
	b := make([]byte, 8)
	for {
		n, err := r.Read(b) //一次读取把buffer 装满
		fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
		fmt.Printf("b[:n] = %q\n", b[:n])
		if err == io.EOF {
			break
		}
	}
}

```



### 产生一个ascii字符A的无限流

```Go
func (mr MyReader) Read(b []byte) (int, error) {

	b[0]='A'

	return 1,nil

}
```





### rot13Reader

有种常见的模式是一个 [`io.Reader`](https://go-zh.org/pkg/io/#Reader) 包装另一个 `io.Reader`，然后通过某种方式修改其数据流。

例如，[`gzip.NewReader`](https://go-zh.org/pkg/compress/gzip/#NewReader) 函数接受一个 `io.Reader`（已压缩的数据流）并返回一个同样实现了 `io.Reader` 的 `*gzip.Reader`（解压后的数据流）。

编写一个实现了 `io.Reader` 并从另一个 `io.Reader` 中读取数据的 `rot13Reader`，通过应用 [rot13](http://en.wikipedia.org/wiki/ROT13) 代换密码对数据流进行修改。

`rot13Reader` 类型已经提供。实现 `Read` 方法以满足 `io.Reader`。



```Go
package main

import (
	"io"
	"os"
	"strings"
)

type rot13Reader struct {
	r io.Reader
}

func rot13(b byte) byte {
    switch {
    case 'A' <= b && b <= 'M':
        b = b + 13
    case 'M' < b && b <= 'Z':
        b = b - 13
    case 'a' <= b && b <= 'm':
        b = b + 13
    case 'm' < b && b <= 'z':
        b = b - 13
    }
    return b
}

func (rot rot13Reader) Read(b []byte) (int, error) {
	n, err := rot.r.Read(b)
	for i := range b {
		b[i] = rot13(b[i])
	}
	return n, err
}



func main() {
	s := strings.NewReader("Lbh penpxrq gur pbqr!")
	r := rot13Reader{s}
	io.Copy(os.Stdout, &r)
}

```



## 并发

### Go程

> Do	not	communicate	by	sharing	memory;	instead,	share	memory	by	communicating
>
> 
>
> 不要通过共享内存来通信，而应通过通信来共享内存。

这种方法意义深远。例如，引用计数通过为整数变量添加互斥锁来很好地实现。	但作为一种 高级方法，通过信道来控制访问能够让你写出更简洁，正确的程序。

在函数或方法前添加	go	关键字能够在新的	Go	程中调用它。当调用完成后，	该	Go	程也会安 静地退出。（效果有点像	Unix	Shell	中的	&	符号，它能让命令在后台运行。）

```go
go	list.Sort()		//	并发运行	list.Sort，无需等它结束。
```



### 信道

信道是带有类型的管道

```go
ch <- v //将v发送至信道ch
v := <-ch //从ch接收值并赋予v
```

箭头就是数据流的方向

和映射与切片一样，信道在使用前必须创建

其结果值充当了对底层数据结构的引用

若提供了一个可选的整数形参，它就会为该信道设置缓冲区大小。默认值是零，表示不带缓 冲的或同步的信道

```go
ch := make(chan int)
```

默认情况下，发送和接收操作在另一端准备好之前都会阻塞。这使得 Go 程可以在没有显式的锁或竞态变量的情况下进行同步。

```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 将和送入 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从 c 中接收

	fmt.Println(x, y, x+y)
}

```

```go
-5 17 12
```

### 带缓冲的信道

将缓冲长度提供给make来初始化一个带缓冲的信道

```go
ch := make(chan int, 100)
```

仅当信道的缓冲区填满后，向其发送数据时才会阻塞

无缓冲信道在通信时会同步交换数据，它能确保（两个	Go	程的）计算处于确定状

```go
c	:=	make(chan	int)//	分配一个信道 
				//	在	Go	程中启动排序。当它完成后，在信道上发送信号。 
go	func()	{				
    list.Sort()				
    c	<-	1		//	发送信号，什么值无所谓。 
}() 
doSomethingForAWhile() 
<-c			//	等待排序结束，丢弃发来的值。
```

接收者在收到数据前会一直阻塞。若信道是不带缓冲的，那么在接收者收到值前，	发送者会 一直阻塞；若信道是带缓冲的，则发送者仅在值被复制到缓冲区前阻塞；	若缓冲区已满，发送者会一直等待直到某个接收者取出一个值为止。

带缓冲的信道可被用作信号量，例如限制吞吐量。在此例中，进入的请求会被传递给 handle，它从信道中接收值，处理请求后将值发回该信道中，以便让该	“信号量”	准备迎接下 一次请求。信道缓冲区的容量决定了同时调用	process	的数量上限，因此我们在初始化时首 先要填充至它的容量上限。

```go


```



### close和range





