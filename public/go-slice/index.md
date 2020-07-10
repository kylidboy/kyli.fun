# Golang Slice源码阅读笔记


<!--more-->
我自己也是醉了。Golang的“make三剑客”，Map,Chan,Slice，居然把最简单的slice放到最后来读了。哈哈。无所谓，反正也没人看。曾经我有一次被面试官问到，“slice在高并发情况下会有bug你知道吗？”，但他也没有告诉我是什么bug。看看能不能自己看出个未来。(我当时的回答是，如果真有，那go可以去死了。)

## slice定义

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

// A notInHeapSlice is a slice backed by go:notinheap memory.
type notInHeapSlice struct {
	array *notInHeap
	len   int
	cap   int
}
```

slice的定义显然是非常简单的，我记得这种形式被称为“胖指针”。我有一个pointer指向在内存某个地方的数据，行指针之能。但是我同时还有一些辅助字段来约束或者辅助我对指针指向的数据的操作。但是从外部看来，我就只是一个`*slice`，那些辅助的字段都不需要让外部知道。
- `len` 就是当前slice可以访问到的长度，超出这个len的都是越界。
- `cap` 就是背后支持这个slice的array的长度，这是真实有效的内存空间的长度。执行append的时候，如果超过了这个cap，将会触发扩容。
`len` <= `cap`

## 创建slice

创建slice是有`makeslice(et *_type, len, cap int) unsafe.Pointer`来实现的。很简单直观，先检查一下要求的大小会不会内存溢出，这里还很细心的区分了如果溢出是因为len还是因为cap。没有问题就直接把要求的内存空间直接分配了。这些`mallocgc`什么的，我以后再慢慢看。不影响。

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}

func makeslice64(et *_type, len64, cap64 int64) unsafe.Pointer {
	len := int(len64)
	if int64(len) != len64 {
		panicmakeslicelen()
	}

	cap := int(cap64)
	if int64(cap) != cap64 {
		panicmakeslicecap()
	}

	return makeslice(et, len, cap)
}
```

## slice扩容

slice的代码量和map是简直不能比的，主要的逻辑就是一个growslice。

```go
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

`growslice`主要就是为了在append过程中扩大slice的容量。它接收3个参数，元素类型，slice，和新的最小cap，并返回一个新的slice包含老的slice里所有的元素。经常会被人问的，slice扩容的大小也在这里。

- 首先会做一些边界检查，新的cap不能小于旧的cap之类的。
- 如果`et.size == 0`的话，这里有一个优化，直接返回一个slice，这个slice实际上指向的是zerobase。这个zerobase在src/runtime/malloc.go里面可以看到他的用处是为了给所有0-byte allocation提供一个地址。所以我们平时写代码的时候会有要求在不需要具体值的时候用`struct{}{}`，而不是`true/false`，这样更加节省内存。
- 先计算cap的两倍大小，放入doublecap，一般情况这个就是扩展后的slice的cap。除非你指明的新cap大于doublecap。
- 那么，如果cap小于doublecap的时候，并且当前slice的长度小于1024，这个doublecap就作为最终的newcap用来向runtime申请内存。但是如果slice的长度大于1024,我们平时所说的2倍扩容就不存在了。而是变成了，一个循环，来找到当前扩容的有效cap（newcap），每次以newcap的1.25倍扩容，直到newcap大于cap，最后如果newcap溢出成了负数，那么就直接把cap作为newcap。到这里，新的slice的new cap就确定下来了，并且保证`newcap >= cap`。
- 接下来cap和newcap只是我们定的元素的数量，要被转换成内存申请的真实内存的大小，会涉及到内存的roundupsize。因为内存分配在runtime里面不是按一个字节来分配的。具体这里不展开，只简单说一下，一部分空闲内存会被按照固定的大小，分成很多的等级，例如16byte，32byte，128byte。我们在申请内存的时候，如果我们实际申请的大小满足小于一个SmallSize常量，那么P会分配给一个能满足我们的最小等级（不要问P是什么，又是一个更长的话题）。如果大于SmallSize，就会对齐到pagesize。这里switch的主要是为了优化，不能用乘法就不用，能用位移操作就用位移操作。
- 保护我们申请的内存不会溢出。
- 分配内存，一些必要的初始化操作，返回slice。


## 工具函数

```go
func panicmakeslicelen() {
	panic(errorString("makeslice: len out of range"))
}

func panicmakeslicecap() {
	panic(errorString("makeslice: cap out of range"))
}

func isPowerOfTwo(x uintptr) bool {
	return x&(x-1) == 0
}

func slicecopy(to, fm slice, width uintptr) int {
	if fm.len == 0 || to.len == 0 {
		return 0
	}

	n := fm.len
	if to.len < n {
		n = to.len
	}

	if width == 0 {
		return n
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicecopy)
		racereadrangepc(fm.array, uintptr(n*int(width)), callerpc, pc)
		racewriterangepc(to.array, uintptr(n*int(width)), callerpc, pc)
	}
	if msanenabled {
		msanread(fm.array, uintptr(n*int(width)))
		msanwrite(to.array, uintptr(n*int(width)))
	}

	size := uintptr(n) * width
	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		*(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
	} else {
		memmove(to.array, fm.array, size)
	}
	return n
}

func slicestringcopy(to []byte, fm string) int {
	if len(fm) == 0 || len(to) == 0 {
		return 0
	}

	n := len(fm)
	if len(to) < n {
		n = len(to)
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicestringcopy)
		racewriterangepc(unsafe.Pointer(&to[0]), uintptr(n), callerpc, pc)
	}
	if msanenabled {
		msanwrite(unsafe.Pointer(&to[0]), uintptr(n))
	}

	memmove(unsafe.Pointer(&to[0]), stringStructOf(&fm).str, uintptr(n))
	return n
}
```
