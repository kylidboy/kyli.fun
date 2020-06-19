---
title: "Golang Select源码阅读笔记"
date: 2020-06-08
lastmod: 2020-06-08
description: 之前已经读过了"new三剑客"。尤其是其中的channel，既然看过了channel，那么select这个结构，是不可能放过的。平时写业务代码的时候，select用的熟不熟，可以看得出是不是一个老狗<i class="fa fa-dog"></i>。
featuredImage: /images/go-select/gopher-select.jpg
featuredImagePreview: /images/go-select/go-select-channels.png
tags:  
- golang
- runtime
- select
- 源码阅读 
categories:
- golang源码阅读 
---

<!--more-->
之前已经读过了"new三剑客"。尤其是其中的channel，既然看过了channel，那么select这个结构，是不可能放过的。平时写业务代码的时候，select用的熟不熟，可以看得出是不是一个老狗<i class="fa fa-dog"></i>。

唉，这select的阅读不可能不涉及到goroutine的部分，不过好在在channel的代码里我们已经看过一大部分了。race detector的部分都略过。

面试的时候很多面试官问的一个高频问题就是select的分支执行是顺序的还是随机的？

## 主要的常量和类型定义

吐嘈一下，go的常量定义真的是丑。

```go
// scase.kind values.
// Known to compiler.
// Changes here must also be made in src/cmd/compile/internal/gc/select.go's walkselectcases.
const (
	caseNil = iota
	caseRecv
	caseSend
	caseDefault
)

// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/internal/gc/select.go's scasetype.
type scase struct {
	c           *hchan         // chan
	elem        unsafe.Pointer // data element
	kind        uint16
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}
```

首先是定义了4个scase.kind常量，表示select分支(case)的类型。结合我们平时写go的经验，select里面的case就4个类型，顾名思义。然后是scase，编译器用来描述一个case用的struct。字段不多，`scase.c`用来保存我们在case里阻塞的读写channel。kind对应之前的常量。releasetime暂时不知道是干嘛的。

[温故知新的超链接Go Chan](https://kyli.fun/go-chan)

## select 的主体

整个select 结构只对应一个函数：`func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)`。所以我们就把整个代码打碎，在代码中直接以注释的形式来阅读吧，不再整段代码copy了。

先看注释，源码可以不看，注释一定要看。
```go
// selectgo implements the select statement.
//
// cas0 points to an array of type [ncases]scase, and order0 points to
// an array of type [2*ncases]uint16 where ncases must be <= 65536.
// Both reside on the goroutine's stack (regardless of any escaping in
// selectgo).
// cas0 是一个指向[ncases]scase数组类型的指针， order0是一个指向[2*ncases]uint16类型数组的指针，其中ncases必须小于等于65535，很明显uint16不能溢出，所以一个select里面只有最多65535个case，写那么多case，会被fired吧。
// 这两个变量都分配在goroutine的栈上（无视selectgo中的任何逃逸）（这句话我也不太理解，功夫还不够）。

// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// Also, if the chosen scase was a receive operation, it reports whether
// a value was received.
// selectgo 返回的是选中的scase的索引，该索引是按照select中的case语句申明的先后顺序。而且，如果选中的scase是一个receive操作，selectgo还会返回是否有值已经被接收了。
```

### 准备工作

编译器把所有的case语句转换成scase数组之后，传给selectgo，selectgo干的第一件事情，是重组两个排序：一个加锁排序和一个轮寻排序。

```go
	// NOTE: In order to maintain a lean stack size, the number of scases
	// is capped at 65536.
	// cas1和cas0没有啥区别，直接通过unsafe转换成一个数组的指针，方便下标访问。
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	// order1是一个2×ncases大小的数组，里面会存放两组cas1的下标队列。
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
	// 保持一个slice指向cas1
	scases := cas1[:ncases:ncases]
	// 这里是划分order1的空间，前半部分给轮寻排序，后半部分给加锁排序，两者的长度都是ncases。
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	// Replace send/receive cases involving nil channels with
	// caseNil so logic below can assume non-nil channel.
	// 先过滤channel为nil的cases，caseDefault是一个特例，要保留它。
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	// 大概是profile相关的
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
		for i := 0; i < ncases; i++ {
			scases[i].releasetime = -1
		}
	}

	// The compiler rewrites selects that statically have
	// only 0 or 1 cases plus default into simpler constructs.
	// The only way we can end up with such small sel.ncase
	// values here is for a larger select in which most channels
	// have been nilled out. The general code handles those
	// cases correctly, and they are rare enough not to bother
	// optimizing (and needing to test).

	// 编译器会把那些只有0或者1个case外加default case的select都重写成更加简单的结构。
	// 我们在这里会遇到sel.ncases非常非常小的唯一的情况是，在一个case比较多的select中，大部分分支的
	// chan都因为nil被排除掉了。但是我们的代码已经很好的兼容这种情况，而且这种情况本身也是足够罕见到不值得专门去优化。

	// generate permuted order
	// 在[]pollorder里填充随机化的轮寻的排序。
	// pollorder本来是[0, 1, 2, 3, ..... ncases-1]
	// 然后fastrandn用来打乱这个排序。这个可以回答面试官的问题吧。
	for i := 1; i < ncases; i++ {
		// 在[0, i+1)区间内快速获得一个随机整数j。
		j := fastrandn(uint32(i + 1))
		// 把pollorder[j]移到pollorder[i]，就等于在pollorder中空出了随机位置j。
		pollorder[i] = pollorder[j]
		// 然后把第i个case的索引i，放入pollorder。因为pollorder在进入for之前，保持的是全部的顺序序列，
		// 所以这里可以的uint16(i)就是原本pollorder[i]的值。
		// 整个逻辑就变成了遍历整个数组，每次循环都把当前值和之前的某一个随机位置的值交换位置。这样就完成了pollorder的随机化。
		pollorder[j] = uint16(i)
	}
	// 接下来，要排lockorder的序，lockorder的排序是有要求的，不像pollorder可以随机乱序。
	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	// 这里是采用堆排序，nlog n，来按照scase所包含的Hchan字段值的地址来生成加锁的顺序。
	// 根据后面的代码以及加解锁的逻辑，按照hchan的地址排序是因为chan可能出现两次，这样比较容易找到出现两次的chan，只加解锁一次。

	// lockorder里生成了一个最大堆
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		// 这里是根据已经被随机排列过的pollorder来遍历所有的cases
		// c保持一个当前case的游标
		c := scases[pollorder[i]].c
		// (j-1)/2是j的父节点，如果j的父节点排序高于c的排序，那么就认为，j的其他祖先的排序也都高于c的排序,
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			// j > 0只是一个保护。
			// k指向j的父节点。
			// 如果c的父节点的排序低于c的排序，那么就把c的父节点降到c的位置
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			// 把j指到被调整的父节点的位置，进入下一个循环，从新的j开始继续检查父节点的排序。
			j = k
		}
		// 直到j的排序满足j的排序小于j的父节点，把pollorder[i]放入lockorder[j]。
		// 因为这里的外循环是从0开始的，所以i在每一个位置上，内循环都可以保证看到一个c的排序小于自己父节点的排序，
		// 就可以认为c的所有的祖先节点的排序都比祖先的父节点要小。
		// 这里j指向的应该是最后一个被移动过的节点，也就是当前pollorder[i]应该存在的节点位置。
		lockorder[j] = pollorder[i]
	}

	// 对lockorder进行堆排序
	for i := ncases - 1; i >= 0; i-- {
		// 在[0, i]这个区间

		// 暂时保存当前游标访问的lockorder中的元素，也就是区间的末尾（将会被沉底的堆顶覆盖）。
		o := lockorder[i]
		// c始终都是lockorder当前循环访问的元素所指向的scase所包含的hchan
		c := scases[o].c
		// 把堆定元素放到当前区间的末尾，堆顶即待排序元素的最大值，让最大值沉底。
		lockorder[i] = lockorder[0]

		// j指向堆顶，再进行一个最大堆的构造
		j := 0
		for {
			// k是左子节点，k+1是右子节点
			k := j*2 + 1
			if k >= i {
				// break内循环
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				// 如果k+1<i，并且左子节点的排序小于右子节点的排序，k++
				// 为了让k取子节点中排序较大的那个。
				k++
			}
			// 因为这个已经是最大堆，所以堆顶空出来之后
			// 要做的事情就是把两个子节点中交大的那个和c做比较
			// 如果c大，那么直接用c作为新的堆顶，一切就都恢复到了最大堆，可以进行下一个沉底动作了。
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				// 如果c小，那么把较大的字节点移动到堆顶
				lockorder[j] = lockorder[k]
				// 进入被移走堆顶的子堆
				j = k
				// continue，继续在被移走堆顶的子堆里面做相同的动作
				continue
			}
			break
		}
		// 这里的j就指向了，这个区间[0, i]原本的末尾元素，在堆顶沉底之后应该在的位置，从而继续保持最大堆，进入下一个循环。
		lockorder[j] = o
	}
	// 到此，我们就拥有了两个不同的scase的排序队列：lockorder和pollorder。他们会在各自的战场发光发热。
```

### 加锁解锁

```go
func sellock(scases []scase, lockorder []uint16) {
	var c *hchan
	for _, o := range lockorder {
		c0 := scases[o].c
		// lockorder是根据scase.c的地址进行排序的，
		// 如果case中有两个case分别对同一个chan进行send/recv
		// 这两个case在排序后会排在一起，这个时候，我们只能在chan第一次出现的时候加锁并跳过第二次出现。
		// 在解锁的时候重复这个动作。
		if c0 != nil && c0 != c {
			c = c0
			lock(&c.lock)
		}
	}
}

func selunlock(scases []scase, lockorder []uint16) {
	// We must be very careful here to not touch sel after we have unlocked
	// the last lock, because sel can be freed right after the last unlock.
	// Consider the following situation.
	// First M calls runtime·park() in runtime·selectgo() passing the sel.
	// Once runtime·park() has unlocked the last lock, another M makes
	// the G that calls select runnable again and schedules it for execution.
	// When the G runs on another M, it locks all the locks and frees sel.
	// Now if the first M touches sel, it will access freed memory.

	// 这段不是很明白，可能要等看过GPM调度之后才会明白。
	// 这里我们必须十分小心，在我们释放了最后一个锁以后，就不能再触碰sel。
	// 因为sel有可能在我们释放最后一个锁之后立马也被释放了。
	// 设想如下场景
	// 当第一个M在runtime.selectgo()里面调用runtime.park()，并传入sel。
	// 一旦runtime.park()释放了最后一个锁，立马有另一个M标记刚才调用select的G为可运行状态并调度G来执行。
	// 当那个G运行在了另一个M上的时候，G会给所有的锁上锁，并释放掉sel。
	// 那么这个时候如果第一个M访问到了sel，它将会访问到已经释放的内存。

	// 顺序加锁，倒序解锁
	for i := len(scases) - 1; i >= 0; i-- {
		c := scases[lockorder[i]].c
		if c == nil {
			break
		}
		// 因为倒序，所以在重复chan出现的时候，在第二次解锁。
		if i > 0 && c == scases[lockorder[i-1]].c {
			continue // will unlock it on the next iteration
		}
		unlock(&c.lock)
	}
}
```

### 是时候开始真正的表演了

一大段代码，在go里面这么密集的使用`goto`真的好吗？不过，大神们留下的代码，还是很值得细品，你品，你慢慢品。

```go
	// 一堆变量，后面会有用的吧。
	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
	// pass 1 - look for something already waiting
	// 第一步，试图找到已经就绪的case
	var dfli int
	var dfl *scase
	// scase的数组下标
	var casi int
	// scase的指针，casi对应的scase
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		// 这里可以看到所有的scases的遍历都是根据pollorder里的排序。
		casi = int(pollorder[i])
		cas = &scases[casi]
		// 获得当前的scase的chan
		c = cas.c

		switch cas.kind {

		case caseNil:
			// 直接pass掉不包含send/receive的case
			continue

		case caseRecv:
			// 如果当前是一个recv操作
			// 就从当前case的chan的发送等待队列中，尝试出队一个sudog
			sg = c.sendq.dequeue()
			// 如果chan当前有阻塞的send，那么就从send等待队列获得一个sudog，那么直接跳到recv
			if sg != nil {
				goto recv
			}
			// 如果没有sudog，检查chan的qcount，判断是否是带缓冲的chan，直接到bufrecv
			if c.qcount > 0 {
				goto bufrecv
			}
			// 如果chan已经被关闭，recv的操作可以立即返回，直接到rclose
			if c.closed != 0 {
				goto rclose
			}
			// 都没有满足，那么这个recv操作继续阻塞

		case caseSend:
			// 如果当前是一个send操作
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			// 如果chan被关闭了，对一个关闭的chan写是会panic的，直接到sclose
			if c.closed != 0 {
				goto sclose
			}
			// 如果chan当前有阻塞的read，那么尝试从chan的读取等待队列出队一个sudog
			sg = c.recvq.dequeue()
			// 如果获得了一个sudog，直接跳到send
			if sg != nil {
				goto send
			}
			// 如果是带缓冲的chan并且缓冲还没有填满，直接到bufsend
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
			// mark default case
			dfli = casi
			dfl = cas
		}
	}
	// 如果default case存在，并且当前没有其他的send/recv就绪，那么select的执行就掉到default上。
	if dfl != nil {
		// 解锁
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		// 跳到retc，这个时候没有recv执行，所以recvOK = false
		goto retc
	}

	// 没有recv/send就绪，也没有defaultcase
	// 就要第二次遍历scases，把所有的case对应的hchan都处理掉。
	// pass 2 - enqueue on all chans

	// 获得当前的goroutine指针gp
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		// 跳过不是send/recv的case
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		// 获得一个sudog，一个当前goroutine在阻塞等待队列的代理。
		sg := acquireSudog()
		// 代理gp
		sg.g = gp
		// mark当前等待是在select
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		// 用nextp把所有的sudog构建成一个list
		*nextp = sg
		nextp = &sg.waitlink

		// 根据send/recv塞进相应的等待队列
		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// gopark让goroutine让出执行，等待被别的goroutine唤醒
	// wait for someone to wake us up
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	gp.activeStackChans = false

	// 加锁，在我们被唤醒之后，hchans不能再有变化。
	sellock(scases, lockorder)

	gp.selectDone = 0
	// 重建唤醒我们的sudog
	sg = (*sudog)(gp.param)
	gp.param = nil

	// 这里一定有至少一个chan发生就绪

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	// 从gp.waiting里拿到之前构造的sudog的list
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	// 把等待的sudog的单向链表做一些整理操作，关联的chan和元素都置为nil
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	// 按照lockorder的顺序遍历，我们在构造sudog waiting list的时候，也是按照lockorder的顺序构造的。
	// 所以我们这里可以按sudog waiting list的顺序，获得scase。
	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		// sg 就是唤醒我们的sudog，这时与之对应的scase和下标分别就是k和casei。
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			// 简单的把sglist指向的sudog从hchan的阻塞队列里移除。
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		// 释放sglist指向的sudog，把sglist指向列表的下一个元素
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	// 这里主要是代码重用，当select中的chan因为关闭，而把我们唤醒了，这个时候cas就会等于nil，
	// 这时代码跳到loop，就足以处理这种情况，所以就直接跳到loop了。
	if cas == nil {
		// We can wake up with gp.param == nil (so cas == nil)
		// when a channel involved in the select has been closed.
		// It is easiest to loop and re-run the operation;
		// we'll see that it's now closed.
		// Maybe some day we can signal the close explicitly,
		// but we'd have to distinguish close-on-reader from close-on-writer.
		// It's easiest not to duplicate the code and just recheck above.
		// We know that something closed, and things never un-close,
		// so we won't block again.
		goto loop
	}

	// 保持一个当前操作的scase的hchan的句柄
	c = cas.c

	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " kind=", cas.kind, "\n")
	}

	// 那么现在我们有了唤醒我们并且就绪的send/recv scase分支

	// 如果是recv，mark recvOK = true
	if cas.kind == caseRecv {
		recvOK = true
	}

	if raceenabled {
		if cas.kind == caseRecv && cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
		} else if cas.kind == caseSend {
			raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
		}
	}
	if msanenabled {
		if cas.kind == caseRecv && cas.elem != nil {
			msanwrite(cas.elem, c.elemtype.size)
		} else if cas.kind == caseSend {
			msanread(cas.elem, c.elemtype.size)
		}
	}

	selunlock(scases, lockorder)
	// 跳到retc，可以返回了，返回值都已经准备就绪了。
	goto retc

bufrecv:
	// can receive from buffer
	// 有recv的chan就绪
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
		}
		raceacquire(chanbuf(c, c.recvx))
		racerelease(chanbuf(c, c.recvx))
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.size)
	}
	// 标记有receive
	recvOK = true
	// 根据`c.recvx`接收索引指向，拿到chan的缓存槽
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		// 如果有变量等待接收消息，而不是丢弃，那么就typedmemmove把消息从缓存槽copy到cas.elem
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	// 清除缓存然后移动recvx
	typedmemclr(c.elemtype, qp)
	c.recvx++
	// recvx必要的roundtrip
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	// 减小buf计数器
	c.qcount--
	selunlock(scases, lockorder)
	// 可以return了
	goto retc

bufsend:
	// can send to buffer
	// 有buffered chan就绪
	if raceenabled {
		raceacquire(chanbuf(c, c.sendx))
		racerelease(chanbuf(c, c.sendx))
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	// 根据c.sendx指向的发送缓存操，把cas.elem塞进去
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	// 移动sendx
	c.sendx++
	// sendx的roundtrip
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	// 增加缓存计数器
	c.qcount++
	selunlock(scases, lockorder)
	// 可以返回了
	goto retc

recv:
	// can receive from sleeping sender (sg)
	// chan上有阻塞的send操作，要么是一个不带buffer的要么是一个带buffer但是buffer塞满了的chan
	// func recv处理这些情况，正确的接收chan上的消息
	// 可以参考chan.go里面的recv注释
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	// mark
	recvOK = true
	goto retc

rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc

send:
	// can send to a sleeping receiver (sg)
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

retc:
	if cas.releasetime > 0 {
		blockevent(cas.releasetime-t0, 1)
	}
	return casi, recvOK

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```