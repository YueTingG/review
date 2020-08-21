# 堆

## 堆的构建

首先，区分一下两个概念，堆是有调整的，但是调整分两种，一种是**自上而下**，一种是**自下而上**，需要注意的是，从数组构建堆的时候，是**自下而上构建**的，但是**运用的调整却是自上而下**。

1. 初始数组为：{ 9,3,7,6,5,1,10,2 }，这里要说一下，往往很多人索引为0的地方不放元素，就是第一个位置就当不存在。这里第一个位置是有元素的。

2. 现在假设长度为len，有len=len(arr)，假设某个元素的索引为i，它的左孩子的索引就为**left=2i+1**，**right=2i+2**，可以通过举例的方式求出这个规律（如果是首位无元素则是left=2i，right=2i+1）。

3. 现在一切**从最后一个结点的父亲开始**，那么我要搞定另外一个问题，**怎么求最后一个结点的父亲**。

   ```
   现在我们可以知道，最后一个结点的索引为 len-1，又存在left=2n+1以及right=2n+2，
   所以如果最后一个结点为左结点,有len-1 = 2n+1,可得n=len/2-1,同理右节点有n=len/2-1.5=len/2-1-0.5
   所以我们能知道，当last为左节点时，用n=len/2-1能正确求出父节点，当last为右节点时，如果我们也用n=len/2-1去求的话，最终的结点将比原来的值大0.5，但是我们知道整数类型是向0取整的，所以最终结果将还是右节点的父亲
   ```

   结论，求最后一个结点的父亲，**把最后一个结点统统当作左孩子处理，再配合上求左孩子的公式，可得到左孩子的父亲**，公式为**i=len(arr)/2-1**

4. 同时我们也可以根据第3点，得到一个推论，结点i（把它当成左孩子）的父节点为 **(i-1)/2**

   ```
   Note:
   Root is at index 0 in array.
   Left child of i-th node is at (2*i + 1)th index.
   Right child of i-th node is at (2*i + 2)th index.
   Parent of i-th node is at (i-1)/2 index.
   ```

i=n/2-1；即i=3，从6开始，选择根、左孩子、右孩子中最小的那一个作为根。这里还需要**注意左孩子和有孩子的索引范围**

 ![img](https://img-blog.csdn.net/20160521000756369?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

6比较完了，就让索引--，即现在索引为2，即元素是7，选择最小的那个作为根。



 ![img](https://img-blog.csdn.net/20160521001319762?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

 ![img](https://img-blog.csdn.net/20160521002504850?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

注意：数字9的节点 将和 数字1的节点 发生对调，**对调后，需要递归进行调整**，请一定注意。

 ![img](https://img-blog.csdn.net/20160521002603476?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

 ![img](https://img-blog.csdn.net/20160521002536788?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

最小堆代码，仍然要强调的是，从构建的角度来讲是**自下而上构建**（从**len(arr) - 1**开始向上构建）的，但是构建的过程需要调整，**调整的过程是自上而下**，不要混淆。

```
func BuildMinHeap(arr []int) {
	start := len(arr) - 1
	for i := start; 0 <= i; i-- {
		minHeapify(arr, i)
	}
}

// 堆化, 实际上是向下调整法
func minHeapify(arr []int, cur int) {
	length := len(arr)
	left := 2*cur + 1
	right := 2*cur + 2
	min := cur
	if left < length && arr[left] < arr[min] {
		min = left
	}

	if right < length && arr[right] < arr[min] {
		min = right
	}

	if min != cur {
		arr[cur], arr[min] = arr[min], arr[cur]
		minHeapify(arr, min)
	}
}

// 迭代的算法
//func minHeapify(arr []int, cur int) {
//	length := len(arr)
//	for cur*2+1 < len(arr) {
//		left := cur*2 + 1
//		right := cur*2 + 2
//		min := cur
//		if arr[left] < arr[min] {
//			min = left
//		}
//		if right < length && arr[right] < arr[min] {
//			min = right
//		}
//		if min != cur {
//			arr[cur], arr[min] = arr[min], arr[cur]
//			cur = min
//		} else {
//			break
//		}
//	}
//}
```

最大堆代码

```
func BuildMaxHeap(arr []int) {
	start := len(arr)/2 - 1
	for i := start; 0 <= i; i-- {
		heapify(arr, i)
	}
}

func heapify(arr []int, i int) {
	length := len(arr)
	left := 2*i + 1
	right := 2*i + 2
	largest := i
	// If left child is larger than root
	if left < length && arr[left] > arr[largest] {
		largest = left
	}

	// If right child is larger than largest so far
	if right < length && arr[right] > arr[largest] {
		largest = right
	}

	// If largest is not root
	if largest != i {
		//swap(arr[i], arr[largest])
		arr[i], arr[largest] = arr[largest], arr[i]
		// Recursively heapify the affected sub-tree
		heapify(arr, largest)
	}
	return
}
```

## 堆的插入

将要插入的数放到末尾，然后调整。这里需要用到**自顶向上调整**，但实际上就是跟cur的父节点进行比较。

 **Arr[(i -1) / 2]** returns its parent node. 

**注意，在实际实现中，不会用slice这么龊的方法，这里为了简便，因为append会改变地址的原因，所以这里不得不传入了一个指针。但是实际实现中大概率会用结构体，所以这里暂时忽视，slice指针的问题。**

```
func insert(arr *[]int, ele int) {
	*arr = append(*arr, ele)
	fmt.Printf("%p\n", &arr)
	cur := len(*arr) - 1
	parent := (cur - 1) / 2
	// (0-1)/2 == 0不是负数
	for (*arr)[cur] < (*arr)[parent] {
		(*arr)[cur], (*arr)[parent] = (*arr)[parent], (*arr)[cur]
		cur = parent
		parent = (cur - 1) / 2
	}
}
```

## 堆的删除

堆只能删除顶部元素（某些包也能删除任意元素，但是通常我们只删顶部）。

1. 先删除小顶堆的顶部元素，**然后将小顶堆的最后一个元素放在顶部**，再依次与它小的孩子节点的值进行比较，
2. 如果大于小的孩子节点的值，则与这个小的孩子节点交换位置，如此直到循环结束；
3. 否则，位置不变，退出循环

总结：一句话，**删除顶部**，然后**和末尾交换**，然后调整，采用**自顶向下调整**

注意，**删除不等于把头结点给删掉，然后从新的头结点自上而下调整**。如果采用这样的操作，会发现，删除头结点之后，之前所有的父子关系全乱了，别人的孩子此时可能变成我的孩子，那么会出现一种情况，就是（拿小顶堆举例）我可能比我的孩子还大，所以此时的堆结构就被全部破坏了。

而如果你只是把最后一个结点跟头结点交换，那么被破坏的堆只有最上面的一颗子树，所以对最上面采用调整。但是如果你采用上面的操作，被破坏的就可能是整个堆结构，这个时候单纯地调整头结点已经没有意义了。

（同样用了slice指针，别在意）

```
func pop(arr *[]int) int {
	tmp := (*arr)[0]
	last := len(*arr) - 1
	(*arr)[0] = (*arr)[last]
	*arr = (*arr)[:last]
	minHeapify(*arr, 0)
	return tmp
}
```

## golang的写法

上面所有的代码旨在讲解底层，要尽量掌握，实际上golang内置了一个heap的实现，还是需要写少量的代码来完成heap但是最基本的两种调整方法heap已经实现，只需要像实现接口一样实现Push和Pop以及Sort的接口。

最大堆最小堆的实现差别只有Less(i, j int) bool实现的区别

最大堆

```
func (h IntHeap) Less(i, j int) bool  { return h[i] > h[j] } // use > for MaxHeap
```

最小堆

```
func (h IntHeap) Less(i, j int) bool  { return h[i] < h[j] } // use < for MinHeap
```

一个是大于号，一个是小于号