---
title: https理论与实践
date: 2018-02-05 09:46:06
tags: 后端
---
说点废话，我一直觉得APPLE是个非常激进的公司， 从早些的Flash，到现在的http，又比如新的Mac连USB口都不给。公司大到这个量级，已经可以凭一己之力促进先进技术的普及了。

#### 本文内容分为以下三部分

+ HTTPS协议
+ 使用Let's Encrypt在后端部署https服务
+ https在iOS上的正确使用姿势

#### Part1 HTTPS协议
普通的HTTP请求，在通信双方建立了TCP连接之后，就可以进行了。而HTTPS则不同，在建立TCP连接之后，需要先进行SSL协议的握手过程，然后才是HTTP的通信。

SSL的握手过程如下图所示
![](http://upload-images.jianshu.io/upload_images/4073552-0d43756862bec209.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

alice想要与bob进行https的通信，需要以下几步

1. alice给出协议版本号、一个客户端生成的随机数（Client random），以及客户端支持的加密方法。
2. bob确认双方使用的加密方法，并给出数字证书（包含bob的公钥）、以及一个服务器生成的随机数（Server random）。
3. alice 确认数字证书（向CA确认）有效，然后生成一个新的随机数（Premaster secret），并使用数字证书中的公钥，加密这个随机数，发给鲍勃。
4. bob使用自己的私钥，获取爱丽丝发来的随机数（即Premaster secret）。
5. alice和bob根据约定的加密方法，使用前面的三个随机数，生成"对话密钥"（session key），用来加密接下来的整个对话过程。

上述步骤的最终目的，是为了生成”对话密钥“，以后的通信都使用这个密钥进行对称加密（一般对称加解密的速度是比较快的）

那么看到这里，就又一个问题出现了。握手阶段的信息安全如何保障？

答案是无法保障。整个握手阶段，都是明文的。因此如果有第三方窃听了通信，他可以获得Client random、Server random以及加密后的Premaster secret。只要第三方无法破解Premaster secret的内容，那么通信就是安全的。

#### Part2 使用Let's Encrypt在后端部署https服务
可以参考这篇文章

[How To Secure Nginx with Let's Encrypt on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)

大致步骤

1. 安装certbot客户端
2. 配置服务器允许方案 /.well-known文件夹
3. sudo letsencrypt certonly -a webroot --webroot-path=/var/www/html -d example.com -d www.example.com 这个命令会生成你需要的证书等文件
4. 配置Nginx ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

如果速度很慢多半是python的源被墙了，需要改一下pip配置。

Let's Encrypt大法好，退沃通保平安！

#### Part3 在iOS上使用自己颁发的HTTPS证书的正确姿势

在开发环境，也需要进行HTTPS的话，需要对AFN进行两个设置：允许不合法的证书和不验证域名

```
    AFHTTPSessionManager *manager = [[AFHTTPSessionManager alloc] initWithBaseURL:[NSURL URLWithString:[self host]]];

    manager.securityPolicy.allowInvalidCertificates = YES;
    manager.securityPolicy.validatesDomainName = NO;
```
