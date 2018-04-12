---
title: Time Wheel-高效定时器
date: 2018-04-12 10:44:25
tags:
    - golang
    - 数据结构
---

之前的文章中有提到过基于最小堆构建定时器的方式，这里介绍另一种更酷的实现--Time Wheel(时间轮)

### 数据结构分析

![1348926970_9123.png](http://7xqlni.com1.z0.glb.clouddn.com/1348926970_9123.png)

解释一下上图，环里是一个个的时间间隔，链表中是到达这个时间点后需要触发的定时事件。

举个例子，假设环中的一格代表1s。

如果我有一个需要延迟3s执行的任务，当前TimeWheel位于1处，那么就把它放入第4格的链表中。

如果需要延迟10s，那么10%8 + 1 = 3， 就放入3处。当然，我们需要记录一下，这个事件需要等TimeWheel第二次转到3处才能触发。

> 复杂度

底层结构 | 添加定时器 | 触发定时器 | PerTickBookkeeping
---- | ---- | ---- | ---- |
最小堆 | O(lg(n)) | O(1) | O(1)
时间轮 | O(1) | O(1) | O(1)

> 缺点与优化

可以看出，如果我需要延迟1天执行，那么会创建一个很大的环形链表，虽然可以通过降低精度来减少内存消耗，但我们需要更好的方案。

我们可以组合多个timewheel，来减少内存开销，同时保持很高的效率

![TimingWheels2.png](http://7xqlni.com1.z0.glb.clouddn.com/TimingWheels2.png)


### 实现

```golang
package timewheel

import (
    "container/list"
    "time"
)

// @author qiang.ou<qingqianludao@gmail.com>

// Job 延时任务回调函数
type Job func(TaskData)

// TaskData 回调函数参数类型
type TaskData map[interface{}]interface{}

// TimeWheel 时间轮
type TimeWheel struct {
    interval time.Duration // 指针每隔多久往前移动一格
    ticker   *time.Ticker
    slots    []*list.List // 时间轮槽
    // key: 定时器唯一标识 value: 定时器所在的槽, 主要用于删除定时器, 不会出现并发读写，不加锁直接访问
    timer             map[interface{}]int
    currentPos        int              // 当前指针指向哪一个槽
    slotNum           int              // 槽数量
    job               Job              // 定时器回调函数
    addTaskChannel    chan Task        // 新增任务channel
    removeTaskChannel chan interface{} // 删除任务channel
    stopChannel       chan bool        // 停止定时器channel
}

// Task 延时任务
type Task struct {
    delay  time.Duration // 延迟时间
    circle int           // 时间轮需要转动几圈
    key    interface{}   // 定时器唯一标识, 用于删除定时器
    data   TaskData      // 回调函数参数
}

// New 创建时间轮
func New(interval time.Duration, slotNum int, job Job) *TimeWheel {
    if interval <= 0 || slotNum <= 0 || job == nil {
        return nil
    }
    tw := &TimeWheel{
        interval:          interval,
        slots:             make([]*list.List, slotNum),
        timer:             make(map[interface{}]int),
        currentPos:        0,
        job:               job,
        slotNum:           slotNum,
        addTaskChannel:    make(chan Task),
        removeTaskChannel: make(chan interface{}),
        stopChannel:       make(chan bool),
    }

    for i := 0; i < tw.slotNum; i++ {
        tw.slots[i] = list.New()
    }

    return tw
}

// Start 启动时间轮
func (tw *TimeWheel) Start() {
    tw.ticker = time.NewTicker(tw.interval)
    go tw.start()
}

// Stop 停止时间轮
func (tw *TimeWheel) Stop() {
    tw.stopChannel <- true
}

// AddTimer 添加定时器 key为定时器唯一标识
func (tw *TimeWheel) AddTimer(delay time.Duration, key interface{}, data TaskData) {
    if delay <= 0 || key == nil {
        return
    }
    tw.addTaskChannel <- Task{delay: delay, key: key, data: data}
}

// RemoveTimer 删除定时器 key为添加定时器时传递的定时器唯一标识
func (tw *TimeWheel) RemoveTimer(key interface{}) {
    if key == nil {
        return
    }
    tw.removeTaskChannel <- key
}

func (tw *TimeWheel) start() {
    for {
        select {
        case <-tw.ticker.C:
            tw.tickHandler()
        case task := <-tw.addTaskChannel:
            tw.addTask(&task)
        case key := <-tw.removeTaskChannel:
            tw.removeTask(key)
        case <-tw.stopChannel:
            tw.ticker.Stop()
            return
        }
    }
}

func (tw *TimeWheel) tickHandler() {
    l := tw.slots[tw.currentPos]
    tw.scanAndRunTask(l)
    if tw.currentPos == tw.slotNum-1 {
        tw.currentPos = 0
    } else {
        tw.currentPos++
    }
}

// 扫描链表中过期定时器, 并执行回调函数
func (tw *TimeWheel) scanAndRunTask(l *list.List) {
    for e := l.Front(); e != nil; {
        task := e.Value.(*Task)
        if task.circle > 0 {
            task.circle--
            e = e.Next()
            continue
        }

        go tw.job(task.data)
        next := e.Next()
        l.Remove(e)
        delete(tw.timer, task.key)
        e = next
    }
}

// 新增任务到链表中
func (tw *TimeWheel) addTask(task *Task) {
    pos, circle := tw.getPositionAndCircle(task.delay)
    task.circle = circle

    tw.slots[pos].PushBack(task)

    tw.timer[task.key] = pos
}

// 获取定时器在槽中的位置, 时间轮需要转动的圈数
func (tw *TimeWheel) getPositionAndCircle(d time.Duration) (pos int, circle int) {
    delaySeconds := int(d.Seconds())
    intervalSeconds := int(tw.interval.Seconds())
    circle = int(delaySeconds / intervalSeconds / tw.slotNum)
    pos = int(tw.currentPos+delaySeconds/intervalSeconds) % tw.slotNum

    return
}

// 从链表中删除任务
func (tw *TimeWheel) removeTask(key interface{}) {
    // 获取定时器所在的槽
    position, ok := tw.timer[key]
    if !ok {
        return
    }
    // 获取槽指向的链表
    l := tw.slots[position]
    for e := l.Front(); e != nil; {
        task := e.Value.(*Task)
        if task.key == key {
            delete(tw.timer, task.key)
            l.Remove(e)
        }

        e = e.Next()
    }
}
```

### 参考

[https://github.com/ouqiang/timewheel](https://github.com/ouqiang/timewheel)
[https://www.ibm.com/developerworks/cn/linux/l-cn-timers/](https://www.ibm.com/developerworks/cn/linux/l-cn-timers/)
[http://www.lpnote.com/2017/11/16/hashed-and-hierarchical-timing-wheels](http://www.lpnote.com/2017/11/16/hashed-and-hierarchical-timing-wheels)
[https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/](https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/)