---
title: 使用vagrant统一开发环境
date: 2016-06-05 09:33:33
tags: vagrant
---

![](http://upload-images.jianshu.io/upload_images/4073552-7b078da0420f2533.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 简介
[vagrant](https://www.vagrantup.com/)类似现在很流行的docker ，相比起docker打包依赖的方式，vagrant打包的是整个虚拟机。

#### 核心原理
vagrant 会把你配置好的虚拟机打包成box， 通过一个Vagrantfile配置这个虚拟机的一些行为。 其他成员只要使用你的box，就可以获得统一的开发环境。

#### 使用
安装步骤略去不提，使用vagrant很简单

1.vagrant init 创建一个文件夹，然后cd到这个文件夹里

2.vagrant box add hashicorp/precise64 (这个命令会下载ubuntu12.04LTS，也可以从[这里](https://atlas.hashicorp.com/boxes/search)寻找可用的box)

3.编辑Vagrantfile

```
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise64"
end

注意！ 这里的box值必须与第二步add的值一致！
```

4.vagrant up 启动虚拟机

5.vagrant ssh 登录（也可以手动ssh，注意端口是2222,例如 ssh abc@192.168.1.1 -P 2222）

6.安装你需要的各种软件，对于我是 RVM, ruby, rails , mysql, redis...

7.sudo poweroff 关闭虚拟机

8.vagrant package 把虚拟机打包成box

9.all done！！！ 分发你的box吧
