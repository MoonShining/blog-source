---
title: goim-router模块源码分析
date: 2018-03-07 16:17:13
tags: 
    - golang
    - goim源码分析
---

这个模块是用于保存状态信息的(例如在线的session)

文档里是这样描述的
router 属于有状态节点，logic可以使用一致性hash配置节点，增加多个router节点（目前还不支持动态扩容），提前预估好在线和压力情况

从main.go入手

```golang
func main() {
    flag.Parse()
    if err := InitConfig(); err != nil {
        panic(err)
    }
    runtime.GOMAXPROCS(Conf.MaxProc)
    log.LoadConfiguration(Conf.Log)
    defer log.Close()
    log.Info("router[%s] start", VERSION)
    // start prof
    perf.Init(Conf.PprofAddrs)
    // start monitor
    if Conf.MonitorOpen {
        InitMonitor(Conf.MonitorAddrs)
    }
    // start rpc
    buckets := make([]*Bucket, Conf.Bucket)
    for i := 0; i < Conf.Bucket; i++ {
        buckets[i] = NewBucket(Conf.Session, Conf.Server, Conf.Cleaner)
    }
    if err := InitRPC(buckets); err != nil {
        panic(err)
    }
    // block until a signal is received.
    InitSignal()
}
```

忽略flag和config部分的处理，这里主要涉及了**perf**，**monitor**监控，主**RPC**逻辑,以及**signal**处理

### perf

内部使用了net/http/pprof进行性能分析

```golang
func Init(pprofBind []string) {
    pprofServeMux := http.NewServeMux()
    pprofServeMux.HandleFunc("/debug/pprof/", pprof.Index)
    pprofServeMux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    pprofServeMux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    pprofServeMux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    for _, addr := range pprofBind {
        go func() {
            if err := http.ListenAndServe(addr, pprofServeMux); err != nil {
                log.Error("http.ListenAndServe(\"%s\", pprofServeMux) error(%v)", addr, err)
                panic(err)
            }
        }()
    }
}
```

### monitor

是一个简单ping请求处理,一看就是用来监测服务存活状态的

```golang
func InitMonitor(binds []string) {
    m := new(Monitor)
    monitorServeMux := http.NewServeMux()
    monitorServeMux.HandleFunc("/monitor/ping", m.Ping)
    for _, addr := range binds {
        go func(bind string) {
            if err := http.ListenAndServe(bind, monitorServeMux); err != nil {
                log.Error("http.ListenAndServe(\"%s\", pprofServeMux) error(%v)", addr, err)
                panic(err)
            }
        }(addr)
    }
}

// monitor ping
func (m *Monitor) Ping(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("ok"))
}
```

### signal
信号处理，收到SIGHUP重载配置，这是符合linux上惯用约定的

```golang
func InitSignal() {
    c := make(chan os.Signal, 1)
    signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT, syscall.SIGSTOP)
    for {
        s := <-c
        log.Info("router[%s] get a signal %s", VERSION, s.String())
        switch s {
        case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGSTOP, syscall.SIGINT:
            return
        case syscall.SIGHUP:
            reload()
        default:
            return
        }
    }
}

func reload() {
    newConf, err := ReloadConfig()
    if err != nil {
        log.Error("ReloadConfig() error(%v)", err)
        return
    }
    Conf = newConf
}
```

## 主RPC逻辑

这里有Session、Cleaner、Bucket这三个主要的结构

Bucket是Session的容器，为了减少锁争夺，会有多个Bucket，根据用户id与Bucket数量进行mod运算来确定，这个Session放到哪个Bucket中，是一种很常见的sharding

```golang
//rpc.go
func (r *RouterRPC) bucket(userId int64) *Bucket {
    idx := int(userId % r.BucketIdx) // mod
    // fix panic
    if idx < 0 {
        idx = 0
    }
    return r.Buckets[idx]
}

func (r *RouterRPC) Put(arg *proto.PutArg, reply *proto.PutReply) error {
    reply.Seq = r.bucket(arg.UserId).Put(arg.UserId, arg.Server, arg.RoomId)
    return nil
}

// bucket.go
func (b *Bucket) Put(userId int64, server int32, roomId int32) (seq int32) {
    var (
        s  *Session
        ok bool
    )
    b.bLock.Lock() //加锁，只影响这个Bucket
    if s, ok = b.sessions[userId]; !ok {
        s = NewSession(b.server)
        b.sessions[userId] = s
    }
    if roomId != define.NoRoom {
        seq = s.PutRoom(server, roomId)
    } else {
        seq = s.Put(server)
    }
    b.counter(userId, server, roomId, true)
    b.bLock.Unlock()
    return
}

```

Cleaner是与Session一一对应的一个结构，用于清理Session信息,每个Bucket会有一个单独的gorutine进行定时清理

```golang
//bucket.go
func NewBucket(session, server, cleaner int) *Bucket {
    b := new(Bucket)
    b.sessions = make(map[int64]*Session, session)
    b.roomCounter = make(map[int32]int32)
    b.serverCounter = make(map[int32]int32)
    b.userServerCounter = make(map[int32]map[int64]int32)
    b.cleaner = NewCleaner(cleaner)
    b.server = server
    b.session = session
    go b.clean() //启动清理gorutine
    return b
}

func (b *Bucket) clean() {
    var (
        i       int
        userIds []int64
    )
    for {
        userIds = b.cleaner.Clean()
        if len(userIds) != 0 {
            b.bLock.Lock()
            for i = 0; i < len(userIds); i++ {
                b.delEmpty(userIds[i]) 从sessions map中删掉对应的session
            }
            b.bLock.Unlock()
            continue
        }
        time.Sleep(Conf.BucketCleanPeriod) //休息一段时间
    }
}
```

Cleaner本身的结构是经过精心设计的，使用了一条双向循环链表来记录当前所有Session的信息，为了克服移除一个节点时需要遍历链表，额外用了一个map来快速定位到节点，然后操作这个节点的指针来进行删除

```golang
func (c *Cleaner) remove(key int64) {
    if e, ok := c.maps[key]; ok { //通过map定位
        delete(c.maps, key) // 从map中删除
        // 从链表中删除
        e.prev.next = e.next
        e.next.prev = e.prev
        e.next = nil // avoid memory leaks
        e.prev = nil // avoid memory leaks
        c.size--
    }
}

func (c *Cleaner) Clean() (keys []int64) {
    var (
        i int
        e *CleanData
    )
    keys = make([]int64, 0, maxCleanNum)
    c.cLock.Lock()
    for i = 0; i < maxCleanNum; i++ { // 每次最多只清理maxCleanNum个节点，don't know why
        if e = c.back(); e != nil {
            if e.expire() {
                c.remove(e.Key)
                keys = append(keys, e.Key)
                continue
            }
        }
        break
    }
    // next time
    c.cLock.Unlock()
    return
}
```

有一个问题是，没有看懂过期的逻辑。从链表尾部开始清理，却在Del时把节点放到头部？
