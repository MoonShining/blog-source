---
title: goim中的数据结构
date: 2018-03-11 13:35:36
tags:
    - golang
    - goim源码分析
---

goim中数据结构的设计非常出彩，值得仔细品味。

### Timer
在长连接这样的场景下，有N条连接需要维护心跳信息，凡人的做法可能就是开启N个gorutine，但goim使用最小堆高效处理了这个问题。

Timer就是定时器的结构，对外提供Add、Del、Set三个方法用于添加，删除、修改TimerData。

TimerData存储单个定时器的信息，到期则执行回调函数fn。

```golang
// libs/time/timer.go

type Timer struct {
	lock   sync.Mutex
	free   *TimerData
	timers []*TimerData
	signal *itime.Timer
	num    int
}

type TimerData struct {
	Key    string
	expire itime.Time
	fn     func()
	index  int
	next   *TimerData
}
```

先看一下添加删除timer的逻辑

```golang
// libs/time/timer.go

func (t *Timer) Add(expire itime.Duration, fn func()) (td *TimerData) {
	t.lock.Lock()
	td = t.get()
	td.expire = itime.Now().Add(expire)
	td.fn = fn
	t.add(td)
	t.lock.Unlock()
	return
}


func (t *Timer) Del(td *TimerData) {
	t.lock.Lock()
	t.del(td)
	t.put(td)
	t.lock.Unlock()
	return
}

func (t *Timer) Set(td *TimerData, expire itime.Duration) {
	t.lock.Lock()
	t.del(td)
	td.expire = itime.Now().Add(expire)
	t.add(td)
	t.lock.Unlock()
	return
}
```

除去加锁部分，内部就是调用了add、、del、get、put这几个方法。

get和put非常简单，只是根据当前的free指针获得或者放回去一个TimerData。

add和del是典型的堆操作，就是往timers这个堆里添加删除元素。

Timer在初始化时就会构造好一条free链表，在Add时，先取出free指向的节点，加入到timers堆中。在Del时，先从堆中删除，再放回链表中。

这条free链表是为了避免频繁申请内存做的优化！get和put负责在链表中申请和释放节点，add和del在获取到节点(TimerData)后进行堆的调整！

```golang
// libs/time/timer.go

// get get a free timer data.
func (t *Timer) get() (td *TimerData) {
	if td = t.free; td == nil {
		t.grow()
		td = t.free
	}
	t.free = td.next
	return
}

// put put back a timer data.
func (t *Timer) put(td *TimerData) {
	td.fn = nil
	td.next = t.free
	t.free = td
}

// Push pushes the element x onto the heap. The complexity is
// O(log(n)) where n = h.Len().
func (t *Timer) add(td *TimerData) {
	var d itime.Duration
	td.index = len(t.timers)
	// add to the minheap last node
	t.timers = append(t.timers, td)
	t.up(td.index)
	if td.index == 0 {
		// if first node, signal start goroutine
		d = td.Delay()
		t.signal.Reset(d)
		if Debug {
			log.Debug("timer: add reset delay %d ms", int64(d)/int64(itime.Millisecond))
		}
	}
	if Debug {
		log.Debug("timer: push item key: %s, expire: %s, index: %d", td.Key, td.ExpireString(), td.index)
	}
	return
}

func (t *Timer) del(td *TimerData) {
	var (
		i    = td.index
		last = len(t.timers) - 1
	)
	if i < 0 || i > last || t.timers[i] != td {
		// already remove, usually by expire
		if Debug {
			log.Debug("timer del i: %d, last: %d, %p", i, last, td)
		}
		return
	}
	if i != last {
		t.swap(i, last)
		t.down(i, last)
		t.up(i)
	}
	// remove item is the last node
	t.timers[last].index = -1 // for safety
	t.timers = t.timers[:last]
	if Debug {
		log.Debug("timer: remove item key: %s, expire: %s, index: %d", td.Key, td.ExpireString(), td.index)
	}
	return
}

```

那么，timer定时这块是怎么实现的呢？

```golang
func (t *Timer) init(num int) {
	t.signal = itime.NewTimer(infiniteDuration)
	t.timers = make([]*TimerData, 0, num)
	t.num = num
	t.grow()
	go t.start() // 此处开始轮询
}
```

可以看到，只启动了一个gorutine来管理所有的timer,start内部是一个无限循环，expire()负责设置一个最近的定期器，然后阻塞等待即可。

```golang
func (t *Timer) start() {
	for {
		t.expire()
		<-t.signal.C
	}
}

func (t *Timer) expire() {
	var (
		fn func()
		td *TimerData
		d  itime.Duration
	)
	t.lock.Lock()
	for {
		if len(t.timers) == 0 { // 没有定时器，无限睡眠
			d = infiniteDuration
			if Debug {
				log.Debug("timer: no other instance")
			}
			break
		}
		td = t.timers[0] // 取第一个元素，如何还没到期就根据剩余时间重置定时器
		if d = td.Delay(); d > 0 {
			break
		}
		fn = td.fn
		// let caller put back, usually by Del()
		t.del(td) // 从堆中删除
		t.lock.Unlock()
		if fn == nil {
			log.Warn("expire timer no fn")
		} else {
			if Debug {
				log.Debug("timer key: %s, expire: %s, index: %d expired, call fn", td.Key, td.ExpireString(), td.index)
			}
			fn() // 执行回调
		}
		t.lock.Lock()
	}
	t.signal.Reset(d)
	if Debug {
		log.Debug("timer: expire reset delay %d ms", int64(d)/int64(itime.Millisecond))
	}
	t.lock.Unlock()
	return
}
```

### Buffer Pool
Pool可以理解成内存，free是空闲内存的指针，Buffer是内存分配的基本单元
```golang
type Pool struct {
	lock sync.Mutex
	free *Buffer
	max  int
	num  int
}

type Buffer struct {
	buf  []byte
	next *Buffer // next free buffer
}
```
在Pool初始化时，会申请一块很大的buf，然后构建Buffer链表，每个Buffer都通过slice指向这个buf的一部分。

```golang
func (p *Pool) grow() {
	var (
		i   int
		b   *Buffer
		bs  []Buffer
		buf []byte
	)
	buf = make([]byte, p.max)
	bs = make([]Buffer, p.num)
	p.free = &bs[0]
	b = p.free
	for i = 1; i < p.num; i++ {
		b.buf = buf[(i-1)*p.size : i*p.size]
		b.next = &bs[i]
		b = b.next
	}
	b.buf = buf[(i-1)*p.size : i*p.size]
	b.next = nil
	return
}

```

这样，申请和释放Buffer只需要操作链表的指针即可，复杂度O（1）

```golang
// Get get a free memory buffer.
func (p *Pool) Get() (b *Buffer) {
	p.lock.Lock()
	if b = p.free; b == nil {
		p.grow()
		b = p.free
	}
	p.free = b.next
	p.lock.Unlock()
	return
}

// Put put back a memory buffer to free.
func (p *Pool) Put(b *Buffer) {
	p.lock.Lock()
	b.next = p.free
	p.free = b
	p.lock.Unlock()
	return
}

```


### Router模块中的Cleaner

Cleaner中主要使用了一条双向循环链表，但额外设计了一个map，来提供快速的节点定位。

```
type CleanData struct {
	Key        int64
	expireTime time.Time
	next, prev *CleanData
}

type Cleaner struct {
	cLock sync.Mutex
	size  int
	root  CleanData
	maps  map[int64]*CleanData
}

func (c *Cleaner) PushFront(key int64, expire time.Duration) {
	c.cLock.Lock()
	if e, ok := c.maps[key]; ok {
		// update time
		e.expireTime = time.Now().Add(expire)
		c.moveToFront(e)
	} else {
		e = new(CleanData)
		e.Key = key
		e.expireTime = time.Now().Add(expire)
		c.maps[key] = e
		at := &c.root
		n := at.next
		at.next = e
		e.prev = at
		e.next = n
		n.prev = e
		c.size++
	}
	c.cLock.Unlock()
}
```

### Comet模块中的RingBuf

