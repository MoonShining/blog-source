---
title: 一次完整的HTTP请求
date: 2018-02-05 09:51:06
tags: 后端
---

这里讲的请求是后端DevOps可以控制的范围内，不包括DNS解析，层层的路由等等，一切都从请求到达我们自己架设的服务器开始。

#### 1.与服务器建立连接
##### 1.1 TCP连接的建立
客户端的请求到达服务器，首先就是建立TCP连接

![](http://upload-images.jianshu.io/upload_images/4073552-575bd444f46a3a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. Client首先发送一个连接试探，ACK=0 表示确认号无效，SYN = 1 表示这是一个连接请求或连接接受报文，同时表示这个数据报不能携带数据，seq = x 表示Client自己的初始序号（seq = 0 就代表这是第0号包），这时候Client进入syn_sent状态，表示客户端等待服务器的回复

2. Server监听到连接请求报文后，如同意建立连接，则向Client发送确认。TCP报文首部中的SYN 和 ACK都置1 ，ack = x + 1表示期望收到对方下一个报文段的第一个数据字节序号是x+1，同时表明x为止的所有数据都已正确收到（ack=1其实是ack=0+1,也就是期望客户端的第1个包），seq = y 表示Server 自己的初始序号（seq=0就代表这是服务器这边发出的第0号包）。这时服务器进入syn_rcvd，表示服务器已经收到Client的连接请求，等待client的确认。

3. Client收到确认后还需再次发送确认，同时携带要发送给Server的数据。ACK 置1 表示确认号ack= y + 1 有效（代表期望收到服务器的第1个包），Client自己的序号seq= x + 1（表示这就是我的第1个包，相对于第0个包来说的），一旦收到Client的确认之后，这个TCP连接就进入Established状态，就可以发起http请求了。


**1.2 常见TCP连接限制**

1.2.1 **修改用户进程可打开文件数限制**

在Linux平台上，无论编写客户端程序还是服务端程序，在进行高并发TCP连接处理时，最高的并发数量都要受到系统对用户单一进程同时可打开文件数量的限制(这是因为系统为每个TCP连接都要创建一个socket句柄，每个socket句柄同时也是一个文件句柄)。可使用ulimit命令查看系统允许当前用户进程打开的文件数限制，windows上是256，linux是1024，这个博客的服务器是65535

1.2.2 **修改网络内核对TCP连接的有关限制**

在Linux上编写支持高并发TCP连接的客户端通讯处理程序时，有时会发现尽管已经解除了系统对用户同时打开文件数的限制，但仍会出现并发TCP连接数增加到一定数量时，再也无法成功建立新的TCP连接的现象。出现这种现在的原因有多种。
第一种原因可能是因为Linux网络内核对本地端口号范围有限制。此时，进一步分析为什么无法建立TCP连接，会发现问题出在connect()调用返回失败，查看系统错误提示消息是“Can’t assign requestedaddress”。同时，如果在此时用tcpdump工具监视网络，会发现根本没有TCP连接时客户端发SYN包的网络流量。这些情况说明问题在于本地Linux系统内核中有限制。

其实，问题的根本原因在于Linux内核的TCP/IP协议实现模块对系统中所有的客户端TCP连接对应的本地端口号的范围进行了限制(例如，内核限制本地端口号的范围为1024~32768之间)。当系统中某一时刻同时存在太多的TCP客户端连接时，由于每个TCP客户端连接都要占用一个唯一的本地端口号(此端口号在系统的本地端口号范围限制中)，如果现有的TCP客户端连接已将所有的本地端口号占满，则此时就无法为新的TCP客户端连接分配一个本地端口号了，因此系统会在这种情况下在connect()调用中返回失败，并将错误提示消息设为“Can’t assignrequested address”。

#### 2.发起HTTP请求
**2.1 请求格式**

例如这样的一个请求
![](http://upload-images.jianshu.io/upload_images/4073552-8758eeb49d3594c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Accept  就是告诉服务器端，我接受那些MIME类型

Accept-Encoding  这个看起来是接受那些压缩方式的文件

Accept-Lanague   告诉服务器能够发送哪些语言 

Connection       告诉服务器支持keep-alive特性

Cookie           每次请求时都会携带上Cookie以方便服务器端识别是否是同一个客户端

Host             用来标识请求服务器上的那个虚拟主机，比如Nginx里面可以定义很多个虚拟主机
                 那这里就是用来标识要访问那个虚拟主机。

User-Agent       用户代理，一般情况是浏览器，也有其他类型，如：wget curl 搜索引擎的蜘蛛等     

条件请求首部：
If-Modified-Since 是浏览器向服务器端询问某个资源文件如果自从什么时间修改过，那么重新发给我，这样就保证服务器端资源
             文件更新时，浏览器再次去请求，而不是使用缓存中的文件

安全请求首部：
Authorization: 客户端提供给服务器的认证信息

**2.2 keep-alive/persitent**

每次HTTP请求都重新建立TCP连接的开销是很大的，于是就出现了keep-alive这个首部，它允许在一次TCP连接中发送/接收多个HTTP报文

![](http://upload-images.jianshu.io/upload_images/4073552-57dca1ce2c43d5e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</img>

然而，keep-alive是有弊端的。在HTTP1.0中，客户端发起请求是加上keep-alive首部，服务端响应时也加上keep-alive首部，那么这个请求就被认为是keep-alive的，直到其中一方主动断开为止。如果没有正确断开，这个资源就会一直被占用了。

![](http://upload-images.jianshu.io/upload_images/4073552-35e40b39b00b6088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

哑代理问题：哑代理只是单纯的转发请求，并不能进行解析处理、维持持久连接等其他工作，而聪明的代理可以解析接收到的报文同时可以维持持久连接。

     如上图，当客户端与服务器之间存在不解析直接转发的代理时，connection：keep-alive这个首部是直接转发给服务器的，服务器接收了这个请求之后，就会向客户端发送带有connection：keep-alive的响应，同样盲代理不会解析响应，直接将全部响应转发回客户端。因为客户端收到了这个首部，就认为建立持久连接已经成功了，但是中间的”笨代理“，并不知道这些事情，笨代理只有一种行为模式：在转发请求和回送服务器响应请求之后就认为这次事务结束了，等待连接断开，而这时由于connection：keep-alive首部已经发送到服务器和客户端，双方都认为持久连接已经建立完成，这样就变成了两边认为持久连接OK而中间的哑代理等待连接断开的情况，这种情况下如果客户端再一次在这条连接上发送请求，请求就会在亚代理处停止，因为哑代理已经在等待连接关闭。这种状态会导致浏览器一直处于挂起状态，直到客户端或服务器之中一个连接超时，关闭连接为止，一段美好的牵手就这么没了（哑代理就是把内容原封不动的转发到代理）。

    为了避免这种情况，现代的代理是不会转发connection：keep-alive这个首部的。


**persistent**

HTTP/1.1的持久连接默认是开启的，只有首部中包含connection：close，才会事务结束之后关闭连接。当然服务器和客户端仍可以随时关闭持久连接。

当发送了connection：close首部之后客户端就没有办法在那条连接上发送更多的请求了。当然根据持久连接的特性，一定要传输正确的content-length。

还有根据HTTP/1.1的特性，是不应该和HTTP/1.0客户端建立持久连接的。最后，一定要做好重发的准备。

**管道化连接**

HTTP/1.1允许在持久连接上使用管道，这样就不用等待前一个请求的响应，直接在管道上发送第二个请求，在高延迟下，提高性能。

管道化连接的限制：

+ 不是持久连接就不能使用管道。
+ 必须按照同样的发送顺序回送响应，因为报文没有标签，很可能就顺序就乱咯。
+ 因为可以随时关闭持久连接，所以要随时做好重发准备
+ 不应该使用管道化发送重复发送会有副作用的请求（如post，重复提交）。

#### 3.负载均衡
接收到HTTP请求之后，就轮到负载均衡登场了，它位于网站的最前端，把短时间内较高的访问量分摊到不同机器上处理。负载均衡方案有软件、硬件两种

F5 BIG-IP是著名的硬件方案，但这里不作讨论

软件方案有LVS HAProxy Nginx等，留作以后补充
#### 4.Nginx(WEB服务器)
在典型的Rails应用部署方案中，Nginx的作用有两个

1. 处理静态文件请求
2. 转发请求给后端的Rails应用

这是一个简单的Nginx配置文件

![](http://upload-images.jianshu.io/upload_images/4073552-69807a8f9b06b68b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</img>

后端的Rails服务器通过unix socket与Nginx通信，Nginx伺服public文件夹里的静态文件给用户

#### 5.Rails(应用服务器)

[这篇文章无敌了](http://ohcoder.com/blog/2014/11/11/raptor-part-1/)，我没有更多可以写的，只能说一句我用的是Puma。因为服务器是单核的，只能用多线程Puma或者事件驱动的Thin，考虑到以后可能用上Rails 5 ActionCabel，还是直接上Puma吧。

#### 6.数据库(数据库服务器)
应用服务器想访问数据库，就需要与数据库建立连接。Rails读取database.yml中的配置，访问对应的数据库。

一个重要的配置指标**pool**: Rails中的数据库连接是完全线程安全的，所有pool的值要配置成与Puma的最大线程数相等，这样就不会出现线程等待数据库连接的情况。

#### 7.Redis、Memercache(缓存服务器)
#### 8.消息队列
#### 9.搜索
###参考文献
[一次完整的HTTP事务是怎样一个过程？](http://www.linux178.com/web/httprequest.html)

[Linux下高并发socket最大连接数所受的各种限制](http://blog.sae.sina.com.cn/archives/1988)

[Keep-Alive](https://www.maxcdn.com/one/visual-glossary/keep-alive/)

[谈谈持久连接——HTTP权威指南读书心得（五）](http://www.cnblogs.com/littlewish/archive/2013/01/17/2865218.html)

[数据库连接池的工作原理 ](http://www.uml.org.cn/sjjm/201004153.asp)
