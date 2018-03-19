---
title: golang热更新的魔法
date: 2018-03-16 10:21:03
tags: golang
---

当我们写一个服务端程序的时候，在更新时可能不可避免的需要停止程序再重启，这里介绍一种非常酷的热更新实现，真正做到zero downtime。

### 思路
0. 更换硬盘上的可执行程序
1. 以相同的参数启动一个子进程，并把正在listen的fd传递给子进程
2. 子进程通过这个fd进行listen，这样父子进程可以同时Accept连接
3. 立马通知父进程停止接受连接，然后父进程gracefully shutdown 

### 实现细节

POSIX提供了fork和exec调用来启动一个新进程，fork复制父进程，然后通过exec来替换自己要执行的程序。在go中，我们使用exec.Command或者os.StartProcess来达到类似效果。
在启动子进程时，需要让子进程知道，我正处于热更新过程中。通常使用环境变量或者参数来实现，例子中使用了-graceful这个参数。

```golang
file := netListener.File() // this returns a Dup()
path := "/path/to/executable"
args := []string{
    "-graceful"}

cmd := exec.Command(path, args...)
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
cmd.ExtraFiles = []*os.File{file}

err := cmd.Start()
if err != nil {
    log.Fatalf("gracefulRestart: Failed to launch, error: %v", err)
}
```

然后在子进程中使用net.FileListener来从fd创建一个Listener

```
func FileListener
func FileListener(f *os.File) (ln Listener, err error)
FileListener returns a copy of the network listener corresponding to the open file f. It is the caller's responsibility to close ln when finished. Closing ln does not affect f, and closing f does not affect ln.
```

```golang
    flag.BoolVar(&gracefulChild, "graceful", false, "listen on fd open 3 (internal use only)")

    if gracefulChild {
        log.Print("main: Listening to existing file descriptor 3.")
        f := os.NewFile(3, "") // 3就是我们传递的listening fd
        l, err = net.FileListener(f)
    } else {
        log.Print("main: Listening on a new file descriptor.")
        l, err = net.Listen("tcp", server.Addr)
    }
```

到这里，子进程就可以Accept并接受连接了，现在我们还需要立刻干掉父进程。使用getpid调用获取到父进程的id，然后kill它。

```golang
    parent := syscall.Getppid()
    syscall.Kill(parent, syscall.SIGTERM)
```

当然，更加完美的方式还需要父进程可以优雅退出，即不再接受新连接，并且处理完当前所有连接后再退出，如果一段时间内没能处理完，也可以选择直接退出。准备另写文章介绍这个内容。

### 参考

[http://grisha.org/blog/2014/06/03/graceful-restart-in-golang](http://grisha.org/blog/2014/06/03/graceful-restart-in-golang)