---
title: Golang Timer源码阅读笔记
date: 2020-06-29
lastmod: 2020-06-29
description: Go的计时器也是一个挺有意思的部分，它的具体实现也是随着时间不断的在改进，我现在看的都是基于Go-1.14的源码。
featuredImage: /images/go-timer/timer.jpeg
featuredImagePreview: 
tags:
- golang
- runtime
- timer
- 源码阅读 
categories:
- golang源码阅读 
---
<!--more-->

Golang Runtime的计时器管理的实现主要有3个版本，我只看1.14的，之前的，等有空的时候再说吧，毕竟已经被历史的车轮滚过了。我只记得以前是有一个`timeproc`自己跑在一个goroutine的。现在，这个是被干掉了。

当前的(1.14+)计时器实现主要分布在3个文件里面：`go/src/time/sleep.go`, `go/src/time/tick.go`和`go/src/runtime/time.go`。
`wakeNetPoller`在`go/src/runtime/proc.go`里面。

有一个问题可能会有好事的面试官会问：Go的计时器的计时最小精度是多少？

```go
// Interface to timers implemented in package runtime.
// Must be in sync with ../runtime/time.go:/^type timer
type runtimeTimer struct {
	pp       uintptr
	when     int64
	period   int64
	f        func(interface{}, uintptr) // NOTE: must not be closure
	arg      interface{}
	seq      uintptr
	nextwhen int64
	status   uint32
}


// Package time knows the layout of this structure.
// If this struct changes, adjust ../time/sleep.go:/runtimeTimer.
type timer struct {
	// If this timer is on a heap, which P's heap it is on.
	// puintptr rather than *p to match uintptr in the versions
	// of this struct defined in other packages.
	pp puintptr

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr

	// What to set the when field to in timerModifiedXX status.
	nextwhen int64

	// The status field holds one of the values below.
	status uint32
}
```
这俩其实是一个东西，runtime真正管理的是这个数据结构。这个结构存活在`P.timers`，一个4叉堆。

![4叉堆](/images/go-timer/golang-timer-quadtree.png "忽略图上写的QuadTree")

## 小菜1：time/sleep.go

这个比较简单，代码不多，大部分都是注释

我们先看一下平时我们用到的那个`time.Timer`。这个结构很简单，只有一个`C <-chan Time`是外部可以访问的。这在我们平时运用计时器的时候也能感受到。创建的Timer我们只关心`<-Timer.C`。

```go
// The Timer type represents a single event.
// When the Timer expires, the current time will be sent on C,
// unless the Timer was created by AfterFunc.
// A Timer must be created with NewTimer or AfterFunc.

// Timer struct代表一个单独的事件。当Timer过期时，过期时的时间戳会丢到C里面，除非这个Timer是通过AfterFunc创建的。
// 外部代码必须通过NewTimer或者AfterFunc来创建Timer。
type Timer struct {
	C <-chan Time
	r runtimeTimer
}
```

然后就是一堆平时使用的Timer的相关操作了，很直白，直接上代码。

```go
// when is a helper function for setting the 'when' field of a runtimeTimer.
// It returns what the time will be, in nanoseconds, Duration d in the future.
// If d is negative, it is ignored. If the returned value would be less than
// zero because of an overflow, MaxInt64 is returned.
//
// when函数主要就是把这duration转成一个确定的naoseconds的时间戳，给runtimeTimer.when的。然后做了一些边界溢出的处理。
func when(d Duration) int64 {
	if d <= 0 {
		return runtimeNano()
	}
	t := runtimeNano() + int64(d)
	if t < 0 {
		t = 1<<63 - 1 // math.MaxInt64
	}
	return t
}

// 这些应该都是一群内联的函数，大概真正的实现在runtime里面。
// 精华都在这里面。
func startTimer(*runtimeTimer)
func stopTimer(*runtimeTimer) bool
func resetTimer(*runtimeTimer, int64) bool
func modTimer(t *runtimeTimer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr)

// NewTimer creates a new Timer that will send
// the current time on its channel after at least duration d.
//
// NewTimer创建一个新的Timer，调用startTimer把Timer.runtimeTimer交给runtime去管理。
// 这里看到注释特别说明了，'at least' duration d，说明golang里面的这个计时器不是德国造的。
func NewTimer(d Duration) *Timer {
	c := make(chan Time, 1)
	t := &Timer{
		C: c,
		r: runtimeTimer{
			when: when(d),
			f:    sendTime,
			arg:  c,
		},
	}
	startTimer(&t.r)
	return t
}

// Stop prevents the Timer from firing.
// It returns true if the call stops the timer, false if the timer has already
// expired or been stopped.
// Stop does not close the channel, to prevent a read from the channel succeeding
// incorrectly.
//
// To ensure the channel is empty after a call to Stop, check the
// return value and drain the channel.
// For example, assuming the program has not received from t.C already:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
//
// This cannot be done concurrent to other receives from the Timer's
// channel or other calls to the Timer's Stop method.
//
// For a timer created with AfterFunc(d, f), if t.Stop returns false, then the timer
// has already expired and the function f has been started in its own goroutine;
// Stop does not wait for f to complete before returning.
// If the caller needs to know whether f is completed, it must coordinate
// with f explicitly.
//
// 如果成功阻止了Timer的触发就返回true，如果Timer已经过时或者已经被Stop过了就返回false。
// 同时，Stop不会关闭这个管道。防止有block在这个管道上的read操作，因为关闭而错误的read成功。
// 对于一个Timer.Stop后确保其管道是空的，可以做一个drain的动作，但是这个动作不能和其他的读取操作并行。
// 如果一个Timer是AfterFunc创建的，那么Stop对于f是无感的，如果调用者需要知道f的执行情况，需要自己解决。
func (t *Timer) Stop() bool {
	if t.r.f == nil {
		panic("time: Stop called on uninitialized Timer")
	}
	return stopTimer(&t.r)
}

// Reset changes the timer to expire after duration d.
// It returns true if the timer had been active, false if the timer had
// expired or been stopped.
//
// Reset should be invoked only on stopped or expired timers with drained channels.
// If a program has already received a value from t.C, the timer is known
// to have expired and the channel drained, so t.Reset can be used directly.
// If a program has not yet received a value from t.C, however,
// the timer must be stopped and—if Stop reports that the timer expired
// before being stopped—the channel explicitly drained:
//
// 	if !t.Stop() {
// 		<-t.C
// 	}
// 	t.Reset(d)
//
// This should not be done concurrent to other receives from the Timer's
// channel.
//
// Note that it is not possible to use Reset's return value correctly, as there
// is a race condition between draining the channel and the new timer expiring.
// Reset should always be invoked on stopped or expired channels, as described above.
// The return value exists to preserve compatibility with existing programs.
//
// Reset和Stop类似，
// 注释最后一段说了，这个返回值是完全不能用的，返回值的存在只是为了向前兼容。
func (t *Timer) Reset(d Duration) bool {
	if t.r.f == nil {
		panic("time: Reset called on uninitialized Timer")
	}
	w := when(d)
	return resetTimer(&t.r, w)
}

// 这个sendTime是在Timer到期的时候，内部调用来将当前时间戳丢进管道里。主要是为了这个动作一定不被阻塞，而包了一层select。
// 在NewTimer，这个管道是必然不会阻塞的
// 在NewTicker的时候，这个周期性的发送操作也不能阻塞，多余的时间戳会直接被丢弃掉。
func sendTime(c interface{}, seq uintptr) {
	// Non-blocking send of time on c.
	// Used in NewTimer, it cannot block anyway (buffer).
	// Used in NewTicker, dropping sends on the floor is
	// the desired behavior when the reader gets behind,
	// because the sends are periodic.
	select {
	case c.(chan Time) <- Now():
	default:
	}
}

// After waits for the duration to elapse and then sends the current time
// on the returned channel.
// It is equivalent to NewTimer(d).C.
// The underlying Timer is not recovered by the garbage collector
// until the timer fires. If efficiency is a concern, use NewTimer
// instead and call Timer.Stop if the timer is no longer needed.
//
// After等待指定的时间长度流过，然后将当前的时间戳丢进返回的管道上。
// After等效`NewTimer(d).C`。
// GC对这个幕后的Timer是无感的，直到Timer过期。如果效率是一个重要的考量，那么请使用NewTimer并在不再需要这个Timer的时候调用Timer.Stop。
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}

// AfterFunc waits for the duration to elapse and then calls f
// in its own goroutine. It returns a Timer that can
// be used to cancel the call using its Stop method.
//
// AfterFunc等待指定的时间长度流过，然后在一个新的goroutine里面调用f函数。
// AfterFunc返回的是一个Timer，这个Timer可以通过调用Stop方法来取消。
func AfterFunc(d Duration, f func()) *Timer {
	t := &Timer{
		r: runtimeTimer{
			when: when(d),
			f:    goFunc,
			arg:  f,
		},
	}
	startTimer(&t.r)
	return t
}

// 
func goFunc(arg interface{}, seq uintptr) {
	go arg.(func())()
}
```

## 小菜2：time/tick.go

看了Timer的代码之后，Ticker就只是在创建runtimeTimer的时候多设置一个peroid字段。其他可以说是一模一样。内部的函数调用都是相同的，对于runtime而言，只有一中timer，在类型上不区分tiker。

```go
// A Ticker holds a channel that delivers `ticks' of a clock
// at intervals.
type Ticker struct {
	C <-chan Time // The channel on which the ticks are delivered.
	r runtimeTimer
}

// NewTicker returns a new Ticker containing a channel that will send the
// time with a period specified by the duration argument.
// It adjusts the intervals or drops ticks to make up for slow receivers.
// The duration d must be greater than zero; if not, NewTicker will panic.
// Stop the ticker to release associated resources.
func NewTicker(d Duration) *Ticker {
	if d <= 0 {
		panic(errors.New("non-positive interval for NewTicker"))
	}
	// Give the channel a 1-element time buffer.
	// If the client falls behind while reading, we drop ticks
	// on the floor until the client catches up.
	c := make(chan Time, 1)
	t := &Ticker{
		C: c,
		r: runtimeTimer{
			when:   when(d),
			period: int64(d),
			f:      sendTime,
			arg:    c,
		},
	}
	startTimer(&t.r)
	return t
}

// Stop turns off a ticker. After Stop, no more ticks will be sent.
// Stop does not close the channel, to prevent a concurrent goroutine
// reading from the channel from seeing an erroneous "tick".
func (t *Ticker) Stop() {
	stopTimer(&t.r)
}

// Reset stops a ticker and resets its period to the specified duration.
// The next tick will arrive after the new period elapses.
func (t *Ticker) Reset(d Duration) {
	if t.r.f == nil {
		panic("time: Reset called on uninitialized Ticker")
	}
	modTimer(&t.r, when(d), int64(d), t.r.f, t.r.arg, t.r.seq)
}

// Tick is a convenience wrapper for NewTicker providing access to the ticking
// channel only. While Tick is useful for clients that have no need to shut down
// the Ticker, be aware that without a way to shut it down the underlying
// Ticker cannot be recovered by the garbage collector; it "leaks".
// Unlike NewTicker, Tick will return nil if d <= 0.

// Tick是有可能会泄露内存。
func Tick(d Duration) <-chan Time {
	if d <= 0 {
		return nil
	}
	return NewTicker(d).C
}
```

## 硬菜：runtime/time.go

### 先看注释，注释总是有很多精彩。划重点：

- 小心地滑
- 申明限制`pp`, `status`, `nextwhen`这三个字段只能在这个文件的代码使用
- 负责创建timer的代码可以赋值`when`, `period`, `f`, `arg`, `seq`这几个字段，但是一旦timer被交给addtimer之后就不能再有任何修改了
- 一个已经激活的timer可以通过deltimer来反激活变为闲置状态。一个闲置状态的timer可以设置`period`, `f`, `arg`, `seq`这几个字段，注意没有`when`。闲置的timer可以被GC回收，但是不能通过addtimer再次激活。只有新创建的timer可以进入addtimer
- 一个激活的timer可以传递给modtimer，但是任何字段都不能修改，并且依然保持激活状态（那我给modtimer干嘛？）
- 一个闲置的timer可以通过resettimer变成激活状态，并更新`when`字段。新创建的timer也可以丢给resettimer。
- 归纳timer的操作：`addtimer`, `deltimer`, `modtimer`, `resettimer`, `cleantimers`, `adjusttimers`, `runtimer`
- 所有处于激活的timer都归于P上的一个堆，P的`timers`字段。其实闲置的timer在被回收之前，也是在这个堆上的，临时。
- timer的状态机

```go
// Code outside this file has to be careful in using a timer value.
//
// The pp, status, and nextwhen fields may only be used by code in this file.
//
// Code that creates a new timer value can set the when, period, f,
// arg, and seq fields.
// A new timer value may be passed to addtimer (called by time.startTimer).
// After doing that no fields may be touched.
//
// An active timer (one that has been passed to addtimer) may be
// passed to deltimer (time.stopTimer), after which it is no longer an
// active timer. It is an inactive timer.
// In an inactive timer the period, f, arg, and seq fields may be modified,
// but not the when field.
// It's OK to just drop an inactive timer and let the GC collect it.
// It's not OK to pass an inactive timer to addtimer.
// Only newly allocated timer values may be passed to addtimer.
//
// An active timer may be passed to modtimer. No fields may be touched.
// It remains an active timer.
//
// An inactive timer may be passed to resettimer to turn into an
// active timer with an updated when field.
// It's OK to pass a newly allocated timer value to resettimer.
//
// Timer operations are addtimer, deltimer, modtimer, resettimer,
// cleantimers, adjusttimers, and runtimer.
//
// We don't permit calling addtimer/deltimer/modtimer/resettimer simultaneously,
// but adjusttimers and runtimer can be called at the same time as any of those.
//
// Active timers live in heaps attached to P, in the timers field.
// Inactive timers live there too temporarily, until they are removed.
//
// addtimer:
//   timerNoStatus   -> timerWaiting
//   anything else   -> panic: invalid value
// deltimer:
//   timerWaiting         -> timerModifying -> timerDeleted
//   timerModifiedEarlier -> timerModifying -> timerDeleted
//   timerModifiedLater   -> timerModifying -> timerDeleted
//   timerNoStatus        -> do nothing
//   timerDeleted         -> do nothing
//   timerRemoving        -> do nothing
//   timerRemoved         -> do nothing
//   timerRunning         -> wait until status changes
//   timerMoving          -> wait until status changes
//   timerModifying       -> wait until status changes
// modtimer:
//   timerWaiting    -> timerModifying -> timerModifiedXX
//   timerModifiedXX -> timerModifying -> timerModifiedYY
//   timerNoStatus   -> timerModifying -> timerWaiting
//   timerRemoved    -> timerModifying -> timerWaiting
//   timerDeleted    -> timerModifying -> timerModifiedXX
//   timerRunning    -> wait until status changes
//   timerMoving     -> wait until status changes
//   timerRemoving   -> wait until status changes
//   timerModifying  -> wait until status changes
// cleantimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// adjusttimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// runtimer (looks in P's timer heap):
//   timerNoStatus   -> panic: uninitialized timer
//   timerWaiting    -> timerWaiting or
//   timerWaiting    -> timerRunning -> timerNoStatus or
//   timerWaiting    -> timerRunning -> timerWaiting
//   timerModifying  -> wait until status changes
//   timerModifiedXX -> timerMoving -> timerWaiting
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerRunning    -> panic: concurrent runtimer calls
//   timerRemoved    -> panic: inconsistent timer heap
//   timerRemoving   -> panic: inconsistent timer heap
//   timerMoving     -> panic: inconsistent timer heap
```

### time.Sleep的实现

`time.Sleep`的实现并不在time包里。而是在runtime下面，里面估计因为要操作timer的非外部方法。然后link过去的。

```go
// Package time APIs.
// Godoc uses the comments in package time, not these.

// time.now is implemented in assembly.

// timeSleep puts the current goroutine to sleep for at least ns nanoseconds.
//go:linkname timeSleep time.Sleep
func timeSleep(ns int64) {
	if ns <= 0 {
		return
	}
	// 获得当前的G	
	gp := getg()
	// G的timer字段，如果没有初始化就创建一个新的timer
	t := gp.timer
	if t == nil {
		t = new(timer)
		gp.timer = t
	}
	// 这里把goroutineReady作为timer的到期触发的回调，然后把当前的G作为参数放在t.arg，很显然是重新唤醒G，变成Grunnbale
	t.f = goroutineReady
	t.arg = gp
	// 将唤醒的时间存在nextwhen，在后面调用resettimer的时候，再根据nextwhen更新when，加入P的timers堆。
	t.nextwhen = nanotime() + ns
	// 并不直接调用resettimer，而是让gopark代劳，保证先park，再启动timer
	gopark(resetForSleep, unsafe.Pointer(t), waitReasonSleep, traceEvGoSleep, 1)
}

// resetForSleep is called after the goroutine is parked for timeSleep.
// We can't call resettimer in timeSleep itself because if this is a short
// sleep and there are many goroutines then the P can wind up running the
// timer function, goroutineReady, before the goroutine has been parked.
//
// gopark在G被park了之后会调用resetForSleep（因为我们把resetForSleep作为第一个参数传进去）。
// 我们无法在timeSleep里面直接调用resettimer，因为假如这是一个很短的sleep并且这时runtime里面有很多的goroutine，
// 那么P可能在G被park之前就直接执行timer的回调goroutineReady。(在park之前ready显而易见是不对的，但是这跟有很多的goroutine有什么关系呢？让品一品)
func resetForSleep(gp *g, ut unsafe.Pointer) bool {
	t := (*timer)(ut)
	resettimer(t, t.nextwhen)
	return true
}

// Ready the goroutine arg.
func goroutineReady(arg interface{}, seq uintptr) {
	goready(arg.(*g), 0)
}
```

### timer状态定义

```go
// Values for the timer status field.
const (
	// Timer has no status set yet.
	timerNoStatus = iota

	// Waiting for timer to fire.
	// The timer is in some P's heap.
	timerWaiting

	// Running the timer function.
	// A timer will only have this status briefly.
	timerRunning

	// The timer is deleted and should be removed.
	// It should not be run, but it is still in some P's heap.
	timerDeleted

	// The timer is being removed.
	// The timer will only have this status briefly.
	timerRemoving

	// The timer has been stopped.
	// It is not in any P's heap.
	timerRemoved

	// The timer is being modified.
	// The timer will only have this status briefly.
	timerModifying

	// The timer has been modified to an earlier time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedEarlier

	// The timer has been modified to the same or a later time.
	// The new when value is in the nextwhen field.
	// The timer is in some P's heap, possibly in the wrong place.
	timerModifiedLater

	// The timer has been modified and is being moved.
	// The timer will only have this status briefly.
	timerMoving
)
```

### timer操作函数

把能干的事情看一看，我们就可以大致知道runtime是如何操作管理timers的了。

#### addtimer(timer)

通过addtimer可以把一个新创建的timer交给当前的P来维护。P通过自己的timers字段来管理维护计时器堆，P要保持这个堆的排序。

```go
// addtimer adds a timer to the current P.
// This should only be called with a newly created timer.
// That avoids the risk of changing the when field of a timer in some P's heap,
// which could cause the heap to become unsorted.
func addtimer(t *timer) {
	// when must never be negative; otherwise runtimer will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = maxWhen
	}
	if t.status != timerNoStatus {
		throw("addtimer called with initialized timer")
	}
	t.status = timerWaiting

	when := t.when

	// pp指向当前绑定的P
	pp := getg().m.p.ptr()
	// 给pp的timersLock加锁
	lock(&pp.timersLock)
	// 清理timers
	cleantimers(pp)
	// 添加timer并维护堆
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	// timer实际是依赖networkpoller的
	wakeNetPoller(when)
}
```

#### cleantimers(P)

cleantimers主要是负责清理堆顶处于删除或被修改状态的timer，优化addtimer的速度。这个只在堆顶操作，如果堆顶不满足条件，就return了。所以只是一个优化步骤，不是必须的。

cleantimers的调用者必须持有当前PP的timersLock。

```go
// cleantimers cleans up the head of the timer queue. This speeds up
// programs that create and delete timers; leaving them in the heap
// slows down addtimer. Reports whether no timer problems were found.
// The caller must have locked the timers for pp.
func cleantimers(pp *p) {
	gp := getg()
	for {
		if len(pp.timers) == 0 {
			return
		}

		// This loop can theoretically run for a while, and because
		// it is holding timersLock it cannot be preempted.
		// If someone is trying to preempt us, just return.
		// We can clean the timers later.
		// 这挺蛋疼的，持有锁而不能被抢占，只好直接return。我先乘凉，留给后人种树。
		if gp.preemptStop {
			return
		}
		
		// 获得堆顶的timer
		t := pp.timers[0]
		// 一定要手牵手心连心，否则就是鬼子
		if t.pp.ptr() != pp {
			throw("cleantimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		// 清理掉状态为timerDeleted的timers
		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			// 持有deletedtimer计数器
			atomic.Xadd(&pp.deletedTimers, -1)
		// 排序被调整过的timers
		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			// Now we can change the when field.
			t.when = t.nextwhen
			// Move t to the right position.
			// 通过del再add，把timer调整到正确的排序
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			// 重新进入timerWaiting
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}
		default:
			// Head of timers does not need adjustment.
			return
		}
	}
}
```

#### doaddtimers(P, timer)

`pp.timers`是一个堆，虽然是以数组的形式保存的，每次addtimer都需要做一次堆整形（美美的）。

```go
// doaddtimer 把`t *timer`添加到`pp *p`的计时器堆上。调用者必须持有pp的timersLock。
func doaddtimer(pp *p, t *timer) {
	// Timers rely on the network poller, so make sure the poller
	// has started.
	// 这里就有趣了，注释告诉我们timer其实是依赖于network poller的，稍后我们展开看一下netpoll。
	// 确保netpoller被有效初始化。
	if netpollInited == 0 {
		netpollGenericInit()
	}

	// 花心是不可以的
	if t.pp != 0 {
		throw("doaddtimer: P already set in timer")
	}
	// pp和timer心连心
	t.pp.set(pp)
	i := len(pp.timers)
	// `pp.timers`是一个数组，所以先把t添加到末尾
	pp.timers = append(pp.timers, t)
	// 堆整形，保持排序
	siftupTimer(pp.timers, i)
	if t == pp.timers[0] {
		// 如果新加入的timer成为了堆顶，那就把timer的when更新到pp.timer0When
		atomic.Store64(&pp.timer0When, uint64(t.when))
	}
	// timer计数器
	atomic.Xadd(&pp.numTimers, 1)
}
```

#### wakeNetPoller(when)
```go
// wakeNetPoller wakes up the thread sleeping in the network poller,
// if there is one, and if it isn't going to wake up anyhow before
// the when argument.
func wakeNetPoller(when int64) {
	if atomic.Load64(&sched.lastpoll) == 0 {
		// In findrunnable we ensure that when polling the pollUntil
		// field is either zero or the time to which the current
		// poll is expected to run. This can have a spurious wakeup
		// but should never miss a wakeup.
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > when {
			netpollBreak()
		}
	}
}
```

#### deltimer(timer)

主要用来删除一个timer（废话）。但是有可能出现这个timer其实是属于另一个P的。所以deltimer里我们不会真的从计时器堆上把这个timer删掉，只是标记这个timer处于删除状态。之后，当拥有这个timer的P执行相应的操作的时候，会将其真正删除。

```go
// deltimer deletes the timer t. It may be on some other P, so we can't
// actually remove it from the timers heap. We can only mark it as deleted.
// It will be removed in due course by the P whose heap it is on.
// Reports whether the timer was removed before it was run.
func deltimer(t *timer) bool {
	for {
		switch s := atomic.Load(&t.status); s {
		case timerWaiting, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			// 阻止抢占调度
			mp := acquirem()
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before changing status,
				// as cleantimers in another goroutine
				// can clear t.pp of a timerDeleted timer.
				tpp := t.pp.ptr()
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				releasem(mp)
				// 为什么到这里就可以允许抢占了而不是下一行？
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else {
				releasem(mp)
			}
		case timerModifiedEarlier:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before setting status
				// to timerDeleted.
				tpp := t.pp.ptr()
				atomic.Xadd(&tpp.adjustTimers, -1)
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				releasem(mp)
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else {
				releasem(mp)
			}
		case timerDeleted, timerRemoving, timerRemoved:
			// Timer was already run.
			// 重复删除也是无事可做
			return false
		case timerRunning, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			// 这个看起来很高级，要删除的timer正在被另一个P执行，要挂起等他执行完
			osyield()
		case timerNoStatus:
			// Removing timer that was never added or
			// has already been run. Also see issue 21874.
			// 删除一个没有被add过的或者已经跑完的timer等于无事可做。
			return false
		case timerModifying:
			// Simultaneous calls to deltimer and modtimer.
			// Wait for the other call to complete.
			// 正在被修改，也要挂起等待
			osyield()
		default:
			// 报告奇怪的timer
			badTimer()
		}
	}
}
```

#### dodeltimer(P)

真正的删除相关的heavy work都在这里。删掉timer可能会破坏了堆的结构，要重整堆。

把第i个timer从当前的P的计时器堆上删掉。被调用时，我们必须已经锁在P上了。

```go
// dodeltimer removes timer i from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer(pp *p, i int) {
	if t := pp.timers[i]; t.pp.ptr() != pp {
		throw("dodeltimer: wrong P")
	} else {
		// 和当前的P脱勾
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if i != last {
		// 如果不是最后一个timer，那么就把最后一个timer填到当前删除的位置，堵上这个洞洞。
		pp.timers[i] = pp.timers[last]
	}
	// 丢掉最后一个timer，要么是要删除的timer，要么就已经被移动到前面的某一个位置了。
	pp.timers[last] = nil
	// 缩短timers
	pp.timers = pp.timers[:last]
	if i != last {
		// Moving to i may have moved the last timer to a new parent,
		// so sift up to preserve the heap guarantee.
		// 把最后一个timer移动到i之后，要重整堆
		siftupTimer(pp.timers, i)
		siftdownTimer(pp.timers, i)
	}
	if i == 0 {
		// 把最新的堆顶的when填到pp.timer0When
		updateTimer0When(pp)
	}
	atomic.Xadd(&pp.numTimers, -1)
}
```

#### dodeltimer0(P)

这个是直接删掉堆顶的timer。基本类似`dodeltimer(P)`。

```go
// dodeltimer0 removes timer 0 from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer0(pp *p) {
	if t := pp.timers[0]; t.pp.ptr() != pp {
		throw("dodeltimer0: wrong P")
	} else {
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if last > 0 {
		pp.timers[0] = pp.timers[last]
	}
	pp.timers[last] = nil
	pp.timers = pp.timers[:last]
	if last > 0 {
		siftdownTimer(pp.timers, 0)
	}
	updateTimer0When(pp)
	atomic.Xadd(&pp.numTimers, -1)
}
```

#### modtimer(timer, when, period, func, arg, seq) bool

modtimer只能被netpoll或者time.Ticker.Reset调用，用来修改一个现有的timer。

```go
// modtimer modifies an existing timer.
// This is called by the netpoll code or time.Ticker.Reset.
// Reports whether the timer was modified before it was run.
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) bool {
	if when < 0 {
		when = maxWhen
	}

	status := uint32(timerNoStatus)
	wasRemoved := false
	var pending bool
	var mp *m
loop:
	for {
		switch status = atomic.Load(&t.status); status {
		case timerWaiting, timerModifiedEarlier, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				pending = true // timer not yet run
				break loop
			}
			releasem(mp)
		case timerNoStatus, timerRemoved:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()

			// Timer was already run and t is no longer in a heap.
			// Act like addtimer.
			if atomic.Cas(&t.status, status, timerModifying) {
				wasRemoved = true
				pending = false // timer already run or stopped
				break loop
			}
			releasem(mp)
		case timerDeleted:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				atomic.Xadd(&t.pp.ptr().deletedTimers, -1)
				pending = false // timer already stopped
				break loop
			}
			releasem(mp)
		case timerRunning, timerRemoving, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerModifying:
			// Multiple simultaneous calls to modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}

	// 更新timer的属性
	t.period = period
	t.f = f
	t.arg = arg
	t.seq = seq

	// 如果wasRemoved，调用doaddtimer加入当前P的计时器堆
	if wasRemoved {
		t.when = when
		pp := getg().m.p.ptr()
		lock(&pp.timersLock)
		doaddtimer(pp, t)
		unlock(&pp.timersLock)
		if !atomic.Cas(&t.status, timerModifying, timerWaiting) {
			badTimer()
		}
		releasem(mp)
		wakeNetPoller(when)
	} else {
		// The timer is in some other P's heap, so we can't change
		// the when field. If we did, the other P's heap would
		// be out of order. So we put the new when value in the
		// nextwhen field, and let the other P set the when field
		// when it is prepared to resort the heap.
		// 这个timer还在某一个P的计时器堆上，我们不能修改它的when属性，否则会破坏了该P的计时器堆。这里我们把新的when放到timer的nextwhen上，让该P自己在重整堆排序的时候更新when
		t.nextwhen = when
		// 同时标记timer状态为timerModiefiedLater，延后执行
		newStatus := uint32(timerModifiedLater)
		if when < t.when {
			// 如果t.when > t.nextwhen
			// 标记状态为timerModifiedEarlier，提早执行
			newStatus = timerModifiedEarlier
		}

		// Update the adjustTimers field.  Subtract one if we
		// are removing a timerModifiedEarlier, add one if we
		// are adding a timerModifiedEarlier.
		adjust := int32(0)
		if status == timerModifiedEarlier {
			adjust--
		}
		if newStatus == timerModifiedEarlier {
			adjust++
		}
		if adjust != 0 {
			atomic.Xadd(&t.pp.ptr().adjustTimers, adjust)
		}

		// Set the new status of the timer.
		if !atomic.Cas(&t.status, timerModifying, newStatus) {
			badTimer()
		}
		releasem(mp)

		// If the new status is earlier, wake up the poller.
		if newStatus == timerModifiedEarlier {
			wakeNetPoller(when)
		}
	}

	return pending
}
```

#### resettimer(timer, when)

modtimer的简单包装，主要是可能已经跑过的timer重新激活。

```go
// resettimer resets the time when a timer should fire.
// If used for an inactive timer, the timer will become active.
// This should be called instead of addtimer if the timer value has been,
// or may have been, used previously.
// Reports whether the timer was modified before it was run.
func resettimer(t *timer, when int64) bool {
	return modtimer(t, when, t.period, t.f, t.arg, t.seq)
}
```

#### moveTimers(P, timers)

从另一个P上转移一部分timer切片到pp上。

目前当STW发生时，才会调用moveTimers，同时调用者应持有pp的timersLock。

STW我还不太了解。

```go
// moveTimers moves a slice of timers to pp. The slice has been taken
// from a different P.
// This is currently called when the world is stopped, but the caller
// is expected to have locked the timers for pp.
func moveTimers(pp *p, timers []*timer) {
	// 遍历偷来的timers
	for _, t := range timers {
	loop:
		for {
			switch s := atomic.Load(&t.status); s {
			case timerWaiting:
				// 如果timer的状态是timerWaiting，直接调用doaddtimer把t加到pp上
				t.pp = 0
				doaddtimer(pp, t)
				break loop
			case timerModifiedEarlier, timerModifiedLater:
				// 如果是timerModified，把状态改成timerMoving
				if !atomic.Cas(&t.status, s, timerMoving) {
					continue
				}
				// 从t.nextwhen更新t.when
				t.when = t.nextwhen
				t.pp = 0
				// 调用doaddtimer加入pp
				doaddtimer(pp, t)
				// 更新状态为timerWaiting
				if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
					badTimer()
				}
				break loop
			case timerDeleted:
				// 顺手把timerDeleted给移除
				if !atomic.Cas(&t.status, s, timerRemoved) {
					continue
				}
				t.pp = 0
				// We no longer need this timer in the heap.
				break loop
			case timerModifying:
				// 不是已经stw了么？还怎么yield？
				// Loop until the modification is complete.
				osyield()
			case timerNoStatus, timerRemoved:
				// We should not see these status values in a timers heap.
				badTimer()
			case timerRunning, timerRemoving, timerMoving:
				// Some other P thinks it owns this timer,
				// which should not happen.
				badTimer()
			default:
				badTimer()
			}
		}
	}
}
```

#### adjusttimers(P)

adjusttimers在当前P的计时器堆上找到被修改更早执行的timer调整它们的位置。这个过程中，adjusttimers还会移动被修改更晚执行的timers和移除已经删除的timers。

不是说好了clearDeletedTimers是唯一一个遍历真个计时器堆的函数吗？背叛了吗？

```go
// adjusttimers looks through the timers in the current P's heap for
// any timers that have been modified to run earlier, and puts them in
// the correct place in the heap. While looking for those timers,
// it also moves timers that have been modified to run later,
// and removes deleted timers. The caller must have locked the timers for pp.
func adjusttimers(pp *p) {
	if len(pp.timers) == 0 {
		return
	}
	// pp.adjustTimers限制adjusttimers可以调整几个timer。
	// 如果pp.adjustTimers是0，就简单验证一下堆然后就退出。
	if atomic.Load(&pp.adjustTimers) == 0 {
		if verifyTimers {
			verifyTimerHeap(pp)
		}
		return
	}
	var moved []*timer
loop:
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		if t.pp.ptr() != pp {
			throw("adjusttimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			// delete就没什么好再看的了，
			if atomic.Cas(&t.status, s, timerRemoving) {
				dodeltimer(pp, i)
				if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
					badTimer()
				}
				atomic.Xadd(&pp.deletedTimers, -1)
				// Look at this heap position again.
				i--
			}
		case timerModifiedEarlier, timerModifiedLater:
			// 标记timerMoving之后，就可以修改timer的when属性了
			if atomic.Cas(&t.status, s, timerMoving) {
				// Now we can change the when field.
				t.when = t.nextwhen
				// Take t off the heap, and hold onto it.
				// We don't add it back yet because the
				// heap manipulation could cause our
				// loop to skip some other timer.
				// 这里只会调用dedeltimer从堆上把这个要move的timer拿掉，存在moved里面。
				// 不重新放回堆是因为会影响当前的遍历。
				dodeltimer(pp, i)
				moved = append(moved, t)
				if s == timerModifiedEarlier {
					if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
						break loop
					}
				}
				// Look at this heap position again.
				i--
			}
		case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
			badTimer()
		case timerWaiting:
			// OK, nothing to do.
		case timerModifying:
			// Check again after modification is complete.
			osyield()
			i--
		default:
			badTimer()
		}
	}

	// 把moved都放回堆里
	if len(moved) > 0 {
		addAdjustedTimers(pp, moved)
	}

	if verifyTimers {
		verifyTimerHeap(pp)
	}
}

// addAdjustedTimers adds any timers we adjusted in adjusttimers
// back to the timer heap.
func addAdjustedTimers(pp *p, moved []*timer) {
	for _, t := range moved {
		// doaddtimer我们已经看过了，
		doaddtimer(pp, t)
		if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
			badTimer()
		}
	}
}
```

#### clearDeletedTimers(P)

- clearDeletedTimers从P的计时器堆上移除所有被标记删除的timer，以防止堆上垃圾成山。
- clearDeletedTimers是唯一的会遍历整个计时器堆的函数，moveTimers只在STW的时候才会被调用。
- 调用者依然应持有timersLock

```go
// clearDeletedTimers removes all deleted timers from the P's timer heap.
// This is used to avoid clogging up the heap if the program
// starts a lot of long-running timers and then stops them.
// For example, this can happen via context.WithTimeout.
//
// This is the only function that walks through the entire timer heap,
// other than moveTimers which only runs when the world is stopped.
//
// The caller must have locked the timers for pp.
func clearDeletedTimers(pp *p) {
	cdel := int32(0)
	cearlier := int32(0)
	to := 0
	changedHeap := false
	timers := pp.timers
nextTimer:
	for _, t := range timers {
		for {
			// 每处理一个timer都需要在这么一个循环，用来处理timer的状态timerModifiying。
			switch s := atomic.Load(&t.status); s {
			case timerWaiting:
				// 如果!changeHeap，保持原样，否则把t插入有效堆的末尾，进行一次siftup
				if changedHeap {
					timers[to] = t
					siftupTimer(timers, to)
				}
				// 移动到下一个有效堆的末尾
				to++
				// 处理下一个timer
				continue nextTimer
			case timerModifiedEarlier, timerModifiedLater:
				// 将timerModified改为timerMoving
				if atomic.Cas(&t.status, s, timerMoving) {
					// 从nextwhen更新timer的when
					t.when = t.nextwhen
					// 放到堆的末尾，进行siftup
					timers[to] = t
					siftupTimer(timers, to)
					// 移动堆尾位置
					to++
					// 标记changedHeap
					changedHeap = true
					// 重新进入timerWaiting
					if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
						badTimer()
					}
					// 增加earlier计数器
					if s == timerModifiedEarlier {
						cearlier++
					}
					continue nextTimer
				}
			case timerDeleted:
				if atomic.Cas(&t.status, s, timerRemoving) {
					// 解绑
					t.pp = 0
					// 增加删除计数器
					cdel++
					// 标记timerRemoved，这里游标to不会移动，所以下次出现激活的timer的时候会覆盖掉这个removed timer，剩下的就交给GC了
					if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
						badTimer()
					}
					// 标记changedHeap
					changedHeap = true
					continue nextTimer
				}
			case timerModifying:
				// Loop until modification complete.
				osyield()
			case timerNoStatus, timerRemoved:
				// We should not see these status values in a timer heap.
				badTimer()
			case timerRunning, timerRemoving, timerMoving:
				// Some other P thinks it owns this timer,
				// which should not happen.
				badTimer()
			default:
				badTimer()
			}
		}
	}

	// Set remaining slots in timers slice to nil,
	// so that the timer values can be garbage collected.
	// 因为有删除timer，所以实际长度比以前短，多余的长度都置为nil，方便GC回收。
	for i := to; i < len(timers); i++ {
		timers[i] = nil
	}

	// 对pp的timer相关的计数器操作
	atomic.Xadd(&pp.deletedTimers, -cdel)
	atomic.Xadd(&pp.numTimers, -cdel)
	atomic.Xadd(&pp.adjustTimers, -cearlier)

	// 更新timers长度
	timers = timers[:to]
	pp.timers = timers
	updateTimer0When(pp)

	// 验证4叉小堆的性质
	if verifyTimers {
		verifyTimerHeap(pp)
	}
}
```

#### runtimer & runOneTimer

`runtimer`和`runOneTimer`都是`//go:systemstack`的

runtimer检查timers里的第一个timer，如果该timer就绪了，runtimer就会执行这个timer然后删除或者更新timer。
- 返回0表示执行了一个timer
- 返回-1表示没有timer了
- 返回下一个timer执行的的时间。

根据堆顶的timer的情况，runtimer可能会访问到多个timer，但被执行timer只会有一个，遇到真爱的时候内部会调用`runOneTimer`，然后返回的。

```go
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//go:systemstack
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		// 反复确定彼此的真心
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now {
				// Not ready to run.
				// 没有就绪的timer，就返回堆顶timer的when
				return t.when
			}
			// 堆顶的timer就绪了
			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			// 调用runOneTimer
			runOneTimer(pp, t, now)
			// 返回0表示执行了堆顶的timer
			return 0

		case timerDeleted:
			// 删掉deleted的timer
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
			if len(pp.timers) == 0 {
				// 如果没有更多的timer，返回-1
				// 如果还有timer，继续循环
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater:
			// 如果有modifed，标记moving，更新when，然后重新加入堆
			// 标记timerWaiting，继续循环
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield()

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}

// runOneTimer runs a single timer.
// The caller must have locked the timers for pp.
// This will temporarily unlock the timers while running the timer function.
//go:systemstack
func runOneTimer(pp *p, t *timer, now int64) {
	if raceenabled {
		ppcur := getg().m.p.ptr()
		if ppcur.timerRaceCtx == 0 {
			ppcur.timerRaceCtx = racegostart(funcPC(runtimer) + sys.PCQuantum)
		}
		raceacquirectx(ppcur.timerRaceCtx, unsafe.Pointer(t))
	}

	f := t.f
	arg := t.arg
	seq := t.seq

	if t.period > 0 {
		// Leave in heap but adjust next time to fire.
		// 因为timer不是精准严格的，所以在更新when的时候，把delta补偿进来
		delta := t.when - now
		t.when += t.period * (1 + -delta/t.period)
		// 当前timer即堆顶，更新了when之后，siftdown到正确的位置
		siftdownTimer(pp.timers, 0)
		if !atomic.Cas(&t.status, timerRunning, timerWaiting) {
			badTimer()
		}
		// 更新pp.timer0When
		updateTimer0When(pp)
	} else {
		// Remove from heap.
		// 一次性
		dodeltimer0(pp)
		if !atomic.Cas(&t.status, timerRunning, timerNoStatus) {
			badTimer()
		}
	}

	if raceenabled {
		// Temporarily use the current P's racectx for g0.
		gp := getg()
		if gp.racectx != 0 {
			throw("runOneTimer: unexpected racectx")
		}
		gp.racectx = gp.m.p.ptr().timerRaceCtx
	}

	// 这个解锁加锁的顺序难得一见
	unlock(&pp.timersLock)

	// 调用timer的f
	// 这里有可能是默认的往timer的channel里丢一个当前时间戳
	// 也有可能是执行AfterFunc()里面的func
	f(arg, seq)

	lock(&pp.timersLock)

	if raceenabled {
		gp := getg()
		gp.racectx = 0
	}
}
```

#### badTimer()

badTimer是难以挽回的感情破裂。

```go
// badTimer is called if the timer data structures have been corrupted,
// presumably due to racy use by the program. We panic here rather than
// panicing due to invalid slice access while holding locks.
// See issue #25686.
func badTimer() {
	throw("timer data corruption")
}
```

#### 计时器堆的维护

这个计时器堆是一个4叉最小堆。大佬万岁！

在每次添加或者修改一个元素之后，都需要对修改的元素位置执行一次`siftup`和一次`siftdown`。

```go
// Heap maintenance algorithms.
// These algorithms check for slice index errors manually.
// Slice index error can happen if the program is using racy
// access to timers. We don't want to panic here, because
// it will cause the program to crash with a mysterious
// "panic holding locks" message. Instead, we panic while not
// holding a lock.


// verifyTimerHeap verifies that the timer heap is in a valid state.
// This is only for debugging, and is only called if verifyTimers is true.
// The caller must have locked the timers.
func verifyTimerHeap(pp *p) {
	for i, t := range pp.timers {
		if i == 0 {
			// First timer has no parent.
			continue
		}

		// The heap is 4-ary. See siftupTimer and siftdownTimer.
		p := (i - 1) / 4
		if t.when < pp.timers[p].when {
			print("bad timer heap at ", i, ": ", p, ": ", pp.timers[p].when, ", ", i, ": ", t.when, "\n")
			throw("bad timer heap")
		}
	}
	if numTimers := int(atomic.Load(&pp.numTimers)); len(pp.timers) != numTimers {
		println("timer heap len", len(pp.timers), "!= numTimers", numTimers)
		throw("bad timer heap len")
	}
}

// 向上整形
func siftupTimer(t []*timer, i int) {
	if i >= len(t) {
		badTimer()
	}
	when := t[i].when
	tmp := t[i]
	for i > 0 {
		p := (i - 1) / 4 // parent
		if when >= t[p].when {
			break
		}
		// 把父元素下移到当前位置，腾出空间，从父元素重复循环
		// 因为持有tmp备份，所以只需要把父元素下移即可，
		t[i] = t[p]
		i = p
	}
	if tmp != t[i] {
		// 把tmp备份元素放入恰当的位置
		t[i] = tmp
	}
}

// 向下整形
func siftdownTimer(t []*timer, i int) {
	n := len(t)
	if i >= n {
		badTimer()
	}
	when := t[i].when
	// 备份
	tmp := t[i]
	for {
		// c1, c2, c3, c4四叉
		c := i*4 + 1 // left child
		c3 := c + 2  // mid child
		if c >= n {
			break
		}
		w := t[c].when
		if c+1 < n && t[c+1].when < w {
			w = t[c+1].when
			c++
		}
		if c3 < n {
			w3 := t[c3].when
			if c3+1 < n && t[c3+1].when < w3 {
				w3 = t[c3+1].when
				c3++
			}
			if w3 < w {
				w = w3
				c = c3
			}
		}
		// c, w分别指向四个子元素中，排序值最小的
		// 到位了
		if w >= when {
			break
		}
		// 否则就把最小的子节点升到父节点的位置，向下腾出一个空间
		t[i] = t[c]
		i = c
	}
	// 把tmp备份放入
	if tmp != t[i] {
		t[i] = tmp
	}
}
```

#### timeSleepUntil() (int64, P)

- timeSleepUntil只能被`sysmon`和`checkdead`调用。`sysmon`是啥？
- timeSleepUntil返回两个值，1下一个timer触发的时间，2下一个timer所在的P

```go
// timeSleepUntil returns the time when the next timer should fire,
// and the P that holds the timer heap that that timer is on.
// This is only called by sysmon and checkdead.
func timeSleepUntil() (int64, *p) {
	next := int64(maxWhen)
	var pret *p

	// Prevent allp slice changes. This is like retake.
	// 锁所有的P
	lock(&allpLock)
	// 整个for range就是为了用pret和next找到下一个触发的timer对应的pp和时间
	for _, pp := range allp {
		if pp == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}
		// 读取pp的adjustTimers	
		c := atomic.Load(&pp.adjustTimers)
		if c == 0 {
			// 如果c==0，那么直接读取pp.timer0When
			w := int64(atomic.Load64(&pp.timer0When))
			// next和pret始终都追踪最小的pp.timer0When和pp
			if w != 0 && w < next {
				next = w
				pret = pp
			}
			continue
		}
		// 到这里说明pp.adjustTimers不为0
		// 在pp.timersLock上锁
		lock(&pp.timersLock)
		for _, t := range pp.timers {
			switch s := atomic.Load(&t.status); s {
			// 只关心timerWaiting和timerModified
			case timerWaiting:
				if t.when < next {
					next = t.when
				}
			case timerModifiedEarlier, timerModifiedLater:
				// 对于timerModified，我们关心的是timer.nextwhen
				if t.nextwhen < next {
					next = t.nextwhen
				}
				if s == timerModifiedEarlier {
					c--
				}
			}
			// The timers are sorted, so we only have to check
			// the first timer for each P, unless there are
			// some timerModifiedEarlier timers. The number
			// of timerModifiedEarlier timers is in the adjustTimers
			// field, used to initialize c, above.
			//
			// We don't worry about cases like timerModifying.
			// New timers can show up at any time,
			// so this function is necessarily imprecise.
			// Do a signed check here since we aren't
			// synchronizing the read of pp.adjustTimers
			// with the check of a timer status.
			
			// 如果所有的timers都排好了，那每个P上我们只关心堆顶的timer，除非P上有状态是timerModifiedEarlier的timers。
			// 我们也不关心timerModifying。因为任何时刻都会有新的timers出现，这个函数必然是不精确的。
			// 检查了所有的adjustTimers就可以退出了。
			if int32(c) <= 0 {
				break
			}
		}
		unlock(&pp.timersLock)
	}
	unlock(&allpLock)
	// 返回我们找到的下一个timer的时间和对应的P
	return next, pret
}
```

## timer的操作看的八九不离十了，但是都是谁来调用这些操作，是谁确保一定在到期的时候触发timer的func并且更新相应的计时器堆的？

# 下回分解
