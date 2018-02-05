---
title: FishChatServer源码探究
date: 2018-02-05 17:53:02
tags:
---

在写im-go的过程中遇到了一些设计上的问题，于是找了一下目前有的开源im服务的源码看看,我选的是[FishChatServer2](https://github.com/oikomi/FishChatServer2)

选它的原因是因为感觉和作者的思路很相似，FishChatServer2的拆包方式和我[上一篇文章](https://moonshining.github.io/blog/2018/02/05/golang-tcp%E6%8B%86%E5%8C%85%E7%9A%84%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF)中提到的使用`ReadFull`的方式是一样的，并且连模块名字都一样叫Codec

主要看了下面两个模块

libnet, 是所有server的基础公共库，封装了诸如Listen Accept之类的调用
![](http://7xqlni.com1.z0.glb.clouddn.com/libnet.png)

server, 具体的服务，看了一下gateway和access两个服务的实现
![](http://7xqlni.com1.z0.glb.clouddn.com/server.png)

### gateway服务
**gateway.go**是gateway服务的入口，核心代码如下

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

到此一次请求就结束了，可用看出代码的结构上非常清晰，很容易就能理解