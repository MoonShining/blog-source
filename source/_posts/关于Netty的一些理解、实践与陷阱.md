---
title: 关于Netty的一些理解、实践与陷阱
date: 2017-06-05 09:49:00
tags:
    - 异步IO
    - 后端
---

### 核心概念的理解
Netty对于网络层进行了自己的抽象，用Channel表示连接，读写就是Channel上发生的事件，ChannelHandler用来处理这些事件，ChannelPipeline基于unix哲学提供了一种优雅的组织ChannelHandler的方式，用管道解耦不同层面的处理。现在回过头来看看，真的是非常天才和优雅的设计，是我心中API设计的典范之一了。

### TCP半包、粘包
使用Netty内置的**LineBasedFrameDecoder**或者**LengthFieldBasedFrameDecoder **,我们只要在pipeline中添加，就解决了这个问题。

### Writtable问题
有时候，由于TCP的send buffer满了，向channel的写入会失败。我们需要检查**channel().isWritable()**标记来确定是否执行写入。

### 处理耗时任务

Netty In Action以及网上的一些资料中，都没有很直接的展示如何在Netty中去处理耗时任务。其实也很简单，只要给handler指定一个事件循环就可以，例如

```java
public class MyChannelInitializer extends ChannelInitializer<Channel> {
    private static EventExecutorGroup longTaskGroup = new DefaultEventExecutorGroup(5);

    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        ...
        pipeline.addLast(longTaskGroup, new PrintHandler());

    }
}
```



### Pitfall

Netty的ChannelPipeline只有一条双向链，消息入站，经过一串InBoundHandler之后，以相反的顺序再经过OutBoundHandler出站.因此，我们自定义的handler一般会处于pipeline的**末尾**!

举个例子，当以如下顺序添加handler时，如果调用ChannelHandlerContext上的writeAndFlush方法，出站消息是无法经过StringEncoder的
```java
public class MyChannelInitializer extends ChannelInitializer<Channel> {
    private static EventExecutorGroup longTaskGroup = new DefaultEventExecutorGroup(5);

    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(64 * 1024));
        pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8))；
        pipeline.addLast(longTaskGroup, new PrintHandler());
        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
    }
}

```
这个问题有两个解决方式
1. 调整handler的顺序
2. 调用channel上的writeAndFlush方法，强制使消息在整个pipeline上流动

调整handler的顺序
```java
public class MyChannelInitializer extends ChannelInitializer<Channel> {
    private static EventExecutorGroup longTaskGroup = new DefaultEventExecutorGroup(5);

    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(64 * 1024));
        pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
        pipeline.addLast(longTaskGroup, new PrintHandler());
    }
}
```

调用Channel上的writeAndFlush方法

```java
public class PrintHandler extends SimpleChannelInboundHandler<String> {
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
//        ctx.writeAndFlush(msg);
        ctx.channel().writeAndFlush(msg);
        System.out.println(msg);
    }
}
```

### 参考
http://www.voidcn.com/article/p-yhpuvvkx-mm.html
https://stackoverflow.com/questions/37474482/dealing-with-long-time-task-such-as-sql-query-in-netty
《Netty In Action》
