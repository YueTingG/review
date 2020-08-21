## 格式化保留小数

```
 fmt.Sprintf("%.2f", **value**) 
```



## slice

### cap和len

 https://juejin.im/post/5ca2b75f51882543ea4b81c8#heading-7 

不是简单的取最小理论值（最小理论值就是旧的长度+新的元素个数，例如本来长度2，新增3个元素，最小理论值就为5），也不是2倍，1.25倍，而是取完上面说的几个值之后，再拿去内存对齐，对齐完取得的cap就是现在的容量。 

```go
func main() {
	s := make([]int, 3)
	fmt.Printf("%d %d %d", len(s), cap(s), s) // 3 3 [0 0 0]
}
```

```go
func main() {
	s := make([]int, 3, 5)
	fmt.Printf("%d %d %d", len(s), cap(s), s) // 3 5 [0 0 0]
}
```

初始化长度的时候，会顺便初始化0值

### slice和数组

golang的数组作为参数传递时是值传递，不是引用传递，跟c++不一样，还有赋值时也是值传递不是引用传递

```
var arr [2]
tmp := arr // 拷贝一份值，不是拷贝引用
arr[0] = 100
fmt.Println(tmp) // [0,0]
```

### 初始化的一些问题

slice初始化的时候

```
var arr []int
arr:=make([]int,5)
```

要注意采用第一种初始化方式，只能不断append进去元素，不能出现没有append进元素然后取元素的情况，例如

```
// 从来没有append进元素
fmt.Println(arr[0]) // 绝对报错
```

而第二种初始化方式，如果初始化长度不为0，则初始化的每个值都用零值，相当于里面有元素，可以用取元素的方式取值

## Map

### 初始化

map初始化一定要用make，不然直接用会爆错，而且make指定不指定长度（无法指定容量），都没事，本身就会扩容，不会说因为你指定了长度而影响它的实际长度

```go
func main() {
    // len(m)还是0，虽然初始化的时候指定了长度
    // 等价于 m := make(map[int]int)，不指定长度
	m := make(map[int]int, 3)
	fmt.Printf("%d %d", len(m), m) // 0 map[]
}
```

### struct as map value

```go
type node struct {
	index int
	freq  int // 频率
}

func main (
	test := map[int]node{
		0: {0, 0},
		1: {1, 1},
	}
	test[0].freq++ // Cannot assign to test[0].freq
    
    node, _ := test[0]
    node.index++ // 这里的node只是一个副本, 操作合法但是无法修改index的值,只修改了副本的值
)
```

**正确做法**

```
   tmp := test[1] // 区别语法, node, _ := test[0]
   tmp.freq = 2
   test[1] = tmp
```

**或者**

```
	test := map[int]*node{ // 定义成指针
		0: {0, 0},
		1: {1, 1},
	}
	test[0].freq++ // 合法操作	
	
	node, _ := test[0]
	node.index++ // 可以修改原值,因为副本是指针,有地址
```

## interface{}

interface{}类型什么时候会nil，必须是type和value都为nil的时候，才等于nil

```go
func main() {
	var i interface{}
	var p *int
	i = p
	fmt.Printf("%v\n", reflect.TypeOf(i)) // *int
	fmt.Printf("%t %v\n", i == nil, i)    // false <nil>
}
```

上面代码中，p是空指针，值是nil，但是它的类型本身是*int，不是nil，所以赋值给interface后，也不等于nil

```go
func main() {
	var i interface{}
	i = nil
	fmt.Printf("%v\n", reflect.TypeOf(i)) // <nil>
	fmt.Printf("%t %v\n", i == nil, i)    // true <nil>
}
```

nil本身的类型为nil，值也是nil，把它赋值给interface才是nil

## 面向对象

 在 Go 語言如何區分 `func (s *MyStruct)` 及 `func (s MyStruct)`，底下我們先來看看簡單的 Struct 例子，使用哪一种好。

**注意：使用func (s MyStruct) 是无法修改s内部的值，相当于拷贝传入**

```
package main

import "fmt"

type Cart struct {
	Name  string
	Price int
}

func (c Cart) GetPrice() {
	fmt.Println("price:", c.Price)
}

func (c Cart) UpdatePrice(price int) {
	c.Price = price
}

func main() {
	c := &Cart{"bage", 100}
	c.GetPrice()
	c.UpdatePrice(200)
	c.GetPrice()
}
```

结果：

```
price: 100
price: 100
// 你原本期望通过UpdatePrice函数来修改price的值，但是结果并没有修改
```

结论： 在開發團隊內，如果有人使用 Pointer 有人使用 Value 方式，這樣寫法不統一，造成維護效率非常低，所以官方建議，**全部使用 Pointer 方式是最好的寫法**。 

## 字符串

### 字符串切割

"  hello world!  "进行字符串切割后为["", "", "hello", "world!", "", ""] ,前后分别多了两个"",要注意。

### 字符串类型

自己看

```
type byte = uint8

type rune = int32
```

forrange是rune，取索引是byte

```
s := "hello"

for _, v := range s {
	fmt.Println(reflect.TypeOf(v)) // int32，也就是rune
}

fmt.Println(reflect.TypeOf(s[0])) // uint8，byte
```

### 字符串forrange顺序

字符串遍历的顺序是从前到后的，例如，[]byte("hello")，还有[]rune()也是一样的

```
s := []byte("hello")
fmt.Println(string(s[0])) // h，不是o
```

### len长度

 len函数获取到的长度并不是字符个数，而是字节个数 

```
a := "中国";
fmt.Println(len(a)) // 6，一个中文3个字节
```

### 字符类型II

#### utf8与golang

 ![img](https://user-gold-cdn.xitu.io/2019/3/7/16957e54fdb79b52?imageView2/0/w/1280/h/960/format/webp/ignore-error/1) 

**UTF规定：如果一个符号只占一个字节，那么这个8位字节的第一位就为0。如果为两个字节，那么规定第一个字节的前两位都为1，然后第一个字节的第三位为0，第二个字节的前两位为10，然后如果是三个字节的话，那么第一个字节的前三位为111，第四位为0，剩余的两个字节的前两位都为10。** （看上图）

所以，utf8能够实现变长就是这样，不是每个字符都是定长的，像“abc中国”前面的"abc"都是一个字节，汉字是三个字节。

> 而golang恰好是按照utf8来编码的，所以遵循上面的原则。

#### 字符串是一个字节切片数组

但是这又与**字符串是一个字节切片数组**会产生冲突（原谅我用"冲突"来描述），例如“abc中国”，对于前面三位，s[0],s[1],s[2]都能很好的表示"a","b","c"，但是s[3]却不是“中”，因为“中”是三个字节，所以s[3]是中的其中一个字节，用s[3:6]才能表示“中”。

#### Unicode与utf8

 Unicode定义了所有符号的二进制形式，也就是符号如何在计算机内部存储的，而且每个符号规定都必须使用两个字节来表示，也就是用16位二进制去代表一个符号，这样就导致了一个问题，英文编码的空间浪费，因为在ANSI中的符号都是一个字节来表示的，而使用了UNICODE编码就白白浪费了一个字节。 

 所以为了解决符号在网络中传输的浪费问题，就出现了UTF-8编码，Unicode transfer format -8 ，后面的8代表是以8位二进制为单位来传输符号的，但是这样又导致了一个问题，虽然UTF-8可以使用一个字节来表示ANSI下的符号，但是对于其它类似汉语的符号，得需要两个字节来表示，**所以计算机不知道如何去截取一个符号**，也就是一个符号对应的二进制的截取开始位置和截取结束位置，解决方法就是上面[utf8与golang](####utf8与golang)。

> Unicode是定长的，所以不存在识别问题，utf8是不定长的，所以有识别问题，但是它的定义解决了该问题。

#### []rune

[]rune是一个4个字节的Unicode，通常用于把utf8转成Unicode，方法是这样的，它先按照utf8的识别规则，识别每个字符（既能识别阿斯克码，又能识别汉字），然后把每个字符转成Unicode，这样所有的字符现在都是2个字节的了，再然后转成4个字节即可。

## 结构体能不能比较

t1 t2 是同一个struct的2个赋值相同的实例

他们本质就是结构体的一个“对象”，因为**成员变量带有了不能比较的成员**，所以只要写 == 就报错

指针可以

**结论：成员变量如果带了可以不可以比较的类型，那么不可以，否则可以。**

## gc

gc的过程一共分为四个阶段：

1. 栈扫描（开始时STW）
2. 第一次标记（并发）
3. 第二次标记（STW）
4. 清除（并发）

整个进程空间里申请每个对象占据的内存可以视为一个图，初始状态下每个内存对象都是白色标记。

1. 先STW，做一些准备工作，比如 enable write barrier。然后取消STW，将扫描任务作为多个并发的goroutine立即入队给调度器，进而被CPU处理
2. 第一轮先扫描root对象，包括全局指针和 goroutine 栈上的指针，标记为灰色放入队列
3. 第二轮将第一步队列中的对象引用的对象置为灰色加入队列，一个对象引用的所有对象都置灰并加入队列后，这个对象才能置为黑色并从队列之中取出。循环往复，最后队列为空时，整个图剩下的白色内存空间即不可到达的对象，即没有被引用的对象；
4. 第三轮再次STW，将第二轮过程中新增对象申请的内存进行标记（灰色），这里使用了write barrier（写屏障）去记录

Golang gc 优化的核心就是尽量使得 STW(Stop The World) 的时间越来越短。

------

#### 三 golang的清除流程 （三色并发标记） 分4个阶段

第一个阶段 gc开始 （stw）

1. stop the world 暂停程序执行
2. 启动标记工作携程（ mark worker goroutine ），用于第二阶段
3. 启动写屏障
4. 将root 跟对象放入标记队列（放入标记队列里的就是灰色）
5. start the world 取消程序暂停，进入第二阶段

第二阶段 marking（这个阶段，用户程序跟标记携程是并行的）

1. 从标记队列里取出对象，标记为黑色
2. 然后检测是否指向了另一个对象，如果有，将另一个对象放入标记队列
3. 在扫描过程中，用户程序如果新创建了对象 或者修改了对象，就会触发写屏障，将对象放入单独的 marking队列，也就是标记为灰色
4. 扫描完标记队列里的对象，就会进入第三阶段

第三阶段 处理marking过程中修改的指针 （stw）

1. stop the world 暂停程序
2. 将marking阶段 修改的对象 触发写屏障产生的队列里的对象取出，标记为黑色
3. 然后检测是否指向了另一个对象，如果有，将另一个对象放入标记队列
4. 扫描完marking队列里的对象，start the world 取消暂停程序 进入第四阶段

第四阶段 sweep 清楚白色的对象
到这一阶段，所有内存要么是黑色的要么是白色的，清楚所有白色的即可
golang的内存管理结构中有一个bitmap区域，其中可以标记是否“黑色”

## golang的调度

 [https://github.com/woxin123/note/blob/master/Golang/Go%E8%AF%AD%E8%A8%80%E9%9D%A2%E7%BB%8F.md](https://github.com/woxin123/note/blob/master/Golang/Go语言面经.md) 

## context 包的用途

context 是 go 语言 goroutine 的长下文，可以用于长下文控制的 goroutine 的结束，也可以用于传递上下文消息。

## slice 的底层原理，扩容机制

golang 中的 slice 数据类型，是利用指针指向某个连续片段的数组。一个 `slice` 在 golang 中占用 24 个 bytes。

在 runtime 的 slice.go 中，定义了 slice 的 struct。

```
type slice struct {
    array unsafe.Pointer  // 8 bytes
    len int    // 8 bytes
    cap int    // 8 bytes
}
```

- array 是指向真实数组的 ptr
- len 是指切片已有元素个数
- cap 是指当前分配的空间

slice 扩容

具体扩容策略：（**好像不准确**）

- 如果申请的容量(cap) 大于 2 倍的原容量，那么新容量 = 申请容量。
- 如果新申请的容量小于 2 倍的原容量，并且原 slice 的长度小于 1024，那么新容量 = 原容量的 2 倍。否则不算计算 `newcap += newcap / 4` 直到不小于需要申请的容量（cap）。

## 基本排序，哪些是稳定的

选择排序、快速排序、希尔排序、堆排序不是稳定的排序算法，

冒泡排序、插入排序、归并排序和基数排序是稳定的排序算法

#### 稳定的重要性

假设你有一个表，又两个列，姓和名（姓和名分开），你对名排序，然后对姓排序，再对名排序的时候稳定性可以保证不破坏姓的顺序。

我先对a和b排序，然后对两个guo排序，a和b很容易排序正确，但是排guo的时候如果是不稳定的，会导致原本的ab的顺序被破坏。

例如：

- a，guo
- b，guo

## defer的坑

 https://deepzz.com/post/how-to-use-defer-in-golang.html 

注意如果闭包的话，传的是引用

return 会做几件事：

1. 给返回值赋值
2. 调用 defer 表达式
3. 返回给调用函数

## golang深拷贝

浅拷贝复制一份地址，就是内存一样。

深拷贝对每个值复制一份，内存不一样。

1. 创建对象空间后慢慢append

2. 调用copy函数，但是注意对象要分配空间。

   ```
   func main() {
   	sliceA := []int{1, 2, 3, 4}
   	sliceB := make([]int, len(sliceA)) // 要分配空间
   	copy(sliceB, sliceA)
   	sliceA[0] = 3
   	fmt.Println(sliceA)
   	fmt.Println(sliceB)
   }
   ```


## 内存管理

为什么go从mspan中获取内存不需要上锁，因为同一时间只有一个goroutine，所以不需要担心上锁的问题。

mspan被分成了大小从8字节到32k字节插槽块，用于存储不同大小的对象

## map实现

 https://zhuanlan.zhihu.com/p/66676224 

 [https://www.cnblogs.com/qcrao-2018/p/10903807.html#%E5%A6%82%E4%BD%95%E8%BF%9B%E8%A1%8C%E6%89%A9%E5%AE%B9](https://www.cnblogs.com/qcrao-2018/p/10903807.html#如何进行扩容) 

#### 数据结构

hmap

```go
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}
```

bmap

```
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```



#### 实现

![下载](C:\Users\78478\Desktop\review\下载.png)

#### key的定位

key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。

例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

```shell
 10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
```

用最后的 5 个 bit 位，也就是 `01010`，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

```text
// key 定位公式
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

// value 定位公式
v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
```

b 是 bmap 的地址，这里 bmap 还是源码里定义的结构体，只包含一个 tophash 数组，经编译器扩充之后的结构体才包含 key，value，overflow 这些字段。dataOffset 是 key 相对于 bmap 起始地址的偏移：

```text
dataOffset = unsafe.Offsetof(struct {
        b bmap
        v int64
    }{}.v)
```

因此 bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。理解了这些，上面 key 和 value 的定位公式就很好理解了。

#### 装载因子

需要有一个指标来衡量前面描述的情况，这就是`装载因子`。Go 源码里这样定义 `装载因子`：

```golang
loadFactor := count / (2^B)
```

count 就是 map 的元素个数，2^B 表示 bucket 数量。

#### 扩容

时机：插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作，渐进式扩容。

再来说触发 map 扩容的时机：在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

1. 装载因子超过阈值，源码里定义的阈值是 6.5。
2. overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。

怎么理解1和2呢？需要注意，每个bucket里面都有8对kv，索引装载因子的阈值不是1（就算要位置跟kv个数1比1的意思也应该是8），2的意思是overflow bucket太多了，每个bucket里面只有一两个kv，因为太多被删掉了，导致本来我应该一个bucket里面装8对kv的，现在变成只有1对kv，而条件1又检测不出来，所以增加了条件2。

针对1和2的扩容机制是不同的，应对条件 1，新的 buckets 数量是之前的一倍，应对条件 2，新的 buckets 数量和之前相等。 Go map 的扩容采取了一种称为“**渐进式**”地方式，原有的 key 并不会一次性搬迁完毕，**每次最多只会搬迁 2 个 bucket**。 

#### 理解

第一次看到golang的map的实现总觉得不对劲，很难理解，跟以前看到过的map的实现有点差距，以前的map都是数组+hash+链表。

实际上golang的map也是数组+hash+链表，你把每一个bmap看成结点就对了，串起来依旧是链表，只不过在这一次的链表结点有点奇怪，以往都是一个结点一个键值对，这次一个结点4个键值对。

所以golang的map本质就是一个结点**8**个键值对。

所以除了定位结点以外，还需要定位结点内部键值对，所以才有了哈希的高8位值。

## sync.map

 https://juejin.im/post/5d36a7cbf265da1bb47da444#heading-8 看看就好，里面不少都是不准确的

以源码为准，其实sync的源码不会很难，调试几次就能看懂，这里要注意不要使用vscode调试看不到里面的map详细的key-val。

#### 好处

使用dirty和read分开的好处，网上说了一堆其实没说到重点，重点就是不用某些操作，例如查询不用加锁，其实删除或者修改如果read也有key的话也不用加锁，但是最重要的是read也是一个map，为什么它不用加锁呢？要知道我们之所以困难的原因就是并发修改会panic，而现在修改read肯定在开发中也是一种并发修改，为什么不用加锁也不会panic？

因为所有对read的修改采用的都是**cas操作**，即**保证原子操作**，这样即使是并发修改，最后也是一个个去修改，这样就不会导致并发修改问题。

所以read的修改才不用加锁。

#### 数据结构

map

```
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
	dirty map[interface{}]*entry
	misses int // miss大于等于dirty长度的时候就会把整个dirty赋值给read（不是只赋值某个key）
}
```

read

```
type readOnly struct {
    m  map[interface{}]*entry
    amended bool //为true代表dirty是最新
}
```

#### 查询

先查read，再查dirty，查dirty的时候要加锁，所以有二次检查，另外miss次数+1，如果miss次数大于等于dirty长度的时候，就dirty赋值给read，整个dirty赋值给read。

#### 删除

先删read（不加锁），删除成功就不删除dirty了，如果没有再删dirty（加索，双重检查），这里涉及到一个问题，怎么删除。

**复盘**：之前没有想过一个问题，为什么我们删除map的时候要加锁，因为并发会panic，那现在为什么删除read的时候不加锁？因为删除read的时候用的是cas操作，就是原子操作，原子操作同一时间只会有一个人去删，所以不会panic。

##### 删除方式

dirty是真删除，read只是标记，**把read的map中的值赋值为nil**，那么问题来了，此时dirty中的值是多少，其实也是nil。

这里就要特别提及，dirty也好read也好，他们的map的值都是指针，而且会相互之间赋值，例如把read中某个key的值（指针里面放着地址，所以是把地址给了对方）给dirty对应的key，所以实际上，dirty和read存的都是同一片内存，他们只是存着指针，只是有人存的key多有人少，但是一旦双方都有某个key必是指向同一片内存。

所以，如果read置nil，此时dirty也是nil，因为内部采用指针存储，而且相互之间互相赋值，都是同一个地址。

##### 查询或者遍历

很容易想到啊，你都没真删，那查询或者遍历的时候，你把nil给搞了出来怎么办？内部是调用`(e *entry) load()`函数，这个函数会判断取出来的值是不是nil，如果是nil则返回false，代表内部没有这个值，遍历的时候自然也会绕过它。

#### 存储（store，更新或者新增）

1. 先尝试存储read，有条件，必须read本身有这个值，且必须不能是expunged（因为expunged代表着dirty中没有这个值，如果你store，待会就会出现read中有而dirty中没有的情况），这里不能直接理解成被标记删除的，实际上从代码上看，read被标记删除有两种情况，通常情况下，被标记删除直接把read的值置nil，另外一种情况下，nil会变成expunged，但是本质上依然代表标记删除的意思。
2. 如果存储read失败，最常见就是存储一个新的key（还有各种情况），存储到dirty上面
3. 需要注意，在存储到dirty上面的时候还有一种场景，就是存储的时候，如果发现read是最新的，amended是false的时候，会把read上面所有不是nil的key迁移到dirty上面，并且把nil置成expunged。

#### Store总结

##### 更新

1. 如果read中有这个key，分2种情况：
   - 如果key的值是expunged，那么不仅要更新该值，说明dirty中也没有该key，所以还要给dirty赋值，其实就是先把read的entry指针赋值给dirty对应的key，然后再给entry赋值，赋值一次read和dirty就都有这个值了。
   - 非expunged的情况，无论read中key的值是不是nil，都说明dirty中有对应的值（或者dirty为nil），直接给entry赋值，由于是指针，那么dirty中的entry也会带上这个值。

2. 如果read中没有这个值，并且dirty中有这个值，那么直接赋值给dirty。
3. 如果dirty中也没有这个值，那么就是新增的情况，详情看下面的新增。

##### 新增

只要是新增并且amended是false，即read是最新的，都会触发把read里面的东西全部赋值给dirty。

对于新增的情况，都是新增到dirty上面，但是这其中有一种特殊情况，那就是dirty为nil，这时候需要把read中不是nil的值全部拷贝到dirty中，并且把read中为nil的值变成expunged

#### 什么情况下会为expunged

只有一种情况，read本身的值是nil，然后又需要把read值给dirty的时候，这时候这些nil的值会被置成expunged，并且不会赋值给dirty。

**所以这时候，会出现这种情况，read的key比dirty多，因为那些nil的值不会赋值过去dirty**。

要弄出这种场景也不难，先随便store几个值进去map，然后range一遍，这时候所有的值都跑到read上面，dirty清空，这时候再把之前所有的key清掉（delete），此时read中所有的值就变成nil，然后store一个新的key，此时store的过程中就会把nil变成expunged，并且这些nil的值不会更新到dirty上面。

```
sm.Store(1, "a")
sm.Store(2, "a")
sm.Range(func(k, v interface{}) bool {
		fmt.Print(k)
		fmt.Print(":")
		fmt.Print(v)
		fmt.Println()
		return true
	}) // dirty清空，dirty全跑到read上面去了
sm.Delete(2) // 标记删除，这时候read是nil，不是expunged
sm.Delete(1)
//在把read所有的key赋值给dirty的时候,会调用tryExpungeLocked判断read的值是否是nil,然后会把nil赋值为expunged,并且不会把该key赋值到dirty上面
sm.Store(3, "a") // 新的key，这时候就会发现nil变成expunged了
```

#### expunged和nil

其实对于read来说，expunged和nil都是删除标记，不同的是nil表示的是dirty中也有这个key，但是现在和read一样值都是nil，而expunged则表示dirty中没有这个key，是真的没有这个key，而不是有这个key然后值等于nil，连有都没有

#### amended

我是这么理解这个字段的，如果为true代表dirty是最新的，按照这个字段的注释，它说，当dirty存在一些key是read中没有的时候，它是true。

那么这个一些key是read中没有，就可以差不多理解为dirty比read新。

反过来，如果是false，read是最新的，但是dirty也可能是最新的。

## channel的实现原理

 [https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#642-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#642-数据结构) 里面说的不错，再辅助源码可以基本看懂

### 数据结构

```
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	elemtype *_type
	sendx    uint  
	recvx    uint
	recvq    waitq
	sendq    waitq

	lock mutex
}
```

1. channel本质就是一个加了锁的循环队列，先说说队列的实现，构体中的五个字段 `qcount`、`dataqsiz`、`buf`、`sendx`、`recv` 构建底层的循环队列：

   - `qcount` — Channel 中的元素个数；
   - `dataqsiz` — Channel 中的循环队列的长度；
   - `buf` — Channel 的缓冲区数据指针；
   - `sendx` — Channel 的发送操作处理到的位置；
   - `recvx` — Channel 的接收操作处理到的位置；

   对于队列本身不想深谈，总之就是一个队列

2. 接下来其实是很关键的部分，你chanel想发送数据，你得知道要发到哪个g上面，所以有两个很重要的结构，`recvq` 和`sendq` 里面存储着要接收该channel数据的g队列，和发送到该channel的g队列。

   这里需要注意，虽然我们用队列来形容，但是本质上这俩应该叫做双向链表比较合适，因为从数据结构上来讲，他们属于链表（可以自己尝试看源码，里面就俩指针，first和last）
   
3. 锁（数据结构由这三个部分组成，缺一不可）。

### 发送

#### 直接发送

##### 史诗级别总结

1. 有chan，有要发送的值。
2. 从chan的接收队列中 dequeue 一个g，g.elem就是接收的变量。
3. 调用 memmove ，把要发送的值拷贝给 g.elem 。
4. 唤醒g（唤醒的方式就是把它挂到当前协程的runnext下面）。

##### 具体流程

接下来我大概讲一下channel是怎么发送的，过程以补充上面的连接为主，以及连接中没有提到的东西。

1. 首先啥都不用管，不用管`ch<-1`是怎么回事，你只需要知道，这个形式的代码会被编译器解析到（经过一系列解析）下面这个函数

   ```
   func chansend1(c *hchan, elem unsafe.Pointer) {
   	chansend(c, elem, true, getcallerpc())
   }
   ```

   从这个函数中我们可以知道，c是chan，elem代表要发送的数据，所以在接下来的函数中，你不需要管，chan是哪来的，它怎么知道要发送的是什么数据，因为我也不知道，我只知道函数走到chansend1的时候，这两个东西已经有了，所以下面的函数默认就有。

   从chansend1可以看出，实际上它就是调用了chansend，这个函数才是大头

2. 在chansend函数中（前面一大段不需要理会）先加锁，因为前面传入的参数是true，如果数据能够直接发送（推测前面就是在检测到底能不能直接发送），那么它会从chan里面的goroutine接收队列中取出最前面的g，所以这里可以推测出，谁最先陷入阻塞接收，就先取出哪个goroutine。

3. 取出goroutine后，调用send函数

   ```
   if sg := c.recvq.dequeue(); sg != nil {
   		// Found a waiting receiver. We pass the value we want to send
   		// directly to the receiver, bypassing the channel buffer (if any).
   		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
   		return true
   	}
   ```

4. send函数中，c代表chan，sg代表刚才取出来的goroutine（接收方的g），ep代表一开始要发送的数据（第1点说了，不要问数据哪里来的，最开始就有），锁函数（调用就完事了），skip不太懂事什么。

   其实在send中关键就是俩函数，一个是真正去发送数据，也就是`ch<-1`这个代码中，真真正正去发送数据的代码，就是

   ```
   // 如果要接收的g的变量不为nil的话，怎么理解呢
   // 接收的chan: x<-ch，那么这里的x就是sg.elem，也就是dst，就是接收的变量
   if sg.elem != nil {
   		sendDirect(c.elemtype, sg, ep)
   		sg.elem = nil
   	}
   ```

   sendDirect代码，elemtype不是不是很懂（后面只是用该类型的大小，推测它的最用就是指明要传送的数据大小而已），sg代表接收的g，ep代表要发送的数据

5. 好了现在就让我们进入send中，进入之前，我们看看，目前来讲，要发送的数据有了（ep），能接收数据的g有了，并且具体是g中的哪个变量接收也有（sg.elem），条件具备，那么到底是怎么发送接收数据的呢？

   其实本质就是汇编move，copy之类的指令，你想想，我channel的发送数据和接收数据，是不是就是把某个变量的值给copy给另外一个变量，只不过他们现在可能在不同的两个goroutine上面，本质还是赋值。

   这里采用的是send函数里面的`memmove`

   ```
   // memmove copies n bytes from "from" to "to".
   // in memmove_*.s
   //go:noescape
   func memmove(to, from unsafe.Pointer, n uintptr)
   ```

   具体实现在`memmove_*.s`汇编文件中，看平台。

6. 回到第4点的send函数那里，函数拷贝完成之后，还需要唤醒接收方，毕竟接收方现在还阻塞着，但是这里采用的方式不是直接唤醒，而是把它加入当前g的p的 `runnext`  。

   1. 先调用`goready`

      ```
      func goready(gp *g, traceskip int) {
      	systemstack(func() {
      		ready(gp, traceskip, true)
      	})
      }
      ```

   2. 在ready函数中先获取到当前运行的g，`_g_ := getg()`，在第4点的时候我们提到了，我们从接收队列中取出了g，现在我们又获取到了当前的g（两个g不一样），根据当前的g可以获取到m进而获取到p（pmg模型）。

   3. 然后把接收队列的g放入当前g的p的 `runnext`  上，等待执行。

      ```
      // _g_可以获取到m进入获取到p，接着把gp也就是接收队列的g放入
      //p的next中
      runqput(_g_.m.p.ptr(), gp, next) 
      ```

#### 放入缓冲区

1.  用`chanbuf`  计算出buf中下一个可以放入的位置sendx，sendx就是dst。
2. 调用`typedmemmove`将数据（一开始就有）move到sendx的位置，`typedmemmove`里面还是用到了`memmove`函数。
3. 最后就是一些处理，sendx++，qcount++

#### 阻塞发送

##### 名词解释

 sudog 代表了一个在等待中的g 

##### 流程

1. 获取当前的goroutine。
2. 创建一个sudog，这个sudog是一个封装的g，代表等待中的g
3. 对这个sudog设置data（就是要发送的数据），设置g，设置chan。
4. 把sudog设置到当前goroutine的waiting上。
5. 将sudog加入到chan的发送队列。
6. 调用goparkunlock，陷入阻塞，等待接收端接收消息。

### 接收

接收也分三种，直接接收对应阻塞发送，其他依次类推，参照上面的连接，这里就不一一列举了。

这里需要注意以下，函数中经常看到一个ep，这个ep可以这么理解，对于发送来说，例如：`ch<-x`，ep就是变量x，也可以说是要发送的数据。

对于接收来说，`y<-ch`，现在ep不是要发送的数据，而是接收数据的变量y，那么要发送的数据在哪？在发送队列里面，取出队列里面一个g，g.elem就是要发送的数据。

同样的，对于发送来说，g.elem代表的是接收的变量