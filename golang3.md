## slice扩容规则

这一次终于搞清楚了，看 https://juejin.im/post/6844903811429957646#heading-0 的 “append 到底做了什么”

首先网上常规说的，确实不准确，确实有内存对齐这一步。

#### 基本规则

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片的长度小于 1024 就会将容量翻倍；
3. 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

#### 讲解

```
s := []int{1,2}
s = append(s,4,5,6)
fmt.Printf("%d %d",len(s),cap(s))
```

例如，现在的 slice cap是2，你一次性，注意是一次性 append 了3个数，那么你不能看成是 append 了3次，而是 append 一次，但是 append 了3个数。

所以，你新申请的 cap 就是5，原来2，现在 append 了3个，所以是5。

所以 `func growslice(et *_type, old slice, cap int) slice {}` 的第三个参数就是5。

当前切片的长度还是2，你说是2x2=4大还是5大，现在最起码要5，肯定不能是4，所以还是要选5。

至于剩下的两点很容易理解。

#### 内存对齐

确实是要内存对齐的，根据 slice 的类型，进行内存对齐，size 参数就是前面求出来的基本规则的新的 cap，而roundupsize的返回值才是最终的 cap。

```
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
			return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
		} else {
			return uintptr(class_to_size[size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return round(size, _PageSize)
}
```

## size_to_class8 （size_to_class128） 和 class_to_size

#### class_to_size

代表的是一个对象的大小，里面的每个元素都代表了一个对象的大小。

```
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

例如，如果你是 class_to_size[3] = 32，代表你现在的 obj（对象）的大小是32字节。

#### size_to_class8 和 size_to_class128 

通常情况下，我们在申请一块内存的时候，都会给出新的内存的长度，也就是 size，但是考虑到内存对齐的要求，我们通常不能直接使用这个 size。

所以这里要进行转换，所以 size_to_class8 和 size_to_class128 是为了求出 size 在 class_to_size 中的索引。

也就是说，size_to_class8 和 size_to_class128 只是一个索引，这个索引最终还要套用到 class_to_size 上面，class_to_size 最终求出来的就是一个 obj 的大小。

## 内存管理

通过  [https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#712-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%BB%84%E4%BB%B6](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#712-内存管理组件) 最重要

加  https://juejin.im/post/6844903795739082760#heading-4 去理解 golang 的内存管理

 Go 语言的内存分配器包含**内存管理单元**、**线程缓存**、**中心缓存**和**页堆**几个重要组件。

## 内存管理单元（mspan）

#### 数据结构

```go
type mspan struct {
	next *mspan
	prev *mspan
	
	startAddr uintptr // 可用内存起始地址，mspan 本身并不是一个可用内存
	npages    uintptr // 页数
	
	allocCache uint64 // 位图
}
```

从数据结构，我们来讲讲内存管理单元 `mspan` 的结构，首先 mspan 可以用链表串起来。像下面这样：

![2020-02-29-15829868066485-mspan-and-linked-list](C:\Users\78478\Desktop\review\2020-02-29-15829868066485-mspan-and-linked-list.png)

其次，`mspan` 其实并不是真正的可用内存，`mspan` 要注意它的**管理**两个字，它是个管理结构，它的 `startAddr` 字段指向的才是真正的可用内存，该内存是个多个 `page`组成的内存（这个 page 不是操作系统的 内存页），大小 8KB。

![2020-02-29-15829868066492-mspan-and-pages](C:\Users\78478\Desktop\review\2020-02-29-15829868066492-mspan-and-pages.png)

而实际上 `page` 本身并不是很重要，实际上在使用的时候，是按照对象来使用的，例如一个可用内存被分成了3个对象（这时候你管它有多少 page，不重要，看下图的 `Object` 那就是对象），那么我需要记录这3个对象被分配了没有，所以需要一个位图来记录，而 `allocCache` 就是那个位图，上面记载了内存中哪些 obj（对象） 被分配了，哪些没有，用1和0来区分。如下：

所以，要记住一句话，**`mspan` 实际上是按照跨度类来分的**。

![2020-02-29-15829868066499-mspan-and-objects](C:\Users\78478\Desktop\review\2020-02-29-15829868066499-mspan-and-objects.png)

#### 如何提取空闲内存（对象）（这个最重要，要结合上面的结构图）

这里要说明的是，申请内存本质**就是获取 mspan 上空闲的跨度类对象**。

你想想嘛，我现在要申请一块内存，是不是为了一个对象去申请一块内存也就是new之类的，最终是不是落到了某一个 mspan 上面？具体现在是那一块 mspan，是不是根据你要申请的 size 来决定？然后最后确定是要哪块 mspan!

那这块 mspan 就会从 `allocCache` 上判断现在的 mspan 上面还有哪些空闲对象，然后分配给要申请的操作。

#### 跨度类

刚才上面，我们重点提到了每个 `mspan` 上其实是按对象来分的，这里重点说一下。

```go
type mspan struct {
	...
	spanclass   spanClass //跨度类
	...
}
```

所有的数据都会被预选计算好并存储在 [`runtime.class_to_size`](https://github.com/golang/go/blob/151ccd4bdb06d77e89f00b8172b70cfb2f49ca2b/src/runtime/sizeclasses.go#L83) 和 [`runtime.class_to_allocnpages`](https://github.com/golang/go/blob/151ccd4bdb06d77e89f00b8172b70cfb2f49ca2b/src/runtime/sizeclasses.go#L84) 等变量中（写死的）：

| class | bytes/obj | bytes/span | objects | tail waste | max waste |
| ----- | --------- | ---------- | ------- | ---------- | --------- |
| 1     | 8         | 8192       | 1024    | 0          | 87.50%    |
| 2     | 16        | 8192       | 512     | 0          | 43.75%    |
| 3     | 32        | 8192       | 256     | 0          | 46.88%    |
| 4     | 48        | 8192       | 170     | 32         | 31.52%    |
| 5     | 64        | 8192       | 128     | 0          | 23.44%    |
| 6     | 80        | 8192       | 102     | 32         | 19.07%    |
| …     | …         | …          | …       | …          | …         |
| 66    | 32768     | 32768      | 1       | 0          | 12.50%    |

也就是讲，一个 `mspan` 会在一开始就定好，你上面一个 obj 是多大，你一共可以存储多少个 obj，不会出现说，你存一个 8B 的 跨度类又可以存储一个 32B 的跨度类，所有的 obj 都是一样的，**但是却可以浪费**。

例如我现在要存储一个 33B 的 obj （实际要存储，不是指 `mspan` 中的跨度类），而你又没有刚好33B的跨度类，那只能浪费一点了，所以选择48的（肯定不能选32B，装不下，而64浪费太大）。所以相当于一个48B的跨度类只能装33B，实际上就是浪费了15B。

所以思考一件事请，如果现在最小的跨度类8B，每个跨度类其实就装了1B浪费是不是最大的？所以我们为什么说不要频繁new小对象，因为小对象的浪费确实是最大的，但是要注意这是指他们的大小没有刚刚好接近跨度类的大小。

![2020-02-29-15829868066504-mspan-max-waste-memory](C:\Users\78478\Desktop\review\2020-02-29-15829868066504-mspan-max-waste-memory.png)

## 线程缓存（mcache）

#### 数据结构

```go
type mcache struct {
    alloc [numSpanClasses]*mspan // 数组
}
```

从上面可以看出，`alloc` 是一个数组，里面存放着 mspan **数组**，真正被使用其实是从这里开始。上面讲的 mspan 的使用其实是 mspan 被使用了之后，如果提取里面的跨度类。

现在讲的是如何提取一个 mspan，概念的维度是不一样的。

![2020-02-29-15829868066512-mcache-and-mspans](C:\Users\78478\Desktop\review\2020-02-29-15829868066512-mcache-and-mspans.png)

#### 与工作线程 M 的关系

mcache 是 Go 语言中的线程缓存，它会与线程上的处理器一一绑定，主要用来缓存用户程序申请的微小对象。 

也因为 mcache 和线程是 1:1 关系，所以不存在锁的竞争关系。

#### 微分配器

线程缓存中还包含几个用于分配微对象的字段，下面的这三个字段组成了微对象分配器，专门为 16 字节以下的对象申请和管理内存：

```go
type mcache struct {
    alloc [numSpanClasses]*mspan
    
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr
}
```

 **微分配器**只会用于分配非指针类型的内存，上述三个字段中 `tiny` 会指向堆中的一篇内存，`tinyOffset` 是下一个空闲内存所在的偏移量，最后的 `local_tinyallocs` 会记录内存分配器中分配的对象个数。 

![](C:\Users\78478\Desktop\review\2020-02-29-15829868066543-tiny-allocator.png)

#### 使用

好了，现在我们可以大概连着 mspan 来讲一下到底是怎么使用这个内存管理（仅仅从 mcache 维度来讲）。

1. 我一个协程上的 g 想要申请内存，可以知道这个内存的大小 `size`。
2. 根据 `size` 可以算出（这边有个计算公式，暂不清楚计算的依据是什么）`mcache.alloc `数组的索引。
3. 根据 `mcache.alloc[spc]` 可以获取到里面的 `mspan`，好了到这一步一切就好办了，下面其实就是上面我们介绍的从 `mspan` 里面提取可用的跨度类对象，根据 `allocCache` 位图去判断还没有空闲对象。
4. 如果 mcache 中的 mspan 上面没有空闲对象，那么就去 mcentral 获取，mcentral 还没有，就去 mheap 获取，mheap 没有就会去操作系统申请一块内存。

## 中心缓存（mcentral）

#### 数据结构

```go
type mcentral struct {
	lock      mutex
	spanclass spanClass
    
    // 尚有空闲object的mspan链表
	nonempty  mSpanList
	empty     mSpanList
	nmalloc uint64
}
```

根据数据结构可以看出，mcentral 有两个 **mspan** **链表**，其中 `nonempty` 是包含空闲的跨度类的链表。

![](C:\Users\78478\Desktop\review\2020-02-29-15829868066519-mcentral-and-mspans.png)

#### 使用

先从 nonempty 当中获取到 mspan，然后也是根据位图去获取 mspan 里面的空闲对象。

然后尝试从 empty 里面获取。

如果没有则向堆申请。

## 页堆（mheap）

#### 数据结构

 Go 语言程序只会存在一个全局的结构，而堆上初始化的所有对象都由该结构体统一管理，该结构体中包含两组非常重要的字段： 其中一个是全局的中心缓存列表 `central`，另一个是管理堆区内存区域的 `arenas` 以及相关字段。 

```go
type mheap struct {
	lock mutex
	
	// spans: 指向mspans区域，用于映射mspan和page的关系
	spans []*mspan 
	
	// 指向bitmap首地址，bitmap是从高地址向低地址增长的
	bitmap uintptr 

	// 管理堆区内存
    arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	central [67*2]struct {
		// 管理 mcentral
		mcentral mcentral
		pad [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
	}
}
```

![](C:\Users\78478\Desktop\review\169755c8b91af501.png)

#### 使用

获取下一个空闲的内存空间，如果没有则向系统申请。

## 内存管理的对象

#### 对象大小

Go 语言的内存分配器会根据申请分配的内存大小选择不同的处理逻辑，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：

要注意的是，微对象（tiny）不包括 16B，16B是小对象，而且微对象的范围很小，而小对象的范围很大，到 32KB，而不是 32B。

|  类别  |     大小      |
| :----: | :-----------: |
| 微对象 |  `(0, 16B)`   |
| 小对象 | `[16B, 32KB]` |
| 大对象 | `(32KB, +∞)`  |

#### 微对象（tiny）

1. 先考虑用 mcache 上的 tiny 指针来获取内存，tiny 指针指向一块 16B 的堆内存。
2. 如果 tiny 没有则从 mcache 上的 mspan 数组获取 mspan，然后就是根据位图获取空闲对象。
3. 还没有就向 mcentral 获取，而 mcentral 没有就会自己向 mheap 获取。

#### 小对象（small）

1. 确定分配对象的大小，以及跨度类。
2. 尝试从 mcache 的数组中获取 mspan，根据位图获取分配对象。
3. 尝试向 mcentral 或者 mheap 获取空闲内存。

#### 大对象

直接从 mheap 上获取内存。

## 假如你发现你写的go程序占用cpu100%，那你要怎么解决

大杀器 https://segmentfault.com/a/1190000016412013

用 pprof 尝试 cpu和内存占用，然后定位到某一个函数，然后 review 代码。

```
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // 引入这个官方包
    "github.com/EDDYCJY/go-pprof-example/data"
)

func main() {
    go func() {
        for {
            log.Println(data.Add("https://github.com/EDDYCJY"))
        }
    }()

    http.ListenAndServe("0.0.0.0:6060", nil) // 监听 6060 这个端口
}
```

命令的使用

```
// 进入交互式终端
go tool pprof http://localhost:6060/debug/pprof/profile

// 查看 CPU 占用较高的调用
top

// 查看问题具体在代码的哪一个位置，Eat 是具体函数
list 函数名
```

#### 内存

如果是内存的问题也是一样的，只不过第一个命令改成  `go tool pprof http://localhost:6060/debug/pprof/heap`  （最后变成 heap），然后 top，看看是哪个函数，然后 list 函数名

## 最基本的生产者消费者模型

```
package main

// 无缓冲的channel

import (
	"fmt"
	"time"
)

func produce(ch chan<- int) {
	for i := 0; i < 10; i++ {
		ch <- i
		fmt.Println("Send:", i)
	}
}

func consumer(ch <-chan int) {
	for i := 0; i < 10; i++ {
		v := <-ch
		fmt.Println("Receive:", v)
	}
}

// 因为channel没有缓冲区，所以当生产者给channel赋值后，
// 生产者线程会阻塞，直到消费者线程将数据从channel中取出
// 消费者第一次将数据取出后，进行下一次循环时，消费者的线程
// 也会阻塞，因为生产者还没有将数据存入，这时程序会去执行
// 生产者的线程。程序就这样在消费者和生产者两个线程间不断切换，直到循环结束。
func main() {
	ch := make(chan int)
	go produce(ch)
	go consumer(ch)
	time.Sleep(1 * time.Second)
}
```

有缓冲

```
package main

// 带缓冲区的channel

import (
	"fmt"
	"time"
)

func produce(ch chan<- int) {
	for i := 0; i < 10; i++ {
		ch <- i
		fmt.Println("Send:", i)
	}
}

func consumer(ch <-chan int) {
	for i := 0; i < 10; i++ {
		v := <-ch
		fmt.Println("Receive:", v)
	}
}

func main() {
	ch := make(chan int, 10)
	go produce(ch)
	go consumer(ch)
	time.Sleep(1 * time.Second)
}
```

## 处理输入

第一种方法，用 bufio，但是这种方法不能同时和 fmt.Scan 一起使用，原因未知，所以最好还是使用原生的 fmt.Scanf

```
package main

import (
	"bufio"
	"fmt"
	"os"
)

// 得到一行
func ScanLine() {
	reader := bufio.NewReader(os.Stdin)
	input, _, err := reader.ReadLine()
	if err != nil {
		panic(err)
	}
	fmt.Println(string(input))
}

func main() {
	ScanLine()
}

```

注意，其实 fms.Scanf 有注释，使用 %c 可以获取空格和换行符，但是必须是 byte 类型，如果你用 string 类型，是无法获取输入的

```
package main

import "fmt"

// 得到一行
//1
//i love you

func main() {
	var in byte // 关键类型
	var bytes []byte
	for {
		// 关键格式化占位符 %c
		fmt.Scanf("%c", &in)
		if in != '\n' {
			bytes = append(bytes, in)
		} else {
			break
		}
	}
	fmt.Println(string(bytes))
}

```

如果一定要使用 bufio，前面必须使用 fmt.Scanln，而不是 fmt.Scan，两者完全不同

```
package main

import (
	"bufio"
	"fmt"
	"os"
)

// 得到一行
//2
//i love you
func main() {
	reader := bufio.NewReader(os.Stdin)
	var num int
	fmt.Scanln(&num) // 如果使用 fmt.Scan 下面就会获取到 '\n'，而不是正常输入了
	fmt.Println(num)
	text, err := reader.ReadString('\n')
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(text)
}

```

柳生的方法

```
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    //整行读取
    scanner := bufio.NewScanner(os.Stdin)
    for scanner.Scan() { //扫描标准输入
        line := scanner.Text() //将标准输入的文本
        if line == "" {
            break
        }
        fmt.Println(line)
    }

    //fmt.Scan(&num)，这种遇到空格就会返回了，无法整行读取
}

```

