---
title: FishChatServer源码探究
date: 2018-02-05 17:53:02
tags:
    - golang
    - 造轮子
---

在写im-go的过程中遇到了一些设计上的问题，于是想找目前有的开源im服务的源码看看。[FishChatServer2](https://github.com/oikomi/FishChatServer2)在一些模块设计上和我的思路很相似，有种英雄所见略同的快感，所以选了它(FishChatServer2的拆包方式和我[上一篇文章](https://moonshining.github.io/blog/2018/02/05/golang-tcp%E6%8B%86%E5%8C%85%E7%9A%84%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF)中提到的使用`ReadFull`的方式是一样的，并且连模块名字都一样叫Codec)

主要看了libnet和server两个模块

libnet, 是所有server的基础公共库，封装了诸如Listen Accept之类的调用
![](http://7xqlni.com1.z0.glb.clouddn.com/libnet.png)

server, 具体的服务，看了一下gateway和access两个服务的实现
![](http://7xqlni.com1.z0.glb.clouddn.com/server.png)

---

### gateway服务
**gateway.go**是gateway服务的入口，其实是一个access服务的负载均衡器，核心代码如下

```go
    // 初始化对象
    gwServer := server.New() 
    // codec编码、解码器
    protobuf := codec.Protobuf() 
    // 使用libnet封装的api进行Listen
    if gwServer.Server, err = libnet.Serve(conf.Conf.Server.Proto, conf.Conf.Server.Addr, protobuf, 0); err != nil {
        glog.Error(err)
        panic(err)
    } 
    // 通过etcd进行服务发现，每5秒向etcd请求一个access服务器列表，并写入AccessServerList 这个变量中
    go job.ConfDiscoveryProc() 
    gwServer.Loop() // 开始不断循环处理请求
```

`gwServer.Loop()`的核心代码在**server.go**中

```go
func (s *Server) sessionLoop(client *client.Client) {
    for {
        //读一个包
        reqData, err := client.Session.Receive() 
        if err != nil {
            glog.Error(err)
        }
        if reqData != nil {
            baseCMD := &external.Base{}
            // protobuf 反序列化
            if err = proto.Unmarshal(reqData, baseCMD); err != nil {
                if err = client.Session.Send(&external.Error{
                    Cmd:     external.ErrServerCMD,
                    ErrCode: ecode.ServerErr.Uint32(),
                    ErrStr:  ecode.ServerErr.String(),
                }); err != nil {
                    glog.Error(err)
                }
                continue
            }
            // client.Parse方法有一点迷惑性，client.Parse 其实做了解析命令，并执行命令的工作
            if err = client.Parse(baseCMD.Cmd, reqData); err != nil {
                glog.Error(err)
                continue
            }
        }
    }
}

func (s *Server) Loop() {
    glog.Info("loop")
    for {
        // 获取libnet封装的session对象
        session, err := s.Server.Accept() 
        if err != nil {
            glog.Error(err)
        }
        // 生成client对象，里面封装了gateway服务的业务逻辑
        go s.sessionLoop(client.New(session)) 
    }
}
```


client.Parse最终调用了**proto_proc.go**里的**client.procReqAccessServer**来执行业务逻辑
```go
func (c *Client) procReqAccessServer(reqData []byte) (err error) {
    var addr string
    var accessServerList []string
    // 从之前提到的access服务地址数组中获取一个可用的access服务
    // 没有看懂为什么要做一次额外的复制数组的操作？
    for _, v := range job.AccessServerList {
        accessServerList = append(accessServerList, v.IP)
    }

    // 处理错误情况
    if len(accessServerList) == 0 {
        if err = c.Session.Send(&external.ResSelectAccessServerForClient{
            Cmd:     external.ReqAccessServerCMD,
            ErrCode: ecode.NoAccessServer.Uint32(),
            ErrStr:  ecode.NoAccessServer.String(),
        }); err != nil {
            glog.Error(err)
        }
        return
    }

    // 返回一个可用地址
    addr = accessServerList[rand.Intn(len(accessServerList))]
    if err = c.Session.Send(&external.ResSelectAccessServerForClient{
        Cmd:     external.ReqAccessServerCMD,
        ErrCode: ecode.OK.Uint32(),
        ErrStr:  ecode.OK.String(),
        Addr:    addr,
    }); err != nil {
        glog.Error(err)
    }
    return
}
```

到此一次请求就结束了，可用看出代码的结构上非常清晰，很容易就能理解。

---

### libnet

这个模块帮我们屏蔽了大量繁琐的网络细节，接下来就要看一下它的实现了。

从**api.go**入手，这里定义了对外的接口

```go
type Protocol interface { 
    // Codec 负责通信协议的解析，封装了读写数据的方法
    NewCodec(rw io.ReadWriter) Codec 
}

type Codec interface {
    Receive() ([]byte, error)
    Send(interface{}) error
    Close() error
}

func Serve(network, address string, protocol Protocol, sendChanSize int) (*Server, error) {
    listener, err := net.Listen(network, address) // 终于看到标准库里的东西了
    if err != nil {
        return nil, err
    }
    // listener用于Accept， protocol用户处理net.Conn, sendChanSize看上去好像是用来控制发送速率的，不过没有明白为什么需要控制?
    return NewServer(listener, protocol, sendChanSize), nil 
}

// 客户端连接+带超时的连接
func Connect(network, address string, protocol Protocol, sendChanSize int) (*Session, error) {
    conn, err := net.Dial(network, address)
    if err != nil {
        return nil, err
    }
    return NewSession(protocol.NewCodec(conn), sendChanSize), nil
}

func ConnectTimeout(network, address string, timeout time.Duration, protocol Protocol, sendChanSize int) (*Session, error) {
    conn, err := net.DialTimeout(network, address, timeout)
    if err != nil {
        return nil, err
    }
    return NewSession(protocol.NewCodec(conn), sendChanSize), nil
}
```

跳过客户的部分的实现，探索一下**server.go**,负责Accept一个连接，并且封装好一个session对象返回

```go
func (server *Server) Accept() (*Session, error) {
    var tempDelay time.Duration
    for {
        conn, err := server.listener.Accept()
        if err != nil {
            // 处理Temporary Error应该是参考了goblog里的error-handling-and-go章节
            // For instance, a web crawler might sleep and retry when it encounters a temporary error and give up otherwise.
            if ne, ok := err.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                time.Sleep(tempDelay)
                continue
            }
            // 感觉直接比较字符串有点太粗暴了？ 但应该是没有办法区分的原因
            if strings.Contains(err.Error(), "use of closed network connection") {
                return nil, io.EOF
            }
            return nil, err
        }
        return server.manager.NewSession(
            server.protocol.NewCodec(conn),
            server.sendChanSize,
        ), nil
    }
}
```

**manager.go**用于管理session，会把session根据id mod 32以后，放进对应的map里, 这里使用了lock来保证并发安全, 但golang1.9以后，应该可以用内置的**sync.Map**替代了

```go

func (manager *Manager) NewSession(codec Codec, sendChanSize int) *Session {
    session := newSession(manager, codec, sendChanSize)
    manager.putSession(session)
    return session
}

func (manager *Manager) putSession(session *Session) {
    smap := &manager.sessionMaps[session.id%sessionMapNum]
    smap.Lock()
    defer smap.Unlock()
    smap.sessions[session.id] = session
    manager.disposeWait.Add(1)
}
```

---

### Session

server在Accept之后，返回的是一个session对象,session负责收发数据，并且实现了**优雅退出(gracefully shutdown)**

```go
type Session struct {
    id             uint64
    codec          Codec
    manager        *Manager
    sendChan       chan interface{}
    closeFlag      int32
    closeChan      chan int
    closeMutex     sync.Mutex
    closeCallbacks *list.List
    State          interface{}
}
```

优雅退出的实现，先通过CAS设置一下closeFlag, 成功设置的gorutine可以执行清理操作，失败的gorutine返回SessionClosedError
```go
func (session *Session) Close() error {
    // 如果成功通过CAS设置了closeFlag
    if atomic.CompareAndSwapInt32(&session.closeFlag, 0, 1) {
        err := session.codec.Close() // 关闭net.Conn
        close(session.closeChan) // 退出sendLoop
        if session.manager != nil { // 从manager中移除session
            session.manager.delSession(session)
        }
        session.invokeCloseCallbacks() // 执行callback
        return err
    }
    return SessionClosedError
}
```

发送数据部分

```
func (session *Session) sendLoop() {
    defer session.Close()
    for {
        // 使用select语句来保证，关闭closeChan之后可以退出sendLoop
        select {
        case msg := <-session.sendChan:
            if session.codec.Send(msg) != nil {
                return
            }
        case <-session.closeChan:
            return
        }
    }
}

func (session *Session) Send(msg interface{}) (err error) {
    // 在每次Send的时候，都会检查closeFlag，实现快速的退出
    if session.IsClosed() {
        return SessionClosedError
    }
    if session.sendChan == nil {
        return session.codec.Send(msg)
    }

    // send block, 返回一个异常, 有点粗暴了
    select {
    case session.sendChan <- msg:
        return nil
    default:
        return SessionBlockedError
    }
}
```

---

### 最后

其实本意是想找找有没有关于心跳和连接保持方面的代码，但没有什么收获.不过也看到了很多高质量的实现，例如**idgen**，粗粗瞟了一眼就发现，应该是使用了雪花算法，此外还有大量微服务的设计，以及一些我很感兴趣的流行开源技术栈(k8s docker etcd hbase kafka)可以看出是一整套经过深思熟虑的系统，决定过年期间要好好看一看这个库，吸收一下营养。