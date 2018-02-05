---
title: golang tcp拆包的正确姿势
date: 2018-02-05 11:07:59
tags: 造轮子
---

最近在造一个叫im-go的服务，看名字也能猜出来，是一个基于Go的IM服务，因为不想引入任何的依赖库，所以是手写每个模块的。

之前看过Netty，于是也想做一个类似Netty Codec的，用于编码解码的模块, 方便地处理TCP粘包这种细节问题。

在网上做了一番搜索之后，发现排名靠前的实现，要么出乎意料地复杂，要么根本就是完全错误的，例如

出乎意料的复杂：

+ [解决golang开发socket服务时粘包半包bug](http://xiaorui.cc/2016/03/08/%E8%A7%A3%E5%86%B3golang%E5%BC%80%E5%8F%91socket%E6%9C%8D%E5%8A%A1%E6%97%B6%E7%B2%98%E5%8C%85%E5%8D%8A%E5%8C%85bug)
+ [http://www.01happy.com/golang-tcp-socket-adhere/](golang中tcp socket粘包问题和处理)

错误的：

+ [https://victoriest.gitbooks.io/golang-tcp-server/content/chapter4.html](服务器的粘包处理)

分析一下这个错误的实现

```go
func Decode(reader *bufio.Reader) (string, error) {
    lengthByte, _ := reader.Peek(4)
    lengthBuff := bytes.NewBuffer(lengthByte)
    var length int32
    err := binary.Read(lengthBuff, binary.LittleEndian, &length)
    if err != nil {
        return "", err
    }
    if int32(reader.Buffered()) < length+4 {
        return "", err
    }

    // 假设执行到了这里，那么已经成功读取了长度到length这个变量中
    pack := make([]byte, int(4+length))
    _, err = reader.Read(pack) //这里是不能保证就能完读到length长度的数据的!!
    if err != nil {
        return "", err
    }
    return string(pack[4:]), nil
}
```

我也受了它的误导，基于[Peek()](https://golang.org/pkg/bufio/#Reader.Peek)做了一个非常复杂的实现

### 正确的姿势
在翻了翻io和bufio这两个包之后，我找到了[ReadFull](https://golang.org/pkg/io/#ReadFull)

ReadFull，就是调用了ReadAtLeast

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    return ReadAtLeast(r, buf, len(buf))
}
```



```go
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error) {
    if len(buf) < min {
        return 0, ErrShortBuffer
    }
    for n < min && err == nil {
        var nn int
        nn, err = r.Read(buf[n:])
        n += nn
    }
    if n >= min {
        err = nil
    } else if n > 0 && err == EOF {
        err = ErrUnexpectedEOF
    }
    return
}
```

标准库里的ReadAtLeast就非常优雅了，用n记录读取的总字节数，nn是每次读取到的字节数，一看就明白。

基于ReadFull的拆包代码

```go
func (c *LenthCodec) Decode(conn net.Conn) (bodyBuf []byte, err error) {
    lengthBuf := make([]byte, 4)
    _, err = io.ReadFull(conn, lengthBuf)
    //check error
    length := binary.LittleEndian.Uint32(lengthBuf)
    
    bodyBuf = make([]byte, length)
    _, err = io.ReadFull(conn, bodyBuf)
    //check error
    return
}
```
