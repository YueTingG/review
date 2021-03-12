## context的实现原理

#### context 的作用

一般来讲，结合使用场景来讲比较符合人的思维，你想想嘛，别人问你它的作用，你一般的思考都是想想自己平时怎么用它的，然后讲出来。

这里推荐伴鱼里面的场景 https://xie.infoq.cn/article/3e18dd6d335d1a6ab552a88e8 其他部分也可以看看

我觉得比较符合我自己的理解，网上其他解答都让我摸不着头脑。 

场景：

1. 请求链路传递参数（值）
2. 超时控制

**高大上**：通过类似于超时的设置，达到控制 goroutine 的生命周期的目的。

#### 差 context.Background()

```
	ch := make(chan int)
	go func() {
		time.Sleep(time.Second * 3)
		fmt.Println("wake up")
		close(ch)
	}()
	x := <-ch
	fmt.Println(x)
```

首先看上面的一串代码，x一直无法执行，因为里面没有数据，但是 goroutine 如果执行close后，x 能否从channel 中读出数据呢？答案是不能，但是读取 channel ，`x := ch` 这一行却不会变得阻塞，可以看以下代码

```
	ch := make(chan int)
	close(ch)
	x, ok := <-ch // 因为关闭而不再阻塞，但是却读不出数据，如果输出x相当于输出默认值
	if !ok {
		fmt.Println("!ok")
	} else {
		fmt.Println(x)
	}
	-------------------
	最终输出是“!ok”
```

**为什么要提这个呢？因为这个就是cancelCtx的cancel机制**

context 内部有一个 done 的 channel，它是怎么通知别的协程 cancel？答案就是直接把 done 给 close 了，然后别线程发现，哎呦 done 不阻塞了，那肯定是cancel了。

#### cancelCtx

```
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

##### children

内部有两个比较重要的字段，一个是 children 里面装着全部的 children，调用 cancel 的时候，会把所有的childre取出来然后所有的 children 都调用 cancel，也就是递归调用，保证所有的子都cancel掉了。

##### done

就是一个channel，怎么通知别的线程同步的，答案就是我上面说的，补充以下，就是调用cancel的时候，其实就是把done给close，所以实际上cancel就是把自身的done给close，也把所有的children的done给close。

```
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	//把自身的done关闭
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	//递归调用children关闭done
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

#### timerCtx

##### 数据结构

```
type timerCtx struct {
	cancelCtx // 从这里可以看出来它的 cancel() 是基于 cancelCtx，剩下的就是多出了 timer。
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

##### cancel

对应的就是 WithDeadline 函数，首先 cancel 里面的通知机制跟 cancelCtx 一样，都是关闭 done，同时 timerCtx 内部还有一个 timer，cancel也会关闭 timer，减少不必要的浪费。

##### 定时 cancel

这个才是最重要的，到底 timerCtx 是怎么定时 cancel，我们已经知道了主动 cancel 的结果，就是关闭 done 然后关闭定时器。

在初始化的时候也就是调用 WithDeadline 的时候，就会生成一个定时器里面需要传入超时时间以及一个超时后执行的函数，timerCtx传入的函数就是cancel函数，所以相当于定时器内部会帮timerCtx检查现在是否超时，如果超时则调用cancel函数。

<u>至于定时器本身是怎么实现不在讨论范围，所以实际上timerCtx并不复杂</u>

##### 总结

1. 它的 cancel 本身就是关闭 done，在 cancelCtx 的基础上多关闭了定时器减少浪费。
2. 然后最终就是定时问题，这个部分最终的就是检查当前时间跟超时时间，这件事本身却是定时器去完成的，timerCtx 只是创建了这么一个结构体，然后传入一个超时后执行函数，这个执行函数就是 cancel。保证超时后关闭 timerCtx。

##### 代码

```
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // 如果已经超时那么将cancel
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		//大头，new一个定时器，定时器会去不断检测，如果超时就调用c.cancel函数
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

####  WithTimeout 

不用说了，本身就是调用 WithDeadline 函数的，代码如下：

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

把 timeout 加在现在的时间上，然后传给 WithDeadline 就完事了。

#### valueCtx

```
type valueCtx struct {
	Context
	key, val interface{}
}
```

在context的基础上多存了一对 key-val，取数据也挺简单的，就是比较以下key是否相同，如果相同则返回 val，如果不同则调用 context 的 value，最原始的Background返回的是nil，所以最少返回nil，也是递归向父级寻找是否有key，如果没有最后返回nil（最开始是Background的情况下）。

```
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

##### 注意

一个 valueCtx 只能存储一对 kv，要多存储 kv，只能通过创建多个 valueCtx。

## 位移运算符

```
fmt.Printf("%08b\n", a<<1)
fmt.Printf("%08b\n", a<<2)
fmt.Printf("%08b\n", a<<3)

fmt.Printf("%08b\n", a>>1)
fmt.Printf("%08b\n", a>>2)
```

结论：要被移位的量是在左边，例如1<<2，是指1被左移2位。x

## 互斥锁实现原理

 https://purewhite.io/2019/03/28/golang-mutex-source/  主要是看它的注释一点点明白的

 https://zhuanlan.zhihu.com/p/75263302 

 https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#mutex 

基础概念我就不讲了，什么有几种状态之类的，链接里都有

#### 数据结构

```
type Mutex struct {
	state int32 // 最有用，最难理解，处理锁的状态
	sema  uint32 // 这个主要是用在信号上，就是唤醒协程
}
```

### 获取锁

#### 到底有哪些获取锁的地方

看互斥锁的代码的时候，对于 lockSlow 函数中主要有两个地方可以获取锁，

```go
 if atomic.CompareAndSwapInt32(&m.state, old, new) {
        // 如果说old状态不是饥饿状态也不是被获取状态
        // 那么代表当前goroutine已经通过CAS成功获取了锁
        // (能进入这个代码块表示状态已改变，也就是说状态是从空闲到被获取)
        if old&(mutexLocked|mutexStarving) == 0 {
            break // locked the mutex with CAS
        }
        // ...
  }
```

上面这个地方，主要是给非饥饿状态下，一种是新来的goroutine，另一种是被唤醒的goroutine来获取锁的，下文会用`第一个代码`来称呼上面获取锁的代码。

另外一个地方获取锁，下文会用`第二个代码`来称呼下面获取锁的代码。

```
		// 如果说锁现在是饥饿状态，就代表现在锁是被释放的状态，当前goroutine是被信号量所唤醒的
        // 也就是说，锁被直接交给了当前goroutine
        if old&mutexStarving != 0 {
            // 如果说当前锁的状态是被唤醒状态或者被获取状态，或者说等待的队列为空
            // 那么是不可能的，肯定是出问题了，因为当前状态肯定应该有等待的队列，锁也一定是被释放状态且未唤醒
            if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                throw("sync: inconsistent mutex state")
            }
            // 当前的goroutine获得了锁，那么就把等待队列-1
            delta := int32(mutexLocked - 1<<mutexWaiterShift)
            // 如果当前goroutine非饥饿状态，或者说当前goroutine是队列中最后一个goroutine
            // 那么就退出饥饿模式，把状态设置为正常
            if !starving || old>>mutexWaiterShift == 1 {
                delta -= mutexStarving
            }
            // 原子性地加上改动的状态
            atomic.AddInt32(&m.state, delta)
            break
        }
```

这个地方主要是给那些饥饿状态下唤醒的goroutine获取锁的。

#### 锁的获取

结合我上面列举的获取锁的两种方式

1. 正常状态，锁可以被一种是新来的goroutine，另一种是被唤醒的goroutine获取，就是第一个代码，要注意的是，第一个代码中会判断当前锁是不是饥饿状态，如果是饥饿状态，那么这两种goroutine都会进入阻塞队列，区别就是进入曾经被唤醒的进入队头或者新来的进入队尾。
2. 饥饿状态，可以在第二个代码中获取锁，从状态也可以看出来，要求必须是饥饿状态，这时候会发送信号量，本来阻塞的协程就会被唤醒，这时候加上锁是饥饿的，那么这个被唤醒的协程就会获取锁。（目前不太确定，是不是这种方式唤醒的协程，这时候锁一定是已经被释放了，还是说锁有可能被锁住）

总结：

1. 第一个代码获取锁的条件是不饥饿（筛选协程）。
2. 第二个代码获取锁的条件是饥饿。

#### 锁的获取条件

首先，必须先说明一点，虽然老是在强调饥饿状态，但是本质上锁能不能被唤醒，是看这个锁是不是被锁住的状态mutexLocked（1），还是被释放的状态（0），所谓的饥饿状态只是筛选现在的协程能不能获取锁。

什么意思呢？举例说明，如果现在锁确实被释放了，现在你是新生的或者被唤醒的协程，来到第一个代码前面，本来你是可以获取锁的，但是现在假如锁是饥饿状态，那么不好意思，锁不能给你们？你可能会说“不能给新生的我可以理解，会什么不能给被唤醒的协程”？

实际上此"被唤醒的协程"不是那个“被唤醒的协程”，被唤醒的协程在饥饿状态下能获取锁，那指的是，你在从阻塞队列里面出来的时候，锁是饥饿的，也就是代码是从第一个代码到第二个代码之间的

```
        runtime_SemacquireMutex(&m.sema, queueLifo)
        
        starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
```

这里被唤醒时，如果在这里锁是饥饿的，那么你走到第二个代码，自然就会获取锁。

但现在问题是，被唤醒的协程的代码居然走到了第一个代码（就是回到了开头），说明什么，说明你刚刚被唤醒的时候，锁并不是阻塞的，所以你重新循环走到了开头一遍，这时候锁的阻塞已经跟你无关了，我们当你是“新生的协程”一样处理。

#### 两种饥饿

实际上从代码中理解，可以看出是有两种饥饿状态，一种是锁的饥饿状态，一种是协程饥饿状态。

我接下来说说这两者的关系

1. 确实是由协程饥饿状态来决定锁的饥饿状态。如果协程本来就是饥饿状态，或者 一旦 Goroutine 超过 1ms 没有获取到锁。

   ```
   		// 设置协程的饥饿状态
           starving = starving || runtime_nanotime()-waitStartTime >starvationThresholdNs
   ```

2. 但是这里有一个误区，那就是当协程是饥饿的时候，锁其实并不会马上就被设置成饥饿，什么意思？我们可以从设置饥饿这段代码一直往下看，都没有发现，代码根据现在协程饥饿把锁设置成饥饿，那什么时候设置饥饿？答案就是代码循环从开头开始，我们可以看到：

   ```
   //开头把new设置成饥饿
   if starving && old&mutexLocked != 0 {
           // 期望状态设置为饥饿状态
           new |= mutexStarving
       }
   // ...
   
   //到这里把state设置成new，也就是new如果是饥饿，那么现在state也是饥饿
   if atomic.CompareAndSwapInt32(&m.state, old, new) {
          
           if old&(mutexLocked|mutexStarving) == 0 {
               break // locked the mutex with CAS
           }
   // ...
   ```

3. 所以实际上又可以得出一个结论，你协程从阻塞队列里面出来后发现自己饥饿了，但是你无法立马把锁设置成饥饿，导致你在第二个代码那里是无法获取锁。但是你循环回到最开始，从第一个代码那里，因为old一定不是饥饿（注意此时协程是饥饿的，经过锁的状态设置，现在锁也是饥饿的，但是比较的时候比较的是old，old不是饥饿，记住，锁确实已经饥饿了，但是比较的时候比较的是old），如果锁没有被锁定的话你也是可以获取到锁。（第一个代码和第二个代码要获取锁的条件看上面[锁的获取](####锁的获取)）

4. 本质上就是因为你协程饥饿的时候，无法立马就把锁设置成，必须循环到开头（很狗）。

#### 互斥锁的主要流程

我们接下来概括性地讲讲互斥锁的主要流程：

1. **自旋**。首先先判断能否自旋，自旋本质就是不阻塞，但是占用cpu，能更好的获取锁，这时候能获取锁的协程有两种，一种是新生的协程，一种从队列中出来，但是一开始锁并没饥饿，所以它也没有获取到锁，对于这样的协程它只能从第一个代码跟新生的一起竞争。
2. 竞争主要有两点，第一谁走到第一个代码的时候，锁是被释放的，也就是谁先走到第一个代码，第二锁不能是饥饿的。
3. **锁的状态设置**。我们从流程上忽视上面第2点（第2点只是一个补充），现在自旋完了，（或者没自旋），我们需要根据当前锁的状态，设置新的锁的状态，为什么要设置新状态，原来是什么状态，现在就是不行？？锁是否锁定释放这个不讲，如果当前无法获取锁，那么就要在 `mutexWaiterShift` 上++，毕竟又阻塞了一个，如果当前协程是饥饿的，那么要设置 `mutexStarving`  （唤醒那个我不太懂），所以不能原来是什么现在还是什么，原来阻塞30个协程，现在这个协程也要阻塞，你就要++，原来锁不是饥饿，但是现在协程饥饿，你要把锁变成饥饿的。
4. **尝试获取锁**。接下来，我们就走到了`第一个代码那里`，还记得第一个代码的条件，锁不能是饥饿，所以带入条件，如果锁不饥饿，那么老生跟新生竞争，谁先到（就是谁比较的时候锁的 `mutexLocked`  是0），谁就拿到锁。如果现在锁饥饿，那么不好意思，协程入队列（不同的是，新生的到尾巴去，老生入队头），然后阻塞，等待唤醒。
5. **被唤醒**。被唤醒之后吗，先判断一下当前协程是否饥饿，但是并不立马设置锁的饥饿状态（后面也会判断是否删除锁的饥饿状态）。被唤醒的协程有两种可能，一种是锁是饥饿，那么不用怀疑了，你这个协程就是天选之子，你去获取锁。如果锁不是饥饿，不好意思你就是老生，你就要循环1~4走一遍，然后在4这里去跟新生竞争，竞争过是最好，竞争不过你就要继续阻塞，不过阻塞着阻塞迟早你就要变成饥饿，然后把锁也变成饥饿，那么下一次你从队列里出来，你就是天选之子了。

### 锁释放

#### 锁释放要做什么

首先，我们谈代码之前，我们要明确，锁释放要做什么？

1. 把state的锁状态位置成0。

2. 看看需不需要唤醒协程，两种情况要唤醒协程：

   1. 协程中没有woken（就是没人在那里自旋了），所有的协程都在阻塞队列里面，这时候需要从队列中~~随机~~（不确定）选取一个协程出来。

      我们说一下一些不能唤醒的情况，如果已经有协程醒了，**就是有人在那里自旋**，**阻塞队列中没有协程**（废话没有协程你唤醒个毛？），**锁又锁住了**（说明又有协程获取到了锁），**锁是饥饿的**（不是很能理解），这些情况属于不能唤醒协程。

   2. 锁是饥饿的，直接从队列的头中获取协程。

## 读写锁实现原理

*网上绝大部分人的解释都是错的，或者存在错误，而且他们还喜欢拷贝别人的错误！！！！*

#### 数据结构

```
type RWMutex struct {
	w           Mutex  // 互斥锁，写锁需要它
	writerSem   uint32 // 用来阻塞写操作和唤醒写操作的信号
	readerSem   uint32 // 用来阻塞读操作和唤醒读操作的信号
	readerCount int32  // 现在有多少个读操作
	readerWait  int32  // 现在这个写操作要等多少个读操作
}
```

我他妈的必须先说两句顺带解释，网上一堆人解释readerWait是错的或者语文有问题，让人搞不懂他想表达什么意思？倒是是写操作的数量还是读操作的数量

正确含义，我一开始也在想`readerCount`和`readerWait`不是重复了吗？不是的，存在一种场景，当前已经有写操作表示自己在等待了，那么从之前到现在为止有x个读操作（即有x个协程获取到读锁），所以当前的写的readerWait值是x，但是写在等的过程中又不断有新的协程要读，这时候readerCount是变化的在增加，但是对于之前的写操作来讲readerWait不变，所以两者不等同。

readerWait记录的是当前的锁需要等待多少个协程完成读操作。

依次来讲解那几个函数，race那些先不讲，看看以后能不能有机会补上。

#### RLock（加读锁）

```go
func (rw *RWMutex) RLock() {
	// ...
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
	//...
}
```

1. 首先把readerCount加一，这个很容易理解，因为有协程读，所以++。但是这里怎么会小于0呢？因为加写锁的时候，会让整个readerCount减去一个很大的数，瞬间这个数就变成负值，**所以当readerCount变成负值的时候，说明有写操作已经在等了**（判断当前是否有写操作在尝试获取锁的重要条件，要注意）。

2. 重点强调一下，只是有写操作在等，不是说就已经轮到写操作了，有可能现在还是之前的读操作在执行，所以有写操作排在你读操作前面，你自然要阻塞等待，这里就是调用 runtime_Semacquire 函数。

3. 这个方法是一个runtime的方法，会一直等待传入的s出现>0的时候
   然后我们可以记得，这里有这样一个情况，当出先readerCount + 1为负数的情况那么就会被等待，看注释我们可以猜到，**是当有写入操作出现的时候，那么读操作就会被等待。** 

#### RUnlock（释放读锁）

```
func (rw *RWMutex) RUnlock() {
	// ...
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		if r+1 == 0 || r+1 == -rwmutexMaxReaders {
			race.Enable()
			throw("sync: RUnlock of unlocked RWMutex")
		}
		if atomic.AddInt32(&rw.readerWait, -1) == 0 {
			runtime_Semrelease(&rw.writerSem, false)
		}
	}
	// ...
}
```

1. 首先把当前表示所有读操作的readerCount进行--，因为读操作完成了1个。然后就是r为什么会小于0的问题，跟上面一样，因为这时候已经有一个写操作了，这个写操作把readerCount变成了一个负数，所以如果r小于0，代表现在有写操作了。
2. `r+1 == 0 || r+1 == -rwmutexMaxReaders`这一段的目的是防止你没锁过就解锁，不深入讲。
3. 接着也很容易想到，我现在释放读锁，那么readerWait就要--，因为我现在这个读操作已经完成了（注意上面说的减去一个很大的数是readerCount，不是readerWait，不要混了）
4. 而readerWait代表的是写操作要等多少个读操作释放锁，现在假如readerWait为0，说明写操作要等的全部都结束了，所以**我如果是最后一个读操作**，**我有义务去唤醒阻塞中的写操作**，就是要唤醒想要进行写操作却阻塞的协程。所以调用`runtime_Semrelease(&rw.writerSem, false)`。

#### Lock（加写锁）

```
func (rw *RWMutex) Lock() {
	// ...
	rw.w.Lock()
	
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
	// ...
}
```

1. 首先操作最前面说的互斥锁，目的就是处理多个写锁并发的情况，因为我们知道写锁只有一把。这里不需要深入互斥锁，只需要知道，互斥锁只有一个人能拿到，所以写锁只有一个人能拿到。

2. 然后重点来了，这里的这个操作细细体会一下，`atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders)`
   是将当前的readerCount减去一个非常大的值`rwmutexMaxReaders为1 << 30`
   大概是1073741823这么大吧。

3. 所以我们可以从源码中看出，readerCount由于每有一个协程获取读锁就+1，一直都是正数，而当有写锁过来的时候，就瞬间减为很大的负数，所以我们可以发现当readerCount为负的时候，可以推断出现在有写操作在等待。
4. 然后做完上面的操作以后的r其实就是原来的readerCount，那么我现在要获取写锁，我自然要知道我要等多少个读操作，因为读操作会不断++，我只要我现在要等多少个，后面++不管我现在的事，所以要获取r，然后赋值给readerWait。
5. 后面进行判断，如果原来的readerCount不为0（原来有协程已经获取到了读锁）并且将readerWait加上readerCount（表示需要等待readerCount这么多个读锁进行解锁）后也不等于0，如果满足上述条件证明原来有读锁，所以暂时没有办法获取到写锁，所以调用runtime_Semacquire进行等待，等待的信号量为writerSem

#### Unlock（释放写锁）

```
func (rw *RWMutex) Unlock() {
	// ...

	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// 唤醒所有的读锁
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false)
	}
	// 释放mutex
	rw.w.Unlock()
	// ...
}
```

1. 上面lock的时候，把 readerCount 变成了一个负数，现在要先把他加回来。
2. 同样没上锁就解锁要报错
3. 唤醒所有读锁，这个因为写锁结束了，这个过程中肯定有很多读操作希望占用，所以都要唤醒。
4. 最后释放lock中上的互斥锁。

#### 原子操作的实现

在加读锁和写锁的工程中都使用atomic.AddInt32来进行递增，而该指令在底层是会通过LOCK来进行CPU总线加锁的， 通过汇编完成这个功能，所以说到底需要有一个地方是原子的。

## 怎么判断channel还有数据

1. 我跟发送方**定好长度**，超过某个长度，其实就是取数据的次数，我就不再去取，直接break

2. 用**定时器**，这里需要注意，定时器不用取全部数据的时间，而是上一次取和这一次取所消耗的时间，例如，我取完某一次数据后，下一次取数据的时候（因为我肯定是循环取监听channel），等了好久，这个好久就是定时器，超过某个时间我就break。

   ```
   func main() {
   	ticker := time.NewTicker(time.Second * 2)
   	ch := make(chan int, 1)
   
   	select {
   	case <-ch:
   		fmt.Println("get chan")
   	case <-ticker.C: // 记住要 .C
   		fmt.Println("time out")
   	}
   }
   ```

3. 另外就是需要对方在没有数据的时候close掉chan，这样我就可以使用ok来判断channel关闭了没有。

4. default

   ```
   func main() {
   	var x int
   	ch := make(chan int, 1)
   	select {
   	case x = <-ch:
   		fmt.Println(x)
   	default:
   		fmt.Println("n")
   	}
   }
   ```


## 怎么判断 channel 是否满了

```

func main() {
	ch := make(chan int)

	println(len(ch) == cap(ch)) // 判断长度跟容量是否一样了,如果满了是 true


	select {
	case ch <- 2: //如果满了,这个分子走不通
	default: // 会走这个分支
		println("Channel full. Discarding value")
	}
}

```

## 根对象到底是什么？

根对象在垃圾回收的术语中又叫做根集合，它是垃圾回收器在标记过程时最先检查的对象，包括：

1. 全局变量：程序在编译期就能确定的那些存在于程序整个生命周期的变量。
2. 执行栈：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针。
3. 寄存器：寄存器的值可能表示一个指针，参与计算的这些指针可能指向某些赋值器分配的堆内存区块。