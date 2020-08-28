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

## 线程缓存

#### 数据结构

```go
type mcache struct {
    alloc [numSpanClasses]*mspan
}
```

从上面可以看出，