---
title: goim-comet模块源码分析
date: 2018-03-09 10:04:31
tags:
    - golang
    - goim源码分析
---

comet是客户端直接连接的节点，设计上是无状态的。通过rpc与logic服务交互，对外提供TCP、HTTP、WebSocket连接方式，自己也作为push这个rpc服务的提供方

```golang
//main.go

    if err := InitLogicRpc(Conf.LogicAddrs); err != nil {
        log.Warn("logic rpc current can't connect, retry")
    }
    // start monitor
    if Conf.MonitorOpen {
        InitMonitor(Conf.MonitorAddrs)
    }
    
    ... //bucket round server的初始化，下面会讲

    // white list
    // tcp comet
    if err := InitTCP(Conf.TCPBind, Conf.MaxProc); err != nil {
        panic(err)
    }
    // websocket comet
    if err := InitWebsocket(Conf.WebsocketBind); err != nil {
        panic(err)
    }
    // flash safe policy
    if Conf.FlashPolicyOpen {
        if err := InitFlashPolicy(); err != nil {
            panic(err)
        }
    }
    // wss comet
    if Conf.WebsocketTLSOpen {
        if err := InitWebsocketWithTLS(Conf.WebsocketTLSBind, Conf.WebsocketCertFile, Conf.WebsocketPrivateFile); err != nil {
            panic(err)
        }
    }
    // start rpc
    if err := InitRPCPush(Conf.RPCPushAddrs); err != nil {
        panic(err)
    }
```

## InitTCP
InitXXX的作用是暴露不同的服务给客户端使用，选一个看就可以了。

在多个gorutine中调用了AcceptTCP，充分发挥多核能力
```golang
        for i := 0; i < accept; i++ {
            go acceptTCP(DefaultServer, listener)
        }
```

accept之后，核心逻辑实现在serveTCP中,首先调用auth服务，获得subKey,然后把channel放进bucket里

```golang
    // ... 
    if p, err = ch.CliProto.Set(); err == nil {
        if key, ch.RoomId, hb, err = server.authTCP(rr, wr, p); err == nil {
            b = server.Bucket(key)
            err = b.Put(key, ch)
        }
    }

```

serveTCP方法中，当前gorutine负责读数据，处理心跳，把数据封装成proto对象然后保存到channel的CliProto中，然后通知dispatchTCP处理
```golang
for {
        if p, err = ch.CliProto.Set(); err != nil { // 从channel中申请一个buffer用来存放proto
            break
        }
        if white {
            DefaultWhitelist.Log.Printf("key: %s start read proto\n", key)
        }
        if err = p.ReadTCP(rr); err != nil { // 读proto
            break
        }
        if white {
            DefaultWhitelist.Log.Printf("key: %s read proto:%v\n", key, p)
        }
        if p.Operation == define.OP_HEARTBEAT { // 维持心跳
            tr.Set(trd, hb)
            p.Body = nil
            p.Operation = define.OP_HEARTBEAT_REPLY
            if Debug {
                log.Debug("key: %s receive heartbeat", key)
            }
        } else {
            if err = server.operator.Operate(p); err != nil {
                break
            }
        }
        if white {
            DefaultWhitelist.Log.Printf("key: %s process proto:%v\n", key, p)
        }
        ch.CliProto.SetAdv()
        ch.Signal() //通知dispatchTCP处理
        if white {
            DefaultWhitelist.Log.Printf("key: %s signal\n", key)
        }
    }
```

dispatchTCP中，如果收到proto.ProtoReady，就表示读取到了一个proto，然后原样写回？

```golang
        case proto.ProtoReady:
            // fetch message from svrbox(client send)
            for {
                if p, err = ch.CliProto.Get(); err != nil {
                    err = nil // must be empty error
                    break
                }
                if white {
                    DefaultWhitelist.Log.Printf("key: %s start write client proto%v\n", key, p)
                }
                if err = p.WriteTCP(wr); err != nil {
                    goto failed
                }
                if white {
                    DefaultWhitelist.Log.Printf("key: %s write client proto%v\n", key, p)
                }
                p.Body = nil // avoid memory leak
                ch.CliProto.GetAdv()
            }

```

### Round
goim自己进行了buffer的管理，避免了频繁申请内存的开销。通过自定义的Pool结构来分配Buffer，因为分配时要加锁，使用Round来组合多个Pool，通过mod运算随机获取一个Pool，来减缓锁的争用。

```golang
// round.go
// Reader get a reader memory buffer.
func (r *Round) Reader(rn int) *bytes.Pool {
    return &(r.readers[rn%r.options.Reader])
}

// Writer get a writer memory buffer pool.
func (r *Round) Writer(rn int) *bytes.Pool {
    return &(r.writers[rn%r.options.Writer])
}
```

Pool内部使用一条单链表，维护一个free指针指向未分配的buffer

```golang
// libs/buffer.go

func (p *Pool) grow() {
    var (
        i   int
        b   *Buffer
        bs  []Buffer
        buf []byte
    )
    buf = make([]byte, p.max) // 所有的Buffer都从这里分配
    bs = make([]Buffer, p.num) // Buffer数组
    p.free = &bs[0] //构造Buffer链
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
```

### Timer
goim的Timer也是基于堆结构改写的，内部只有一个timer，不断把定时器设置成堆顶元素的触发时间来提高效率。

### Channel

TCP连接会被封装到Channel这个结构中，使用CliProto来处理封包拆包

```golang
type Channel struct {
    RoomId   int32
    CliProto Ring
    signal   chan *proto.Proto
    Writer   bufio.Writer
    Reader   bufio.Reader
    Next     *Channel
    Prev     *Channel
}
```
#### Ring
Ring是Channel内部用来保存并重用proto的一个结构
```
type Ring struct {
    // read
    rp   uint64
    num  uint64
    mask uint64
    // TODO split cacheline, many cpu cache line size is 64
    // pad [40]byte
    // write
    wp   uint64
    data []proto.Proto
}
```

### Bucket
bucket是channel的容器
```golang
//main.go

    buckets := make([]*Bucket, Conf.Bucket)
    for i := 0; i < Conf.Bucket; i++ {
        buckets[i] = NewBucket(BucketOptions{
            ChannelSize:   Conf.BucketChannel,
            RoomSize:      Conf.BucketRoom,
            RoutineAmount: Conf.RoutineAmount,
            RoutineSize:   Conf.RoutineSize,
        })
    }

```
