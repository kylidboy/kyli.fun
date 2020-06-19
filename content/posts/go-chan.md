---
title: "Golang Channel源码阅读笔记"
date: 2020-05-29T11:09:25+08:00
lastmod: 2020-05-29
description: Golang的并发是一大特色。写起来真的大快人心。channel作为并发routine之间共享信息的主要手段，是gopher不可不知的杀器。
featuredImage: /images/go-chan/chan3.jpg
featuredImagePreview: /images/go-chan/chan2.jpeg
tags:  
- golang
- runtime
- chan
- 源码阅读 
categories:
- golang源码阅读 
---
<!--more-->

Golang的并发是一大特色。写起来真的大快人心。channel作为并发routine之间共享信息的主要手段，是gopher不可不知的杀器。眼睛长了麦粒肿，现在只能独眼龙看channel的源码。我还计划写一个select的源码阅读笔记，这个和channel的源码有着无比紧密的联系。

## 开篇注释
```go
// Invariants:
//  At least one of c.sendq and c.recvq is empty,
//  except for the case of an unbuffered channel with a single goroutine
//  blocked on it for both sending and receiving using a select statement,
//  in which case the length of c.sendq and c.recvq is limited only by the
//  size of the select statement.
//
// For buffered channels, also:
//  c.qcount > 0 implies that c.recvq is empty.
//  c.qcount < c.dataqsiz implies that c.sendq is empty.
```
代码的最开始就用注释给出了channel的“钻石恒久远一颗永流传”：`c.sendq`和`c.recvq`就像太阳和月亮，总有一个是空的。

## channel相关的定义
```go
const (
	maxAlign  = 8
	hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
	debugChan = false
)

type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog
    last  *sudog
}

func (q *waitq) enqueue(sgp *sudog) {
	sgp.next = nil
	x := q.last
	if x == nil {
		sgp.prev = nil
		q.first = sgp
		q.last = sgp
		return
	}
	sgp.prev = x
	x.next = sgp
	q.last = sgp
}

func (q *waitq) dequeue() *sudog {
	for {
		sgp := q.first
		if sgp == nil {
			return nil
		}
		y := sgp.next
		if y == nil {
			q.first = nil
			q.last = nil
		} else {
			y.prev = nil
			q.first = y
			sgp.next = nil // mark as removed (see dequeueSudog)
		}

		// if a goroutine was put on this queue because of a
		// select, there is a small window between the goroutine
		// being woken up by a different case and it grabbing the
		// channel locks. Once it has the lock
		// it removes itself from the queue, so we won't see it after that.
		// We use a flag in the G struct to tell us when someone
		// else has won the race to signal this goroutine but the goroutine
		// hasn't removed itself from the queue yet.
		if sgp.isSelect && !atomic.Cas(&sgp.g.selectDone, 0, 1) {
			continue
		}

		return sgp
	}
}
```

- `qcount`： 数据队列的计数器，如果是unbuffered，那么就永远都是0。
- `dataqsize`： 对于buffered channel，内部维护了一个环形队列，这个队列的长度，一般就是我们`make(chan, size)`对应的size。
- `buf`： 指向环形队列的指针。
- `elemsize`： 管道传送的数据结构的大小。
- `closed`： 管道是否关闭。
- `elemtype`： 指向传送数据的类型定义。
- `sendx`： 发送消息的缓存下标。
- `recvx`： 接受消息的缓存下标。
- `recvq`： 阻塞在管道的等待接收的sudog的waiting list。
- `sendq`： 阻塞在管道的等待发送的sudog的waiting list。
- `lock`： 保证chan是并发安全。

然后一个waitq的数据结构和它的enqueue,dequeue方法实现。

对于buffered channel，这里给一个buf的示意图。
![](/images/go-chan/chan_queue.png "管道的buffer")

## 创建channel

```go
//go:linkname reflect_makechan reflect.makechan
func reflect_makechan(t *chantype, size int) *hchan {
	return makechan(t, size)
}

func makechan64(t *chantype, size int64) *hchan {
	if int64(int(size)) != size {
		panic(plainError("makechan: size out of range"))
	}

	return makechan(t, int(size))
}

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

`makechan(t *chantype, size int)`就是我们写go代码时，`make(chan)`背后的英雄。这也可以看出，为什么我们自己写go代码时候以及面试被拷问的时候，`chan`为什么要用make来创建，而不是new。

- 从编译器传入的`*chantype`中拿出`elem`，channel中传递的数据结构。
- 两个边界检测。
- 检查管道的buffer会不会导致内存的溢出。
- 分配内存，根据不同的情况有一点优化。如果是无buffer的管道或者管道消息结构的大小是0，直接分配`hchan`的内存。如果管道消息的结构不含有指针，就在一个alloc中一起分配`hchan`和buffer的内存。其他情况下，`hchan`和对应的buffer各自分配内存。
- `elemsize`，`elemtype`，`dataqsiz`初始化。

## chan <- elem

好长好精彩的send来了。

```go
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

`chansend1(c *hchan, elem unsafe.Pointer)`函数只是`chansend`的一个简单的封装。还是直奔`chansend`吧。
- 如果无聊到给一个chan赋了nil，然后还对他<-，这就是结果。
- 快速检测如果send不要求阻塞，chan没有被关闭，并且没有准备好接收一个send动作，那么就直接返回false。
- 获取管道的mtx。接下来，任何显式的返回，都会释放这个锁。
- 安全检查，对一个已经close的管道，<-会导致panic。
- 接下来就是各种情况下的管道<-处理。
    1. 尝试从管道的等待接收队列里，获取一个之前block在接收消息的goroutine，如果获取成功，那么直接把消息丢给这个goroutine，并把这个之前阻塞的goroutine唤醒，然后chansend就返回true了。
    2. 如果`c.qcount < c.dataqsiz`，这就是两层含义，第一这是一个buffered管道，第二buffer还没有被填满。那么通过`chanbuf`函数和`c.sendx`，获得管道buffer的send槽，把消息直接copy到这个槽，发送就完成了。不存在阻塞的问题。只是要记得增加qcount和向前移动sendx。
    3. 到了这里，说明管道没有准备好接收一个send操作，原因有二，一是管道本身是一个没有buffer的并且没有goroutine在接收，二是这个buffered管道已经被塞满了。那么这时候看，block参数，如果不是true，那么就返回false。否则，就要准备被block。
- 调用`getg()`，获得一个指向当前执行send的这个groutine的指针gp。然后调用`acquireSudog`，获得一个sudog。我们可以忽略sudog的具体定义，只看sudog的宏观定义，了解获取这两个指针的充分必要性。sudog应该是类似于G的一个代理，一个G可以有许多个sudog，如下面的注释所言，一个G可以存在很多个等待队列，一个多对多的关系，所以利用sudog来解绑这种关系。sudog作为一个G的代理的职责就是存在于各个channel的收发等待队列里。在这里，我们要把我们的gp放到管道的send队列，就需要为这个gp做一个sudog，把两者之间建立好联系，然后把这个sudog存在sendq里面，~~等待有一天王子出现给它一个吻~~。（G指的是go runtime调度模型的GPM中的G，可以理解为一个物理上的go的runtime线程）。
```go
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
type sudog struct {
...
}
```
- 把sudog塞进sendq这个双向链表。
- gopark会把当前groutine挂起，让出调度。
- 然后就是无尽的长眠，~~直到王子出现~~。不知道过了多久，一个recv打破沉默把我们唤醒。
- 在长眠之前，我们设定了gp和mysg的关系，这个关系，在唤醒之后应该是保持原样。
- 查看gp.param，是否是channel关闭触发的唤醒，如果是，那就panic没商量。
- 这个时候我们做一些整理的工作，已经不需要的sudog，跟它说拜拜了。chansend可以返回true了。

在`chansend`中，如果我们找到了一个正在等待的routine，就会调用`send`直接把消息发给它。让我们看看`send`里面做了啥。

```go
// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```
首先，`send`函数接收一个`*hchan`(废话)，一个`*sudog`指向一个recv routine，ep指向发送的消息，unlockf函数。
- 如果`sg.elem`不为空，那么我们就调用`sendDirect`把消息丢给receiver，sudog不再持有指向消息的指针。
- 获得sudog代理的g
- 因为调用send之前，chan是被锁住的，这里要主动释放锁。
- 告诉g是，你sudog大爷解放的你。
- 通过goready唤醒g。大功告成。

## elem := <- chan

```go
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) {
		// After observing that the channel is not ready for receiving, we observe whether the
		// channel is closed.
		//
		// Reordering of these checks could lead to incorrect behavior when racing with a close.
		// For example, if the channel was open and not empty, was closed, and then drained,
		// reordered reads could incorrectly indicate "open and empty". To prevent reordering,
		// we use atomic loads for both checks, and rely on emptying and closing to happen in
		// separate critical sections under the same lock.  This assumption fails when closing
		// an unbuffered channel with a blocked send, but that is an error condition anyway.
		if atomic.Load(&c.closed) == 0 {
			// Because a channel cannot be reopened, the later observation of the channel
			// being not closed implies that it was also not closed at the moment of the
			// first observation. We behave as if we observed the channel at that moment
			// and report that the receive cannot proceed.
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}

// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

`chanrecv1`和`chanrecv2`都只是`chanrecv`的封装。老样子，直奔`chanrecv`，和`chansend`非常相似。注释一如既往，干货。`chanrecv`在channel上做接收的动作，并把接收到的消息写到`*ep`指向的内存。如果`*ep`为空，那么接收到的消息将被丢弃。
- 如果发现channel现在为空，调用接收会被阻塞，并且block为false，这个时候可以快速返回。
    1. 如果channel已经被close了，那么直接返回false， false
    2. 如果channel是空，那么清空*ep，然后返回true, false
- 锁住当前channel。
- 如果现在channel已经被close了并且channel里面已经没有缓存的消息，可以直接返回true, false
- 从此往下，channel都没有被close。
- 先尝试从channel的发送等待队列里拿sudog。如果拿到sg，那么调用`recv`。然后返回true, true。这个`recv`比对应的`send`要稍微复杂一丢丢，接收的时候要区分channel有没有buffer。
    - 如果channel没有buffer，那么如果ep不为空，我们就直接从sg代理的g上获取消息，保存到ep。
    - 如果是buffered，这就意味着channel当前的消息buffer被塞满了。这时的操作和上面是完全不同的。从channel的buffer里拿出recvx指向的消息槽，消费掉这个消息，如果ep不为空，消息会被拷贝到ep指向的内存。这时buffer就空了一个位置出来，我们就把sg代理的消息塞到刚才的buffer消息槽里。buffer又重新被填满。我们已经执行了一次recv，所以把recvx向前挪一个位置。同样也要维护sendx的位置，虽然是满buffer。也是向前挪一个。环形队列首尾相接，满。
    - 这是sg代理的消息，已经被处理了。sg就不需要再持有这个消息的指针了。
    - 取出sg代理的g。
    - 解锁channel。
    - 告诉g是从sg这个waiting里醒来的。
    - goready唤醒这个被我们操作过的阻塞在发送上的g。
- 执行到这里的话，判断`c.qcount`，如果是一个buffered管道，就说明buffer还有空余的槽。获取当前recvx指向的槽，消费掉这个消息，如果ep不为空，那么拷贝消息到ep指向的内存。然后清除槽。把recvx移动到下一个槽。qcount计数器也同步减小1。返回true, ture
- 到了这里，就是一个阻塞的接收操作了。除非block参数为false，直接返回false, false
- 和chansend的时候一样的创建一个sudog，塞到channel的recvq里面，让出调度，~~等待王子~~。
- 醒来之后，消息的接收已经被sender处理好了，查看一下channel有没有被关闭，如果gp.param不为空，说明我们被王子唤醒的，而不是因为channel被关闭。我们~~只要把衣服穿好~~跟sudog说拜拜。
- 返回true, !closed

## 关闭channel

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```
- 关闭一个nil的channel，panic没什么好说的。
- 锁住当前channel。
- 如果channel已经被关闭了，panic，没什么好说的，常识。
- 标记`c.closed` = 1。
- 循环拿出所有当前recvq队列里等待的sg，标记是channel关闭导致的唤醒。从sg里得到g，推到glist里。
- 循环拿出所有当前sendq队列里等待的sg，标记是channel关闭导致的唤醒。把相应的g推到glist里。(这些g最终会panic，常识)。
- 把glist里所有的g都设为就绪(唤醒)。

## 工具函数

```go
// chanbuf(c, i) is pointer to the i'th slot in the buffer.
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}

// full reports whether a send on c would block (that is, the channel is full).
// It uses a single word-sized read of mutable state, so although
// the answer is instantaneously true, the correct answer may have changed
// by the time the calling function receives the return value.
func full(c *hchan) bool {
	// c.dataqsiz is immutable (never written after the channel is created)
	// so it is safe to read at any time during channel operation.
	if c.dataqsiz == 0 {
		// Assumes that a pointer read is relaxed-atomic.
		return c.recvq.first == nil
	}
	// Assumes that a uint read is relaxed-atomic.
	return c.qcount == c.dataqsiz
}

// Sends and receives on unbuffered or empty-buffered channels are the
// only operations where one running goroutine writes to the stack of
// another running goroutine. The GC assumes that stack writes only
// happen when the goroutine is running and are only done by that
// goroutine. Using a write barrier is sufficient to make up for
// violating that assumption, but the write barrier has to work.
// typedmemmove will call bulkBarrierPreWrite, but the target bytes
// are not in the heap, so that will not help. We arrange to call
// memmove and typeBitsBulkBarrier instead.
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}

func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}

// empty reports whether a read from c would block (that is, the channel is
// empty).  It uses a single atomic read of mutable state.
func empty(c *hchan) bool {
	// c.dataqsiz is immutable.
	if c.dataqsiz == 0 {
		return atomic.Loadp(unsafe.Pointer(&c.sendq.first)) == nil
	}
	return atomic.Loaduint(&c.qcount) == 0
}
```

## select里的chan操作，神有特殊的安排

主要是`chansend`和`chanrecv`的封装，调用时block参数都是false，所以select不会因为channel而阻塞。

```go
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}
```

## TODO

```go
func chanparkcommit(gp *g, chanLock unsafe.Pointer) bool {
	// There are unlocked sudogs that point into gp's stack. Stack
	// copying must lock the channels of those sudogs.
	gp.activeStackChans = true
	unlock((*mutex)(chanLock))
	return true
}
```