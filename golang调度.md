# 从 golang 的启动说起

 [https://zboya.github.io/post/go_scheduler/#g0%E5%92%8Cm0](https://zboya.github.io/post/go_scheduler/#g0和m0) 

 [https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#653-%E8%B0%83%E5%BA%A6%E5%99%A8%E5%90%AF%E5%8A%A8](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#653-调度器启动) 

## 启动 go 进程的 4 个步骤

#### 4个步骤

这 4 个步骤，我们如果查看 `schedinit()` 函数，我们会发现：

```
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
```

可以解释为：

1. 调用 `runtime·osinit` 来获取系统的cpu个数。
2. 调用 `runtime·schedinit` 来初始化调度系统，会进行p的初始化，也会把m0和某个p绑定。 
3. 调用 `runtime·newproc` 新建一个goroutine，也叫`main goroutine`，它的任务函数是 `runtime.main` 函数，建好后插入m0绑定的p的本地队列。
4. 调用 `runtime·mstart` 来启动m，进入启动调度系统。 

#### 4个步骤的实现

这 4 个步骤的具体实现是在`runtime/asm_amd64.s`  这个文件，更前面的我也不知道，咱们不要去理，咱们看 golang 的启动就从这个代码看起，这个代码块里面就有这 4 个步骤

```assembly
// runtime·rt0_go

// 程序刚启动的时候必定有一个线程启动（主线程）
// 将当前的栈和资源保存在g0
// 将该线程保存在m0
// tls: Thread Local Storage
// set the per-goroutine and per-mach "registers"
get_tls(BX)
LEAQ	runtime·g0(SB), CX
MOVQ	CX, g(BX)
LEAQ	runtime·m0(SB), AX

// m0和g0互相绑定
// save m->g0 = g0
MOVQ	CX, m_g0(AX)

// save m0 to g0->m
MOVQ	AX, g_m(CX)

// 处理args
CALL	runtime·args(SB)

// 1.os初始化， os_linux.go
CALL	runtime·osinit(SB) 

// 2.调度系统初始化, proc.go
CALL	runtime·schedinit(SB) 

// 3.创建一个goroutine，然后开启执行程序
MOVQ	$runtime·mainPC(SB), AX		// entry
PUSHQ	AX
PUSHQ	$0			// arg size
CALL	runtime·newproc(SB)
POPQ	AX
POPQ	AX

// 4.启动线程，并且启动调度系统
CALL	runtime·mstart(SB)
```

#### m0 和 g0 的创建

我们尝试理解一下这些汇编代码，一开始会创建主线程也就是m0，也会创建 g0（在代码中不太能体现，我们就当作已知条件），并且 g0 的栈是操作系统利用系统调用创建的，跟一般的 g 的2kB的栈不一样。

所以，从一开始就会有 m0，g0，**他们相互绑定**，这两个也是全局的 m0，g0，注意这个时候没有 p，一个都没有。

#### g0 的使用

所以接下来的 4 个步骤从理解上来讲都是 g0 去执行的，证据就是调用 `_g_ := getg()` 的时候获取到的是 g0。

## schedinit 调度初始化

其实，`schedinit` 很简单，用一句话就能概括（其他的记不住的，不如简单记住就好）：初始化 p。 

#### 初始化过程

1. 初始化 p，p 的个数是 cpu 数。

   （所有的 p 都已经初始化好了，后面不需要再 new 之类的，但是 m 和 g 需要）

2.  通过指针将线程 m0 和处理器 `allp[0]` 绑定到一起。

#### 注意

这时候的 `p` 是全部初始化好了，但是 `m` 没有，`g` 更没有，也就是讲，现在其实就是有 n （n 是 cpu 的个数）个p，然后1个 `m`，2个 `g`，分别就是 `m0`，`g0` 和 第 3 步创建的 runtime.main 的 Goroutine。

`m` 和 `g` 都要在后边 new 出来，他们都有对应的 `newm` 和 `newg` 函数。

## 调用 mstart 函数

#### 提前要知道

这个时候的 m 就是 m0，g 是 g0，为什么会知道这个？因为 mstart 代码中执行了 `_g_ := getg()`，可以知道当前执行 mstart 的是 g0 （你就当作是已知条件）。

上面汇编的时候说了，现在 m0 和 g0 互相绑定，而且因为调度初始化完毕，现在 m0 已经和 p0 绑定好了，所以现在 m0，g0，p0 已经在一块了。

#### 函数内容

1. 做一些基本判断，然后 panic，特殊处理之类的
2. 查看 m 上面是否有任务函数（一开始没有，runtime.main函数是 Goroutine 的，不是 m 的任务函数，不要搞混了）
3. 执行 `schedule()` （最重要）

整个 mstart 函数其实在第一次调用的时候唯一的意义就是调用 `schedule()` 函数，让它去执行调度。

## 真正的调度函数 schedule

```
func schedule() {
	_g_ := getg()

	...

top:
	// 如果当前GC需要停止整个世界（STW), 则调用gcstopm休眠当前的M
	if sched.gcwaiting != 0 {
		// 为了STW，停止当前的M
		gcstopm()
		// STW结束后回到 top
		goto top
	}

	...

	var gp *g
	var inheritTime bool

	...
	
	if gp == nil {
		// 每隔61次调度，尝试从全局队列种获取G
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		// 从p的本地队列中获取
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
	if gp == nil {
		// 想尽办法找到可运行的G，找不到就不用返回了
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	...

	// println("execute goroutine", gp.goid)
	// 找到了g，那就执行g上的任务函数
	execute(gp, inheritTime)
}
```

#### 调度过程

1. 每隔61次调度轮回从全局队列找，避免全局队列中的g被饿死。
2. 从p.runnext获取g，从p的本地队列中获取。
3. 调用 `findrunnable` 找 g，找不到的话就将m休眠，等待唤醒。 

`findrunnable` 会去尝试窃取其他 p 的 Goroutine。有关它的请看 [`findrunnable` 的窃取](##`findrunnable` 的窃取)

找到 g 之后，就会调用 `execute(gp, inheritTime)`。

#### 调度是为了干嘛

我们这里需要总结一下，这个 `schedule` 的作用是什么？我们为什么要调用这个函数？什么情况下我们要调用这个函数？

其实就一句话，**选出一个 g，然后运行（execute）它**，这是调度的真正意义。

所以当我们要运行 g 的时候，我们才需要调度函数，什么时候我们需要运行g？当你写了类似 `go fn()` 的代码去起一个协程的时候，你就需要 `schedule` 函数。

我们可以看到调度函数的最后一句 是`execute(gp, inheritTime)`，所以下面我们需要讲解 `execute` 代码。

## goroutine 的执行 execute

```go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	_g_.m.curg = gp
	gp.m = _g_.m
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	gogo(&gp.sched)
}
```

我们从代码中可以看到，最重要的就是最后一行 `gogo(&gp.sched)`，每个平台的 gogo 的实现不同，但是有基本的实现方式：

```
	MOVL gobuf_sp(BX), SP  // 将 runtime.goexit 函数的 PC 恢复到 SP 中
	MOVL gobuf_pc(BX), BX  // 获取待执行函数的程序计数器
	JMP  BX                // 开始执行
```

![](C:\Users\78478\Desktop\review\2020-02-05-15808864354661-golang-gogo-stack.png)

以上三个之类模拟了 call 指令，goexit 是用来退出 goroutine，先不用管，我们获取到 runtime.main 的程序计数器之后，`JMP` 到那个位置就能自动执行 runtime.main 函数了（你就只需要知道跳到函数的程序计数器就可以执行函数，而 Goroutine 的作用就是执行函数，就行了）。

#### 执行 runtime.main

在 runtime.main 中会调用 main_main 执行用户代码的 main，也就是我们通常写的 main 函数（我们平常写的 main 函数）。

#### goexit 协程的退出

```
TEXT runtime·goexit(SB),NOSPLIT,$0-0
	CALL	runtime·goexit1(SB)

func goexit1() {
	mcall(goexit0)
}

func goexit0(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gdead)
	gp.m = nil
	...
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	dropg()
	// 将G放入P的G空闲链表
	gfput(_g_.m.p.ptr(), gp)
	
	schedule() // 非常重要的一句，证明了调度是一种循环调度
}
```

goroutine 的退出本身没什么好说的，就是把各种字段置 nil， 移除 m 和 g 的关联，并把 g 加入到 p 的 gFree 空闲列表中。

但是最重要的就是调用了 `schedule()`，证明了调度是一种循环调度。

# go 进程启动之后，运行 go协程

从第一部分，我们已经了解了 go 进程启动过程的调度，我们可以知道在这个过程中所有的 p 已经初始化完毕，但是现在 m 只有一个 m0，g 有一个 g0 以及一个运行着 runtime.main 的 g，如果我们现在使用语法 `go func()` ，创建一个 goroutine，会怎么样？调度该如何？

```
func main() {
	go func(){
		println("a")
	}()
}
```

例如上述代码运行后，内部的调度是怎么样的？

我们在第一部分的过程中其实省略如何创建一个 goroutine，现在就补完这一部分。

## 创建 goroutine

 想要启动一个新的 Goroutine 来执行任务时，我们需要使用 Go 语言中的 `go` 关键字，这个关键字会在编译期间通过以下方法 `cmd/compile/internal/gc.state.stmt` 和 `cmd/compile/internal/gc.state.call` 两个方法将该关键字转换成 `runtime.newproc` 函数调用： 

```go
// 新建一个goroutine，
// 用fn + PtrSize 获取第一个参数的地址，也就是argp
// 用siz - 8 获取pc地址
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	pc := getcallerpc()
	// 用g0的栈创建G对象
	systemstack(func() {
		// 关键的 new 函数是这个
		newproc1(fn, (*uint8)(argp), siz, pc)
	})
}
```

其实重点在 `newproc1(fn, (*uint8)(argp), siz, pc)`，但是这个函数特别长，长的离谱，具体看上面的连接查找，所以这里总结一下怎么 new g。

#### goroutine 的初步创建

1. 从 Goroutine 所在处理器的 `gFree` 列表或者调度器的 `sched.gFree` 列表中获取 `runtime.g` 结构体；

2. 调用 `runtime.malg` 函数生成一个新的 `runtime.g`函数并将当前结构体追加到全局的 Goroutine 列表 `allgs` 中。

   调用 new 创建 g 结构体（就是我们通常创建结构体那样，只不过它用的是 new），然后调用 `runtime.malg` 会为 g 分配 2KB 的栈空间。

*我们这里需要稍微说明一下，一个G一但被创建，那就不会消失，因为runtime有个`allgs`保存着所有的 g 指针，但不要担心，g 对象引用的其他对象是会释放的，所以也占不了啥内存，而且会把 G 放到 gFree 里面*

#### 将传入的参数移到 Goroutine 的栈上 

`memmove` 函数将 `fn` 函数的全部参数拷贝到栈上，`argp` 和 `narg` 分别是参数的内存空间和大小，我们在该方法中会直接将所有参数对应的内存空间整片的拷贝到栈上： 

```go
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
	// ...
	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize
	totalSize += -totalSize & (sys.SpAlign - 1)
	sp := newg.stack.hi - totalSize
	spArg := sp
	
	// 移动的关键
	if narg > 0 {
		memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
	}
	...
```

#### 更新 Goroutine 调度相关的属性

最重要 3 个属性：

1. 栈指针。
2. 函数的程序计数器。
3. g 的状态

```go
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {	
	// ...
    // 为了方便理解，这些代码的位置经过调整，实际代码这些肯定是有的，但是位置不一样
    
    sp := newg.stack.hi - totalSize
	newg.sched.sp = sp
    
    // goexit 函数要放到程序计数器里面
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum
    
    // 把 newg.sched 的程序计数器设置成 fn，里面顺带把 goexit 放到 sp 里面去了
	gostartcallfn(&newg.sched, fn)
    
    // 设置 g 的状态
	casgstatus(newg, _Gdead, _Grunnable)
	// ...
```

#### 将 Goroutine 加入处理器的运行队列（ 若本地队列已满，插入全局队列）

来了，这个地方才是第 2 部分的一个重点，我们在最开始的时候的问题其实没有解决，我们最开始的问题就是，我现在运行了我们 main 函数，而且现在我的 main 函数又运行了一个 Goroutine，现在该怎么办？我们难道就直接加入到处理器的运行队列？也就是 p0 里面，那这样永远都不会用到别的 p，不是吗？你想想看，我在 p0 上创建 Goroutine，永远都只能加入到 p0 的队列，那别的 p 永远都用不上，因为别的 p 上面没有 Goroutine，无法调度，无法窃取别的 p。

我们知道现在其实就 1 个m，明显不符合 gmp 模型啊，你永远都只有一个 m？那你还不赶紧回家种田去！

所以这里的代码应该是这样的：

```go
	...
	runqput(_p_, newg, true)

	// 如果有空闲的 p 且 m 没有处于自旋状态 且 main goroutine已经启动，那么唤醒某个 m 来执行任务
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		// 唤醒M或者新建M
		wakep()
	}
}
```

首先，无论如何，g 创建好了，先放到 p 里面去，现在 p 正是 g0 和 runtime.main 所在的 p，先放进去（ 若本地队列已满，插入全局队列）。

然后我们尝试去唤醒 m，然后你就会发现唤醒个鬼，现在 m 就一个，而且符合上面的条件，所以进入 wakep，看看是唤醒或者新建，其实可以看出一定是新建，我都说了现在只有一个 m，你要唤醒谁？必然新建。

新建 m 之后，m 自然会去调用 `mstart` 函数，然后调用 `schedule`函数，然后 m 就回去找 g，发现自己没有，就回去抢别人的...好了，我们就先看看 `wakep` 的实现。

## 什么时候会调用wakep？

首先，m 的唤醒和创建都是通过调用 `wakep`，我们自然会问一个问题，**m 的创建或者唤醒都是在什么时候**？**即什么时候会调用 wakep**

根据上面我们可以知道，在创建 Goroutine 的时候，实际上还有在 ready 的时候，只不过我上面没说而已。

#### 新建goroutine的相关代码

```go
	// 如果有空闲的p 且 m没有处于自旋状态 且 main goroutine已经启动，那么唤醒某个m来执行任务
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		// 唤醒M或者新建M
		wakep()
	}
```

#### goroutine准备好相关代码

```go
// 将gp的状态更改为_Grunnable，以便调度器调度执行
// 并且如果next==true，那么设置为优先级最高，并尝试wakep
func ready(gp *g, traceskip int, next bool) {
    // ...
    
	runqput(_g_.m.p.ptr(), gp, next)
	// 如果有空闲P且没有自旋的M。
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		// 唤醒M或者新建M
		wakep()
	}
	
    // ...
}
```

##  wakep 函数

#### `wakep` 代码： 

```go
func wakep() {
	// be conservative about spinning threads
	// 如果有其他的M处于自旋状态，那么就不管了，直接返回
	// 因为自旋的M会拼命找G来运行的，就不新找一个M（劳动者）来运行了。
	if !atomic.Cas(&sched.nmspinning, 0, 1) {
		return
	}
	startm(nil, true)
}
```

可以看出，大头在 `startm`（区分 `mstart` ，`mstart` 是在调用 `schedule` 之前必须有的步骤，也就是说如果你想不起来是怎么调用 `schedule`  ，你就记住是通过 `mstart`  调用的 `schedule`）

#### startm 函数

```
// startm是启动一个M，先尝试获取一个空闲P，如果获取不到则返回
// 获取到P后，在尝试获取M，如果获取不到就新建一个M
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
	// 如果P为nil，则尝试获取一个空闲P
	if _p_ == nil {
		_p_ = pidleget()
		if _p_ == nil {
			unlock(&sched.lock)
			if spinning {
				if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
					throw("startm: negative nmspinning")
				}
			}
			return
		}
	}
	// 获取一个空闲的M
	mp := mget()
	unlock(&sched.lock)
	if mp == nil {
		var fn func()
		if spinning {
			fn = mspinning
		}
		// 如果获取不到，则新建一个，新建完成后就立即返回
		// 苍天啊，上帝啊，新建 m 的代码就在这里
		newm(fn, _p_)
		return
	}

	// 到这里表示获取到了一个空闲M
	if mp.spinning {
	// 从midle中获取的mp，不应该是spinning状态，获取的都是经过stopm的，stopm之前都会推出spinning
		throw("startm: m is spinning")
	}
	// 这个位置是要留给参数_p_的，stopm中如果被唤醒，则关联nextp和m
	if mp.nextp != 0 { 
		throw("startm: m has p")
	}
	// spinning状态的M是在本地和全局都获取不到工作的情况，不能与spinning语义矛盾
	if spinning && !runqempty(_p_) {
		throw("startm: p has runnable gs")
	}
	// The caller incremented nmspinning, so set m.spinning in the new M.
	mp.spinning = spinning //标记该M是否在自旋
	mp.nextp.set(_p_)      // 暂存P
	notewakeup(&mp.park)   // 唤醒M
}
```

思路：

1. 获取空闲的 p，如果没有，return
2. 获取空闲的 m，如果没有**新建 m**（这里就是第 2 部分的答案，很隐蔽，但是就是在这里新建了 m 的）。
3. 否则唤醒 m。

新建 m 的代码时调用 `newm`，我们接下来就看看它是怎么干活的。

## 创建 m

#### newm 函数

```go
func newm(fn func(), _p_ *p) {
	// 根据fn和p和绑定一个m对象
	mp := allocm(_p_, fn)
	// 设置当前m的下一个p为_p_
	mp.nextp.set(_p_)
	
	...
	
	// 真正的分配os thread
	newm1(mp)
}

func newm1(mp *m) {
	// 对cgo的处理
	...
	
	execLock.rlock() // Prevent process clone.
	// 创建一个系统线程，并且传入该 mp 绑定的 g0 的栈顶指针
	// 让系统线程执行 mstart 函数，后面的逻辑都在 mstart 函数中
	newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))
	execLock.runlock()
}
```

从代码中，我们可以看出，真正最重要的一句代码是 `newosproc` 函数，我们看一下它是怎么实现的。

#### linux 平台 `newosproc` 实现。

```go
// 分配一个系统线程，且完成 g0 和 g0上的栈分配
// 传入 mstart 函数，让线程执行 mstart
func newosproc(mp *m, stk unsafe.Pointer) {
	...
	var oset sigset
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
	// stk 是 g0.stack.hi，也就是说 g0 的堆栈是当前这个系统线程的堆栈，也被称为系统堆栈
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	sigprocmask(_SIG_SETMASK, &oset, nil)
	...
}
```

linux平台下，是调用 `clone` 系统调用来实现创建线程。然而，你可能会发现一个问题，我们创建 m 的目的是为了运行 `go func(){ println("a") }()` ，但是现在在创建 m 的过程中，并没有把 m 和 Goroutine 绑定，那运行个鸡？

还记得我们在创建完 Goroutine 的时候，我们最后是怎么处理这个 Goroutine 吗？我们把它放进了 p0 的本地运行队列了，然后运行了 `wakep` 函数，才有 m 的创建，现在确实没有把 m 和 g 关联起来。

但是，新建线程的有任务函数，为 `mstart` ，所以当线程启动的时候，是执行 `mstart` 函数的代码。 还记得 `mstart` 最后干了什么吗？执行调度函数 `schedule` 啊，调度函数中有一个 `findrunnable` ，它会去窃取别的 p 的 g，所以最后确实没有关联 g 和 m，但是调度函数会帮我们解决这个问题。

## `findrunnable` 的窃取

首先，这个鬼函数太长了，如果要看，自己看连接，我讲一下它的具体实现就行了。

#### 运行过程

1. 从本地运行队列、全局运行队列中查找；
2. 从网络轮询器中查找是否有 Goroutine 等待运行；
3. 通过 [`runtime.runqsteal`](https://github.com/golang/go/blob/8d7be1e3c9a98191f8c900087025c5e78b73d962/src/runtime/proc.go#L5147) 函数尝试从其他随机的处理器中窃取待运行的 Goroutine，在该过程中还可能窃取处理器中的计时器；（这个就是上面说的 m 去窃取）

# 触发调度

从标题来讲，什么叫触发调度，那我问第一个问题，调度是什么，在 golang 就是指调度函数 `schedule`，所以什么叫触发调度？指的是哪些能调用  `schedule` 函数 。

看图：

![](C:\Users\78478\Desktop\review\2020-02-05-15808864354679-schedule-points.png)

- 主动挂起 — [`runtime.gopark`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L287) -> [`runtime.park_m`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L2668)
- 系统调用 — [`runtime.exitsyscall`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L3074) -> [`runtime.exitsyscall0`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L3236)
- 协作式调度 — [`runtime.Gosched`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L267) -> [`runtime.gosched_m`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L2709) -> [`runtime.goschedImpl`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L2693)
- 系统监控 — [`runtime.sysmon`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L4455) -> [`runtime.retake`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L4569) -> [`runtime.preemptone`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L4666)

## 阻塞是怎么做到的

我们知道，调度的本质其实就是阻塞和运行，运行我们能够理解，选出一个 G 然后 `execute`，这个就是运行？问题是怎么让原来已经在运行的 G 停下来呢？

其实，你仔细想一想，是不是用新的 G 替换掉原来旧的 G 就是阻塞呢？或者我这么讲，当我执行 `execute`  的时候，原来的 G 还在继续运行吗？肯定没在运行啦，你每个 p 只有一个 G 可以在运行，也就是说实际上并没有什么停掉 G 的代码，当 `schedule` 函数执行到最后的 `execute`  的时候，就是在阻塞原来的 G 并且运行新的 G。

因为我新的运行了，意味着把旧的 G 给顶替掉了。

## 主动挂起

我又看了一下连接，它里面的意思好像是”挂起“代表着，不会被 `schedule` 再一次调度到，总之挂起就是无法再被调度到，除非你用 channel 做一些事情，否则什么事情不做，无法再次运行，这才能叫挂起，区别于阻塞，阻塞你完全有可能随便阻塞一会，实际上你啥都没干（你没做实际意义上解除阻塞的事情）一会后又轮到你运行了。

#### gopark 函数

https://blog.csdn.net/u010853261/article/details/85887948

[`runtime.gopark`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L287) 是触发调度最常见的方法，该函数会将当前 Goroutine 暂停，**被暂停的任务不会放回运行队列**，我们来分析该函数的实现原理： 

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	mp := acquirem()
	gp := mp.curg
	
    ...
    
    // 最重要
	mcall(park_m)
}

func park_m(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	schedule()
}
```

#### mcall 函数

首先，我们需要回答几个问题，你挂起协程（线程，进程），肯定要做的一件事请是什么？我们通常说的，要保存先尝，你不保存以后怎么恢复嘛。但是，你看无论是 `gopark` 还是 `park_m`，都没有有关现场保存的代码（不要说是我没贴出来，本来没保存）

这个任务就是 `mcall` 完成的，具体的实现这里没有，但是可以根据官方的注释，大概说一下它做了什么事：

1. `mcall` 从 g 切换到 g0 堆栈并调用 `fn(g)`，其中 g 是发出调用的 goroutine。
2. `mcall` 将 g 的当前 `PC / SP` 保存在 `g-> sched` 中，以便以后可以恢复。

#### park_m 函数

```go
func park_m(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	schedule()
}
```

两件事：

1.  将当前 Goroutine 的状态从 `_Grunning` 切换至 `_Gwaiting` 。
2.  调用 [`runtime.dropg`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L2571) 移除线程和 Goroutine 之间的关联。

好了，这里就有一个疑问了，你把 G 给移除了，又没有把它放到运行队列（全局和本地都没有 G 了），那我以后怎么恢复这个 G，我上哪去找它？？

其实调度函数 `schedule` 是真的从此就不调度这个 G 了，因为它已经不在调度这个模块了。

那它是怎么恢复的？总不能就把这个 G 给扔了把？详情看下面

#### goready

其实，`gopark`  本身就不是单独使用的，它总是要配合 golang 的其他特性一起完成工作，例如 channel。channel 的代码中会调用 `gopark` 挂起当前 G，上面也说了，G 不会放到调度的队列里面去了，但是它会被放到 channel 的接收队列或者发送队列（具体看我的 channel 实现的讲解）。

所以它的恢复，也是由 channel 负责，举个例子：

1. 我的一个 G 往一个无缓冲 channel 发送，这样会阻塞嘛，这个过程会调用 `gopark`。所以现在的 G 就被移除了（不在调度这个模块了），但是 channel 会把 G 放到自己的发送队列。
2. 接着我另外一个 G （不是上面的 G）去监听接收这个 channel，它就会把 channel 的发送队列里面的 G 拿出来，这个过程会执行到 goready。

所以，实际上，gopark 和 goready 总是一块配对使用，但是使用他们的往往不是调度这个模块，而是 golang 的其他特性，例如 channel。

```go
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}

func ready(gp *g, traceskip int, next bool) {
	_g_ := getg()

	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
}
```

## 协作式调度

我第一次看到这玩意惊呆了，卧槽居然还有这种东西。真的有主动让出处理器的函数（有并不稀奇，稀奇的居然能直接用），还是可导出的，也就是我们也可以调用的函数，牛逼。

```go
func Gosched() {
	checkTimeouts()
	mcall(gosched_m) // mcall 保存现场
}

func gosched_m(gp *g) {
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	casgstatus(gp, _Grunning, _Grunnable) // 设置 G 的状态为 _Grunnable
	dropg() // 移除 g 和 线程之间的关联
	lock(&sched.lock)
	globrunqput(gp) // 把 g 放到全局队列
	unlock(&sched.lock)

	schedule()
}
```

1. 同样要注意保存现场，切换到 g0，所以有 `mcall`。
2. 把 G 的状态设置成  `_Grunnable`。
3.  移除 g 和 线程之间的关联。
4. 把 g 放到全局队列。

## 系统调用

```go
#define INVOKE_SYSCALL	INT	$0x80

TEXT ·Syscall(SB),NOSPLIT,$0-28
	// 调用前准备
	CALL	runtime·entersyscall(SB)
	...
	INVOKE_SYSCALL // 系统调用
	...

	// 调用后的恢复
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	...
	CALL	runtime·exitsyscall(SB)
	RET
```

#### 调用前准备

```go
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	_g_.m.locks++
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	casgstatus(_g_, _Grunning, _Gsyscall)

	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.m.mcache = nil
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}
	_g_.m.locks--
}
```

1. 保存当前的程序计数器 PC 和栈指针 SP 中的内容。（保存现场）
2. 将 Goroutine 的状态更新至 `_Gsyscall`。
3. 将 Goroutine 的处理器和线程暂时分离并更新处理器的状态到 `_Psyscall`。（分离 p 和 m）

#### 调用后恢复

```go
func exitsyscall() {
	_g_ := getg()

	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
	if exitsyscallfast(oldp) {
		_g_.m.p.ptr().syscalltick++
		casgstatus(_g_, _Gsyscall, _Grunning)
		...

		return
	}

	mcall(exitsyscall0)
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}
```

两种方式恢复： `exitsyscallfast`  和   [`exitsyscall0`](https://github.com/golang/go/blob/cf630586ca5901f4aa7817a536209f2366f9c944/src/runtime/proc.go#L3151-L3179)  

#### exitsyscallfast

你想想，本来 p 和 m 分离了，现在要恢复，当然就是找 p 喽，只不过是找原来的 p 或者新的 p 的区别而已。

1. 如果 Goroutine 的原处理器处于 `_Psyscall` 状态，就会直接调用 `wirep` 将 Goroutine 与处理器进行关联；
2. 如果调度器中存在闲置的处理器，就会调用 `acquirep` 函数使用闲置的处理器处理当前 Goroutine；

#### exitsyscall0

好了，你又可以想想，现在找 p 的方法行不通，另一种方法就是跟原来一样，空闲 g 找 m。

将当前 Goroutine 切换至 `_Grunnable` 状态，并移除线程 M 和当前 Goroutine 的关联： 

1. 当我们通过 `pidleget` 获取到闲置的处理器时就会在该处理器上执行 Goroutine；（找空闲的 p 运行，只不过人家是有 m 的）
2. 在其它情况下，我们会将当前 Goroutine 放到全局的运行队列中，等待调度器的调度；（放全局队列）