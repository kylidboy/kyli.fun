---
title: Golang Map源码阅读笔记
date: 2020-05-26
lastmod: 2020-05-29
description: 最近，终于找了时间耐着性子读了go的map的实现，挺有意思的，没有想到map是这样实现的。这里做一些阅读的笔记。细节都是魔鬼。
featuredImage: images/go-map/gopher_map.png
featuredImagePreview: /images/go-map/gopher_map.png
tags:  
- golang
- runtime
- map
- 源码阅读 

categories:
- golang源码阅读 
---
<!--more-->
最近，终于找了时间耐着性子读了go的map的实现，挺有意思的，没有想到map是这样实现的。

这里做一些阅读的笔记。

大体上是跟着代码顺序记录(jump to defination)。

细节都是魔鬼。

## 开篇的注释，高度浓缩了整个map的设计理念

```go
// A map is just a hash tableG The data is arranged
// into an array of buckets. Each bucket contains up to
// 8 key/elem pairs. The low-order bits of the hash are
// used to select a bucket. Each bucket contains a few
// high-order bits of each hash to distinguish the entries
// within a single bucket.
//
// If more than 8 keys hash to a bucket, we chain on
// extra buckets.
//
// When the hashtable grows, we allocate a new array
// of buckets twice as big. Buckets are incrementally
// copied from the old bucket array to the new bucket array.
//
// Map iterators walk through the array of buckets and
// return the keys in walk order (bucket #, then overflow
// chain order, then bucket index).  To maintain iteration
// semantics, we never move keys within their bucket (if
// we did, keys might be returned 0 or 2 times).  When
// growing the table, iterators remain iterating through the
// old table and must check the new table if the bucket
// they are iterating through has been moved ("evacuated")
// to the new table.
```

*map*就是一个hash表，这个没有什么可疑惑的。这个hash表内部的数据元素并不是简单的key到value的映射，如果这么直白，那这个也没有什么好看的了。

hash表内部的数据其实是以一个叫*bucket*的数据结构作为组织存储的单位，所以整个hash表基本上就是一个包含若干*bucket*的数组。*bucket*的具体定义在后面跟着代码的结构走到的时候再细说。先来看看宏观的定义。每一个*bucket*都包含8个键值对，这个`8`是严格定义的，所以即使是一个空的*bucket*，也依然有占8个键值对的内存空间被分配给了*bucket*。

一个*key*被hash后等到一个哈希值，然后go会用这个哈希值的'low-order bits'去定位数据存储的*bucket*。当定位到目标*bucket*之后，（我们先假设当前是写访问），如果有空余的键值对空间（刚才说过的一定分配的8个键值对空间），那就直接在下一个空闲的键值对空间写入key和elem，并且在该bucket里面写入当前key的tophash作为key的索引，供访问的时候进行快速查找（就是一个索引，tophash后面在代码里再说）。如果当前定位到的*bucket*已经存满了8个键值之后，那么go会采用一个链结构去链接一个新的*bucket*来给这个hash对应的存储空间扩容，这个新的*bucket*被称为*overflow bucket*。

然后当*map*剩余空间吃紧或者含有太多的*overflow bucket*的时候，*map*会自动扩容，每次扩容都是之前的2倍大小。

```go
// Picking loadFactor: too large and we have lots of overflow
// buckets, too small and we waste a lot of space. I wrote
// a simple program to check some stats for different loads:
// (64-bit, 8 byte keys and elems)
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe
//        4.00         2.13        20.77         3.00         4.00
//        4.50         4.05        17.30         3.25         4.50
//        5.00         6.85        14.77         3.50         5.00
//        5.50        10.55        12.94         3.75         5.50
//        6.00        15.27        11.67         4.00         6.00
//        6.50        20.90        10.79         4.25         6.50
//        7.00        27.14        10.15         4.50         7.00
//        7.50        34.03         9.73         4.75         7.50
//        8.00        41.10         9.40         5.00         8.00
//
// %overflow   = percentage of buckets which have an overflow bucket
// bytes/entry = overhead bytes used per key/elem pair
// hitprobe    = # of entries to check when looking up a present key
// missprobe   = # of entries to check when looking up an absent key
//
// Keep in mind this data is for maximally loaded tables, i.e. just
// before the table grows. Typical tables will be somewhat less loaded.
```

这个表似乎是一些测试数据，只能说，需要的时候来参考一下。

## 定义的常量

```go
const (
    // 每个bucket包含的键值对数量：8
	//Maximum number of key/elem pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits

    // 计算map当前的负载，如果超过这个负载，就会出发扩容
	// Maximum average load of a bucket that triggers growth is 6.5.
	// Represent as loadFactorNum/loadFactorDen, to allow integer math.
	loadFactorNum = 13
	loadFactorDen = 2

	// Maximum key or elem size to keep inline (instead of mallocing per element).
	// Must fit in a uint8.
	// Fast versions cannot handle big elems - the cutoff size for
	// fast versions in cmd/compile/internal/gc/walk.go must be at most this elem.
	maxKeySize  = 128
	maxElemSize = 128

    // 这个是获取在一个bucket结构里键值对存储空间的内存偏移量，
    // 这个设计是因为在bucket的结构定义里没有直接定义字段，而是通过指针+偏移量来直接访问的。
	// data offset should be the size of the bmap struct, but needs to be
	// aligned correctly. For amd64p32 this means 64-bit alignment
	// even though pointers are 32 bit.
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)

    // 每个bucket内部，还有一个键值对的索引
    // 空置的键值对所对应的索引里面存的是标记状态，例如当前是最后的一个元素，或者是因为扩容已经把这个键值对迁移走了等等。
    // 如果键值对不为空的话，这里存的是索引的值， key的哈希值的高8位，可以快速找到bucket中的键值对
	// Possible tophash values. We reserve a few possibilities for special marks.
	// Each bucket (including its overflow buckets, if any) will have either all or none of its
	// entries in the evacuated* states (except during the evacuate() method, which only happens
	// during map writes and thus no one else can observe the map during that time).
	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
    emptyOne       = 1 // this cell is empty

    // 这两个值在扩容的时候会很重要，根据growth是原大小，还是2倍大小，分别使用x或者y
	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash     = 5 // minimum tophash for a normal filled cell.

	// flags
	iterator     = 1 // there may be an iterator using buckets
	oldIterator  = 2 // there may be an iterator using oldbuckets
	hashWriting  = 4 // a goroutine is writing to the map
	sameSizeGrow = 8 // the current map growth is to a new map of the same size

	// sentinel bucket ID for iterator checks
	noCheck = 1<<(8*sys.PtrSize) - 1
)
```

## map header定义

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```

这个就是*map*在runtime里面的定义了。

- count 是*map*内存储数据的计数器
- flags 是*map*运行状态的一些标志位，碰到了会说明的
- B 是一个很重要的参数。为什么叫B我不知道，但是这个B表示了当前*map*直接默认分配的*bucket*: 2的B次方个*bucket*。分配的是连续的内存片段，通过指针来进行检索，并且实际分配的*bucket*数量可能会大于这个数字，作为预分配的*overflow bucket*。
- noverflow *map*所有的*overflow bucket*的数量，这是一个概数。在B比较小的时候，它是一个精确数字，B超过一定的大小之后，它只是一个概数。
- hash0 用来做Hash运算的种子。
- buckets 这也是一个重要的字段。要注意它的类型并不是`* bucket`，而是一个普通的`unsafe.Pointer`。这是因为一个*map*所有的*bucket*都是通过这个指针访问。因为分配的*bucket*是一段连续的内存，所以，通过指针+偏移量，可以访问任何你想要的*bucket*。那这个offset我们等到代码出现的时候再说。

- oldbuckets 是在*map*扩容时持有扩容前的*bucket*。
- nevacuate 在扩容时，记录扩容迁移进程的，偏移量小于这个值的*bucket*都是已经被迁移过了。
- extra 包含了一些可选的字段，不是每一个*map*都会有这个字段。

## mapextra的定义

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```
这个简单翻译一下：

如果key和elem都不包含指针并且是内联的话，那么我们就标记bucket的类型是不含有指针的。为了保持overflow buckets存活，我们在这个`hmap.extra.overflow`和`hmap.extra.oldoverflow`里为每一个*overflow bucket*保存一个指针。只有在key和elem都不包含指针的情况下，才会使用`overflow`和`oldoverflow`。`overflow`存的都是指向`hmap.buckets`的`overflow bucket`，`oldoverflow`同理。然后这两个字段采用了指向切片的指针是为了能在`hiter`中也保存一个指针的副本。

最后这个`nextOverflow`是一个*bucket*的指针，指向下一个未被使用的*overflow bucket*。

## 有请bucket的定义

```go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

当我第一次看到这里的时候，还是很惊讶的，为什么感觉很麻烦的*bucket*，却只有这么简单的结构定义？这个`tophash`又是用来干嘛的？

注释给我们讲的已经很清楚了。

这个`tophash`是一个`uint8`的数组，数组大小就是8。刚好是每一个bucket所包含的键值对的个数。这不是一个巧合，它确实是为每一个键值对服务的。大体上来说，这个`tophash`里面存的值有两个作用：1.用key的hash值的最高byte作为索引值。2.参考前面定义的常量，tophash值的0~4都预留给了状态定义，在满足条件的时候这里就会存对应的状态值。

但其实`bmap`的结构，并不是这样简单结束了。在`tophash`后面还有两个隐藏的字段，没有字段名，一个是指向分配的键值对空间，另一个是当前*bucket*的*overflow bucket*的指针。

之所以这样实现，我猜是因为*map*可以存放各种类型key和elem，没有办法预先知道这个键值对空间的大小，（说白了就是没有范型我觉得，也有可能是节约空间的优化）。这也是为什么我们会在常量定义里面看到`dataOffset`那个偏移量的定义了。

所以刚才我们提到的所有的*bucket*，在代码实现层面都是一个`bmap`或者是一个`bmap`的指针。

还有一点要说明的，键值对的存储并不是key1/elem1/key2/elem2...这样的，而是key1/key2/...key8/elem1/elem2...elem8这样的，说因为这样可以避免一些不必要的内存对齐问题，比如`map[int64]int8`。

<a id="ref-img-bucket"></a>
![](/images/go-map/go_hash_map_bucket.png "这图从网上找的，凑合看一下，overflow没有")

## hmap给bmap分配overflow

```go
// incrnoverflow increments h.noverflow.
// noverflow counts the number of overflow buckets.
// This is used to trigger same-size map growth.
// See also tooManyOverflowBuckets.
// To keep hmap small, noverflow is a uint16.
// When there are few buckets, noverflow is an exact count.
// When there are many buckets, noverflow is an approximate count.
func (h *hmap) incrnoverflow() {
	// We trigger same-size map growth if there are
	// as many overflow buckets as buckets.
	// We need to be able to count to 1<<h.B.
	if h.B < 16 {
		h.noverflow++
		return
	}
	// Increment with probability 1/(1<<(h.B-15)).
	// When we reach 1<<15 - 1, we will have approximately
	// as many overflow buckets as buckets.
	mask := uint32(1)<<(h.B-15) - 1
	// Example: if h.B == 18, then mask == 7,
	// and fastrand & 7 == 0 with probability 1/8.
	if fastrand()&mask == 0 {
		h.noverflow++
	}
}

func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
	if h.extra != nil && h.extra.nextOverflow != nil {
		// We have preallocated overflow buckets available.
		// See makeBucketArray for more details.
		ovf = h.extra.nextOverflow
		if ovf.overflow(t) == nil {
			// We're not at the end of the preallocated overflow buckets. Bump the pointer.
			h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		} else {
			// This is the last preallocated overflow bucket.
			// Reset the overflow pointer on this bucket,
			// which was set to a non-nil sentinel value.
			ovf.setoverflow(t, nil)
			h.extra.nextOverflow = nil
		}
	} else {
		ovf = (*bmap)(newobject(t.bucket))
	}
	h.incrnoverflow()
	if t.bucket.ptrdata == 0 {
		h.createOverflow()
		*h.extra.overflow = append(*h.extra.overflow, ovf)
	}
	b.setoverflow(t, ovf)
	return ovf
}

func (h *hmap) createOverflow() {
	if h.extra == nil {
		h.extra = new(mapextra)
	}
	if h.extra.overflow == nil {
		h.extra.overflow = new([]*bmap)
	}
}
```

`newoverflow`用来创建分配一个新的*bucket*对象，在内部会调用这另外的两个函数。我们就来顺着`newoverflow`看一下代码，逻辑是很简单的。

1. 首先检查`hmap.extra`和`hmap.extra.nextOverflow`。
   - 如果都不为空的话，就说明我们之前在*map*扩容的时候预先分配了用做overflow的*bucket*了。这个时候我们只需要调用`hmap.extra.nextOverflow`的`overflow(t)`方法获取nextOverflow指向的*bucket*的*overflow bucket*。因为是预先分配的overflow，我们不知道这些overflow的占用情况，所以要先判断一下，是否依然有可用的*overflow bucket*。这里呢，就要根据预先设计好的规则来检查，我们在后面的代码会看到这个地方分配的overflow采用的是将自己的`overflow(t)`设为一个非空值（作为哨兵）来表示自己是这一片*overflow bucket*的队尾。这里如果看到自己已经是队尾了，就会把这个哨兵值给删掉，设为nil，并且同时将nextOverflow也置为nil，表示预分配的这些overflow已经全部用完，没有nextOverflow可以用了。而如果不是队尾，那么就直接简单的把下一个*overflow bucket*的地址放到`hmap.extra.nextOverflow`里面，就结束了。`ovf`此时就拿到了有效的*overflow bucket`。
   - 其他情况下，直接调用`newobject(t.bucket)`来分配新的内存对象赋给`ovf`。
2. 调用`hmap.incrnoverflow`增加overflow bucket的计数器，在`incrnoverflow`方法里面，会根据`hmap.B`的大小，来决定这个计数器是否准确计数。
3. 如果`t.bucket.ptrdata == 0`，这个就表示当前的map类型key和elem都不包含指针，这么这个时候就直接调用`hmap.createOverflow`创建`hmap.extra.overflow`（如果不存在），然后直接把当前获得的*overflow bucket*指针加入到`hmap.extra.overflow`切片中保存。
4. 把`ovf`指向的*bucket*设置为当前*bucket*的overflow，然后返回这个`ovf`。

## 华丽丽的创建map

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

这两个函数就是主要的负责创建map的家伙。

函数`makemap`对应的就是我们平时在写golang代码的时候，会经常写的`make(map[key]elem, size)`。这里compiler会自己做一些处理，不去深究，对我们看代码没有影响。简单理解，size就是hint。
- 先检查`hint`和`t.bucket.size`是否会造成内存分配的溢出，如果溢出或者是需要的内存大小超过了分配内存的上限，那么这个hint会被置为0。
- new一个`hmap`，调用`fastrand()`产生一个计算hash的随机种子。
- 根据loadfactor，找到一个满足`hint`的参数B，赋给`hmap.B`。
- 如果B大于0的话，就调用`makeBucketArray`分配相应的*bucket*数组空间。并且，如果返回的nextOverflow如果不为空，那么就在`hmap`里初始化好`hmap.extra`和`hmap.extra.nextOverflow`。

函数`makeBucketArray`才是真正的把创建map时的脏活累活全都给包了。在这个函数里，就会出现前面说的至少分配`1 << hmap.B`个*bucket*。最后一个参数，`dirtyalloc`如果传空值，那么bucket数组会新分配内存，但是如果传了一个之前分配的bucket数组指针，那么就会使用这个之前分配的内存空间来作为bucket数组，不会向系统新申请alloc。
- 调用函数`bucketShift(b)`得到需要分配的最少bucket数量`base`。
- 如果 `b > 4`，那么在`base`的基础上再增加`1 << (b-4)`个bucket，再经过roundup和对齐之后，得到真实创建的bucket数量`nbuckets`。
- 如果`dirtyalloc`为空，那么直接调用`newarray(t.bucket, int(nbuckets))`，新创建一个nbuckets × t.bucket大小的数组。
- 如果`dirtyalloc`不为空，那么就把指向的内存空间重置清空，作为当前的bucket空间。
- `base != nbuckets`就意味着之前计算实际分配bucket数量的时候，包含了一部分预分配的*overflow bucket*。这其中偏移量小于`base * t.bucketsize`的都是*bucket*，这之后的都是*overflow bucket*。然后，还记得我们之前说的“预分配overflow bucket的队尾哨兵”吗？这里最后一句话`last.setoverflow(t, (*bmap)(buckets))`就是设置哨兵。这个哨兵的需求很简单，只要不是nil，并且将来判断队尾的时候也只检查是否为nil，所以就直接使用分配的bucket数组的地址作为这个哨兵地址。
- 返回分配的内存地址

这样，一个新的map就在golang的runtime里创建好了。接下来，我们再来看看map的访问。

## 若干个map访问的函数

```go
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the elem type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}

func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess2)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0]), false
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}

// returns both key and elem. Used by map iterator
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {
	if h == nil || h.count == 0 {
		return nil, nil
	}
	hash := t.hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.key.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return k, e
			}
		}
	}
	return nil, nil
}

func mapaccess1_fat(t *maptype, h *hmap, key, zero unsafe.Pointer) unsafe.Pointer {
	e := mapaccess1(t, h, key)
	if e == unsafe.Pointer(&zeroVal[0]) {
		return zero
	}
	return e
}

func mapaccess2_fat(t *maptype, h *hmap, key, zero unsafe.Pointer) (unsafe.Pointer, bool) {
	e := mapaccess1(t, h, key)
	if e == unsafe.Pointer(&zeroVal[0]) {
		return zero, false
	}
	return e, true
}

// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	hash := t.hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/elem at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

这几个"mapaccess"都长得特别像，主要是返回值略有不同，"_fat"结尾的似乎就是给了一个默认零值，没啥好多看的。我们先把`mapaccess1`拎出来。

race相关的check我们都跳过。

- mapaccess1的特点是永远不会返回nil，当key找不到的时候，会默认返回一个指针，指向一个elem类型的默认零值。
- `t.hasher`应该是默认用来计算key的hash值的，传入key和hmap.hash0种子，得到hash
- `bucketMask(h.B)`获得一个掩码。我们之前已经知道，`h.B`规定了一定有`1 << h.B`个*bucket*，那么这个mask就主要为了用来做偏移量运算时，省去溢出检查。通过`offset & mask`，保证offset小于`1 << h.B`。假设：h.B = 7，那么一共有b10000000个bucket，这个时候mask即b01111111，也就是最大的bucket数组下标，&操作及保证了不会大于b01111111。
- 用hash&mask来作为目标*bucket*的偏移量，并找到目标*bucket*
- 检查map扩容。这里比较tricky的是，如果当前正在扩容，要判断一下扩容的类型，如果是要大小扩大一倍的扩容，那么需要先把mask右移一位，也就是把偏移量的空间缩小一倍，回到扩容前。然后再做一次偏移量定位到bucket。这个时候获得的bucket，存在一个不确定性，就是它是否已经被迁移走了，这个需要调用`evacuated`来检查，其实就是看具体tophash[0]的值。如果没有被迁移，那么就把这个bucket作为定位到的bucket，否则还是使用原来的已经定位的bucket，这段代码也就没有什么用了。
- 调用`tophash`函数计算key的tophash。
- 接下来就是用一个嵌套循环去遍历*bucket*和相应的*overflow bucket*，每个*bucket*都分配了8个键值对空间，内循环就直接检查这8个键值对，先访问`bmap`的tophash数组，去对比key的tophash是否有相同，如果遇到相同的，那么就意味着有可能对应的键值对就是我们要找的键值对，这仅仅是有可能，这里的tophash的作用很类似一个前缀索引。假设遇到了一个相同的tophash，利用之前定义的`dataOffset`快速找到key的存储空间，再进一步判断key的类型是否是一个指针并进行一次去引用操作。然后就是key和key空间的对比，如果相同，那这个就是我们要找的键值对，接下来找到相应的elem就可以了，偏移量的操作可以参考[bucket示意图](#ref-img-bucket)。
- 在检查tophash的过程中，如果碰到emptyRest，这个就是说后面的键值对都是空的，也不存在overflow的问题，那么就直接break到bucketloop，结束外循环，认为key不存在。

几个mapaccess的逻辑都是差不多的，但是需要特别注意的是`mapAssign`。

`mapAssign`也是一个access函数，但是区别是他在找不到key的时候会在map中插入这个key，并且返回指向相应elem位置的指针。其中有一个特殊操作，如果满足相应的条件，就要进行一次`hashGrow()`的调用，map会扩容，所以代码又跳转到最开始重新寻找key，稍显复杂。其余，找key插入位置和因为8个键值空间都占用后创建overflow的代码都和之前的类似，就不再赘述了。

## mapdelete和mapclear

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	hash := t.hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}

// mapclear deletes all keys from a map.
func mapclear(t *maptype, h *hmap) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapclear)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
	}

	if h == nil || h.count == 0 {
		return
	}

	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	h.flags ^= hashWriting

	h.flags &^= sameSizeGrow
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0
	h.count = 0

	// Keep the mapextra allocation but clear any extra information.
	if h.extra != nil {
		*h.extra = mapextra{}
	}

	// makeBucketArray clears the memory pointed to by h.buckets
	// and recovers any overflow buckets by generating them
	// as if h.buckets was newly alloced.
	_, nextOverflow := makeBucketArray(t, h.B, h.buckets)
	if nextOverflow != nil {
		// If overflow buckets are created then h.extra
		// will have been allocated during initial bucket creation.
		h.extra.nextOverflow = nextOverflow
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

`mapdelete`函数顾名思义，从map里删除一个key。如果理解了mapaccess的代码，就会发现这个函数大部分的逻辑都如出一辙，无非就是
1. mapaccess找到key在map中的位置。
2. 清空key和elem，如果是指针，就设为nil，如果不是指针，就清空相应大小的内存，然后把tophash对应的元素设为emptyOne。
3. 这里有一个delete自己的优化，占用了一个独立的循环。在执行完清空操作之后，进行如下的检测：
	- 如果被删除的key占用的是这个bucket的最后一个键值对空间，那么就看有没有overflow，有的话，overflow是否为空，如果overflow不为空，那就直接跳到结束删除的收尾工作。
	- 如果被删除的key不是这个bucket的最后一个键值对空间，那么就直接检查下一个tophash是不是emptyRest，如果不是，那么就直接跳到结束删除的收尾工作。
	- 如果上面两种情况没有满足，那么进入一个回溯的循环。先把这个当前位置的tophash设置为emptyRest，表示后面都是空。然后再看一下，key的位置i是不是0。如果是，在当前bucket所属的overflow bucket链中，找到上一个bucket，并把i设为7,指到最后一个键值对空间。如果不是，那么直接i向前走1,指向前一个键值对空间。然后检查这个键值对是不是emptyOne，如果不是，就直接break，进入删除收尾的工作。如果是，继续这个循环，在下一次循环开始的时候，这个emptyOne会被设为emptyRest。这样就一步一步把当前被删除key的位置之前所有连续的emptyOne都改成了emptyRest。这样做的目的是当下一次mapaccess访问到这里的时候可以及早退出，不需要连续的访问empty的键值对。

`mapclear`函数就是简单的全部清除重置，背后的bucket数组也复用了`makeBucketArray`来进行清除。

## map的扩容

```go
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}

func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/elem index into b
	k unsafe.Pointer // pointer to current key storage
	e unsafe.Pointer // pointer to current elem storage
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```

废话不多说，先上图。<a id="ref-img-grow"></a>
![](/images/go-map/go_hash_map_evacuate.png "迁移")

首先，go的map扩容不是STW一次完成，而是采用的增量操作。`growWork()`中只要当前的扩容没有完成，就会调用2次`evacuate`。一次是我们当前明确的要操作的bucket，通过参数bucket传入的。而另一次，则是根据当前的迁移进度`hmap.nevacuate`进行一次evacuate，以推进整个扩容的进展。

我们来看一下`hashGrow`都做了什么，这个函数相对比较简单。它只是把map的容量扩大，并做好所有的house keeping。
- 默认扩容增加一倍的空间，但如果当前情况没有overLoadFactor，那么就不增加空间，并在`hmap.flags`标记当前是`sameSizeGrow`。
- 把当前的buckets转给oldbuckets，然后调用`makeBucketArray`来分配新的bucket数组，`h.B + bigger`有可能是扩大一倍有可能是保持原来的大小。在这一步不需要关心具体的情况。
- 拷贝一个flags标志，同时清除iterator和oldIterator标志位。如果当前有iterator标志，那么就标记oldIterator。所以iterator标志位在这之后始终是清除的。
- 更新h.B，h.flags标志，h.oldbuckets, h.buckets，新的map迁移进度和overflow计数器都是0。
- 同样把overflow buckets过渡到oldoverflow
`hashGrow()`的工作就完成了。等待下次触发`growWork()`或者`evacuate()`，进行实际的增量迁移。

接下来，重头戏来了，函数`evacuate(t *maptype, h *hmap, oldbucket uintptr)`干的事情就是上面的[示意图](#ref-img-grow)中表达的迁移过程。被迁移的*bucket*用参数oldbucket表示，它其实是一个指针的偏移量。
- 首先在`h.oldbuckets`里面根据oldbucket找到目标*bucket* b。
- 然后得到oldbuckets的数量newbit。
- 如果b已经被迁移了，那么就比较一下oldbucket值和当前记录的迁移进度，如果相等就调用`advanceEvacuationMark`来追踪最新的迁移进度。evacuate就结束了。
- 如果b没有被迁移，那么下面就要进行对oldbucket的迁移工作了。
- 创建一个`[2]evacDst`的数组xy，这个xy同时表示了两个迁移的目的地，因为是扩容一倍，把新的地址分成高低两部分，那么后面根据情况的不一样，最终的迁移目标可以在高位，也可以在低位。当前默认是使用低位的目的地，所以先在`h.buckets`里根据oldbucket偏移量直接定位到新的bucket地址，并把它保存在xy[0]里。
- 判断当前是哪种类型的扩容，如果是double空间的扩容，这个时候我们需要计算一次在新的2倍大小的buckets数组里面，我们要迁移的目的地就变成了高位的目的地，保存存在xy[1]中。
- 然后进入喜闻乐见的嵌套循环，外层循环是爬overflow bucket链的，内循环是遍历bucket内的8个键值对。
- 沿袭mapaccess的套路，先检测tophash，如果tophash isEmpty()，那么检测下一个键值对。
- <a id="evacuate-special"></a>然后到了如果当前是double的扩容，那么我们就要决定究竟是用x还是y目的地。刚才只是进行了目的地的寻找和记录。现在是决定命运的时刻。先计算key的hash。然后这里有点骚操作，我们挑主要的逻辑进行跟踪。
- 第一种情况，根据注释的说明，`t.key.equal(k2, k2)`会出现false的情况， 例如NaN，这就导致了所有的操作都会变得不确定了，因为逻辑上相同的key现在不等了。而因为有iterator的存在，这个确定性和可重复性是一个刚需(否则会出现无法遍历到某一个bucket的情况，这是我自己的猜测)。在这种情况下，注释又说可以随意的决定迁移目的地，认为当前对应的tophash没有任何意义，所以就用了tophash的最低位来决定高低位，一个50%的随机性。同时再用当前key的hash重新计算一次tophash。覆盖掉之前的top。这一步的意义还要待观察。
- 其他情况下，就直接根据`hash&newbit != 0`来决定`useY`的值。`useY == 1`就是高目的地。同一个key，在扩容前和扩容后算出来的hash应该是一致的。而在oldbuckets里面，当前key算出来的hash被`1 << noldbuckets - 1`这个mask掩码处理了溢出。但是新的buckets的size扩大了1倍，所以hash的用来参与选择bucket的"lower bits"又多了一位，那么就应该用新增的高位直接决定扩容后的目的地，是高还是低，这样在后续的mapaccess中，也会保持这个寻址的可重复性。
- 接下来一个很骚气的常量检查，自己定义的常量自己检查，为了保证`evacuatedY - evacuatedX == 1`。这样，`evacuatedX + useY`就可以标记当前键值对最新的状态，同时说明了是迁移到了高位还是低位。
- 再下来就是把当前的键值对拷贝到目的地的相应位置，并且标注tophash对应的值。然后这个evacDst数组是一直复用的，所以在内循环的最后也要相应的更新指针。
- 在每次外循环的最后，把当前bucket的tophash数组保留，因为只需要继续保存迁移的结果状态就够了，数据已经在新的bucket中了。清空键值对空间的指针，让GC更好的回收垃圾。
- 外循环结束后，本次evacuate的工作就全部完成了，同样也调用一次`advanceEvacuationMark()`来更新迁移的进度。并且如果发现oldbuckets已经全部被迁移了，就标记当前的扩容完成，清除oldbuckets和oldoverflow指针。

## 普通工具函数

```go
// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
	// Masking the shift amount allows overflow checks to be elided.
	return uintptr(1) << (b & (sys.PtrSize*8 - 1))
}

// bucketMask returns 1<<b - 1, optimized for code generation.
func bucketMask(b uint8) uintptr {
	return bucketShift(b) - 1
}

// tophash calculates the tophash value for hash.
func tophash(hash uintptr) uint8 {
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	if top < minTopHash {
		top += minTopHash
	}
	return top
}

func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}

func (b *bmap) overflow(t *maptype) *bmap {
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}

func (b *bmap) setoverflow(t *maptype, ovf *bmap) {
	*(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize)) = ovf
}

func (b *bmap) keys() unsafe.Pointer {
	return add(unsafe.Pointer(b), dataOffset)
}

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}

// growing reports whether h is growing. The growth may be to the same size or bigger.
func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}

// sameSizeGrow reports whether the current growth is to a map of the same size.
func (h *hmap) sameSizeGrow() bool {
	return h.flags&sameSizeGrow != 0
}

// noldbuckets calculates the number of buckets prior to the current map growth.
func (h *hmap) noldbuckets() uintptr {
	oldB := h.B
	if !h.sameSizeGrow() {
		oldB--
	}
	return bucketShift(oldB)
}

// oldbucketmask provides a mask that can be applied to calculate n % noldbuckets().
func (h *hmap) oldbucketmask() uintptr {
	return h.noldbuckets() - 1
}

func bucketEvacuated(t *maptype, h *hmap, bucket uintptr) bool {
	b := (*bmap)(add(h.oldbuckets, bucket*uintptr(t.bucketsize)))
	return evacuated(b)
}
```

- 函数`bucketShift`接收一个参数是`hmap.B`，计算得到分配*bucket*的数量，里面的`b & (sys.PtrSize*8 - 1)`，只是为了省略掉溢出的检测，在64位的平台上，最多把`uintptr(1)`左移63位，其他位宽同理，这个位宽就是通过`sys.PtrSize`获得的。
- 函数`bucketMask`在`bucketShift`基础上，通过-1操作，获得一个`111111111...`的掩码。这个掩码后面在定位*bucket*的时候被用来保证偏移量不会溢出。
- 函数`tophash`接收一个参数hash，应该是一个平台位宽长度的哈希值，例如amd64平台上就是一个64bit的哈希值。取这个哈希值的最高8bit作为key的tophash（索引），注意这里要预留`minTopHash`给迁移状态位。
- 函数`evacuated` 通过读取`tophash[0]`的值，判断一个*bucket*是否已经在扩容过程中被迁移了。
- 方法`overflow` 根据`*maptype`的`bucketsize`来获得当前*bucket*所包含的指向*overflow bucket*的指针。因为操作内容就是指针本身，所以通过偏移量计算之后得到的是指针的指针。
- 方法`setoverflow` 同上，这个是相对应的setter方法。
- 方法`keys` 通过在b的地址上+`dataOffset`来获取存放key的起始地址。`dataOffset`是在常量定义里预先计算好的固定偏移量，其实可以认为就是`tophash`占用的内存大小。

## 把hiter放在最后

```go
// A hash iteration structure.
// If you modify hiter, also change cmd/compile/internal/gc/reflect.go to indicate
// the layout of this structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
	elem        unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}

// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
	}

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || t.key.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := t.hasher(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || t.key.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

我觉得代码读到这个时候，已经不需要再去一字一字的翻译每个字段的意义，应该看一眼就能知道个大概了吧。

`mapiterinit(t *maptype, h *hmap, it *hiter)`函数，用来初始化遍历map的iter struct。这个被初始化的iter必须在外部零值初始化好。然后把这个指针传进mapiterinit。
- 内部check，hiter占用12个sys.PtrSize。
- 先给`hiter`的[`t`, `h`, `B`, `buckets`]赋值。
- 判断maptype是否不包含指针，如果是，那么就对`hmap`进行`createOverflow`，并且把`overflow`和`oldoverflow`也赋值给`hiter`。这两个都是指向`[]*bmap`的指针。这样即使在遍历的过程中发生了扩容或者是新增了overflow bucket，it也可以让所有相关buckes保持活性不被GC。
- `fastrand()`随机一个偏移量。如果`h.B` > 28，在随机一次并且放大一些，这个随机产生的只是决定遍历起点。r经过bucketMask掩码操作之后，赋值给`it.startBucket`。
- `it.offset`依然利用随机值r来决定键值对的起点偏移量。
- `it.bucket`是iterator state，从startBucket开始。
- 在`h.flags`里面标记当前有一个iterator在这个map上。但是不能有多个。所以这里要先检测，然后再用原子操作。
- 把it传给mapiternext

`mapiternext(it *hiter)`是真正遍历map的函数。每次调用返回当前指向的key和elem并向前移动遍历的游标。主要麻烦的是要处理遍历过程中发生的扩容。`bptr`和`b`就是同一个东西，指向当前访问的*bucket*，但是这要兼容两种情况：1.正常的*bucket* 2.*bucket*的*overflow bucket*。所以，真正在遍历buckets空间的是游标偏移量`iter.bucket`。
- 先从it里面恢复当前的遍历的进程，指向的*bucket*以及键值对的偏移量。
- 整个遍历逻辑是从label next一直到函数结束。next的构成主要是一个if判断和一个for循环。
- 这个if判断主要用来做边界检测。当我们的遍历第一次开始的时候(go代码里面第一次调用next)，`it.bptr`这个bucket游标，应该是nil，所以这时会进入这个if。
    - 首先判断当前的遍历是否已经全部完成，如果wrapped并且bucket == startBucket，那么就把iterator的key和elem设为nil，并直接return，结束遍历。
	- 接下来就是合理合法的给b这个`*bmap`指向下一个被遍历的*bucket*。先看如果map不在扩容，那直接根据当前bucket偏移量找到将被遍历的*bucket*，指给b，同时checkBucket不需要，就设为noCheck。但是如果map正在扩容，因为这个bucket偏移量有可能落在扩容后的新增空间也有可能落在old空间，我们优先考虑oldbuckets就要先把bucket偏移量映射到oldbucket偏移量，然后在`h.oldbuckets`里面去定位到*old bucket*并指给b。判断b是否已经被迁移，如果未迁移，那么checkBucket就会记录当前的bucket偏移量，表示扩容后的目的*bucket*。如果已经迁移了，那么就直接用bucket偏移量直接找到`h.buckets`里的*bucket*，并且这个*bucket*的键值对不会有位置变化，所以checkBucket标记noCheck。
	- bucket++，游标偏移量前进。顺便看一眼是不是已经到了`h.buckets`的队尾，如果是，那么游标回到队首，并标记wrapped。然后每次bucket移动之后都需要完整遍历整个键值对空间，所以计数器i被覆写为0。
- 来看for循环。这是一个单层循环。在给定的*bucket*前提下，遍历这个*bucket*的所有键值对。这时我们的游标指针b已经肯定有指向一个存在的*bucket*，不会出现空指针。
	- 这个计数器i也是挺有讲究的。我把它叫做“体外循环计数器”。循环条件里的"i++"和循环体中的return前的“i+1”，共同控制着i的步进。当没有找到有效的键值对时，通过i++快速遍历键值对空间。但是当找到有效的键值对，mapiternext函数返回，当我们下次再次调用mapiternext的时候，所有的偏移量都会被恢复出来，要保证我们访问的就是i的下一个键值对，所以这个i+1会被保存到iter.i里，然后离开for循环，在循环体外面保存这循环的计数器。我们说过it.offset是随机的起始位置，所以用`(i + it.offset) & (bucketCnt - 1)`来做溢出处理。这样可以在任意的起始位置，经历bucketCnt次循环之后，遍历完bucketCnt个元素。
	- 如果tophash表示元素为空或着已经被迁移走了，直接continue。
	- k和e分别指向当前的key和elem。
	- 如果当前checkBucket != noCheck并且map在double空间扩容，会出现一些特殊情况需要考虑。这里的逻辑需要和[evacuate](#evacuate-special)里面的选择迁移目的地的代码逻辑保持一致。因为迁移键值对的目的地，可能会有两个，所以这里我们要根据相应的情况去判断。第一种情况，常规的key，得到key的hash，然后如果这个hash在扩容后的bucket偏移量不等于checkBucket所记的bucket偏移量，那么我们就认为当前的key不需要在这次遍历的时候访问到。这个“不应该访问却访问到”发生的原因是，如果有扩容发生，我们的b访问的是`b.oldbuckets`，但这个*bucket*中的键值对在扩容过程中，会有可能迁移到两个不同的*bucket*里面，所以通过`h.oldbuckets`可能访问到在当前的*bucket*中不存在的key。(感觉上像是因为扩容造成的幻读一样)。当然，这个key不是真的不存在，而是iteration期待在别的*bucket*访问过程中会遍历到这个key。这里continue是为了防止同一个key因为扩容而被遍历两次的情况。另一种NaN的情况也是同理。
	- 这一步我们就认为已经正确遍历到了键值对，把k和e分别赋给`it.key`和`it.elem`。通过调用一次`mapaccessK`来应对遍历过程中key被删掉的情况。
	- 在it里面记录bucket游标偏移量，bptr游标指针，体外循环计数器，checkBucket标记，然后返回当前访问的key和elem。
- 这里说明当前的*bucket*的键值对已经被全部遍历了。是时候更新检查当前*bucket*对应的*overflow buckets*链，同时重置体外计数器i。如果overflow不是nil，那么goto next就会直接进入for循环开始新的键值对遍历。如果overflow是nil，那么next的时候，先会进入if，推进bucket游标偏移量，开始下一个*bucket*的征程。