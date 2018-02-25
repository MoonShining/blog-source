---
title: linux服务管理-upstart
date: 2018-02-25 09:48:30
tags:
    - linux
    - 后端
---
![](https://assets.digitalocean.com/articles/upstart_event_system/1.png)
当我们写完一个go程序时，部署只需要把二进制包拷贝到服务器上即可。

但在真正的**生产环境**中，如果程序出于某种原因崩溃了，我们会面临两个问题

1. 如何得知程序崩溃，如果靠人工的方式，无疑是有巨大时间滞后的
2. 如何方便的在崩溃后重启

在Ruby的世界中，例如unicorn这样的应用服务器，会有一个master进程，专门解决以上问题，当worker崩溃时，master就会重新fork出一个worker进程。

### upstart简介

upstart是linux主流发行版(ubuntu,RHEL...)本自带的进程管理系统


程序开发时需要注意的事项

作为程序开发人员，在编写系统服务时，需要了解 upstart 的一些特殊要求。只有符合这些要求的软件才可以被 upstart 管理。

规则一，派生次数需声明。

很多 Linux 后台服务都通过派生两次的技巧将自己变成后台服务程序。如果您编写的服务也采用了这个技术，就必须通过文档或其它的某种方式明确地让 upstart 的维护人员知道这一点，这将影响 upstart 的 expect stanza，我们在前面已经详细介绍过这个 stanza 的含义。

规则二，派生后即可用。

后台程序在完成第二次派生的时候，必须保证服务已经可用。因为 upstart 通过派生计数来决定服务是否处于就绪状态。

规则三，遵守 SIGHUP 的要求。

upstart 会给守护进程发送 SIGHUP 信号，此时，upstart 希望该守护进程做以下这些响应工作：

•完成所有必要的重新初始化工作，比如重新读取配置文件。这是因为 upstart 的命令"initctl reload"被设计为可以让服务在不重启的情况下更新配置。

•守护进程必须继续使用现有的 PID，即收到 SIGHUP 时不能调用 fork。如果服务必须在这里调用 fork，则等同于派生两次，参考上面的规则一的处理。这个规则保证了 upstart 可以继续使用 PID 管理本服务。

规则四，收到 SIGTEM 即 shutdown。

•当收到 SIGTERM 信号后，upstart 希望守护进程进程立即干净地退出，释放所有资源。如果一个进程在收到 SIGTERM 信号后不退出，upstart 将对其发送 SIGKILL 信号。

### 日常upstart命令

全部 | 简写
---- | ---
initctl start | start
initctl restart | restart
initctl reload | reload
initctl stop | stop

### 编写upstart配置文件

1. upstart脚本必须包含一个exec或者script片段，用于启动你的程序
2. pre-start script and post-stop script是一些钩子，可以在程序启动前后做一些事
3. start on and stop on定义了何时启动、停止你的程序

---

golang多用于写网络服务，以Nginx的官方upstart脚本为参考比较合适

```
# http://wiki.nginx.org/upstart
# nginx

description "nginx http daemon"
author "George Shammas <georgyo@gmail.com>"

start on (filesystem and net-device-up IFACE!=lo) 两个条件 文件系统和网络设备启动以后
stop on runlevel [!2345]

env DAEMON=/usr/sbin/nginx
env PID=/var/run/nginx.pid

expect fork
respawn
respawn limit 10 5
#oom never

pre-start script
        $DAEMON -t
        if [ $? -ne 0 ] 判断nginx -t命令的返回值是否是0(不是EXIT_SUCCESS)
                then exit $?
        fi
end script

exec $DAEMON

```

简单写一个

```
description "naive http golang server"
author "xxx"

start on (filesystem and net-device-up IFACE!=lo)
stop on runlevel [!2345]

respawn

chdir /vagrant

exec ./naivehttpserver
```

这样，就可以使用```sudo start naivehttpserver```这样的命令来管理naivehttpserver服务了。

更全的用法可以看[http://upstart.ubuntu.com/cookbook/](http://upstart.ubuntu.com/cookbook/)和[http://upstart.ubuntu.com/wiki/](http://upstart.ubuntu.com/wiki/)

### systemd
目前upstart有被systemd取代的趋势, 列一些资料

+ http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html
+ http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html
+ https://wiki.archlinux.org/index.php/systemd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

### 参考资料

[浅析 Linux 初始化 init 系统，第 2 部分](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/index.html)

[http://upstart.ubuntu.com/getting-started.html](http://upstart.ubuntu.com/getting-started.html)

[http://blog.fens.me/linux-upstart/](http://blog.fens.me/linux-upstart/)