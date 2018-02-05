---
title: 使用ELK构建分布式日志分析系统
date: 2018-02-05 09:51:39
tags: 
    - ELK
    - 后端
---

[![](https://badge.juejin.im/entry/596f38526fb9a06bbb32e759/likes.svg?style=plastic)](https://juejin.im/entry/596f38526fb9a06bbb32e759/detail)

分布式系统的日志散落在各个服务器上，对于监控和排错非常不利，我们基于ELK构建了整套日志收集，分析，展示系统。

#### 架构图
![](http://upload-images.jianshu.io/upload_images/4073552-2ea1c542bb9d7c12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 主要思路

#### 1.整理Rails日志
我们最关心的是Rails的访问日志，但是Rails日志本身的格式是有问题的，举个例子

```
Started GET "/" for 10.1.1.11 at 2017-07-19 17:21:43 +0800
Cannot render console from 10.1.1.11! Allowed networks: 127.0.0.1, ::1, 127.0.0.0/127.255.255.255
Processing by Rails::WelcomeController#index as HTML
  Rendering /home/vagrant/.rvm/gems/ruby-2.4.0@community-2.4/gems/railties-5.1.2/lib/rails/templates/rails/welcome/index.html.erb
  Rendered /home/vagrant/.rvm/gems/ruby-2.4.0@community-2.4/gems/railties-5.1.2/lib/rails/templates/rails/welcome/index.html.erb (2.5ms) Completed 200 OK in 184ms (Views: 10.9ms)
```

可以看到，一次请求的日志散落在多行中，而且在并发情况下，不同请求的日志会交织在一起，针对这个问题，我们使用[logstasher](https://github.com/shadabahmed/logstasher)重新生成一份JSON格式的日志

```
{"identifier":"/home/vagrant/.rvm/gems/ruby-2.4.0@community-2.4/gems/railties-5.1.2/lib/rails/templates/rails/welcome/index.html.erb","layout":null,"name":"render_template.action_view","transaction_id":"35c707dd9d4cd1a79f37","duration":2.34,"request_id":"bc291df8-8681-47d3-8e10-bd5d93a021a0","source":"unknown","tags":[],"@timestamp":"2017-07-19T09:29:05.969Z","@version":"1"}
{"method":"GET","path":"/","format":"html","controller":"rails/welcome","action":"index","status":200,"duration":146.71,"view":5.5,"ip":"10.1.1.11","route":"rails/welcome#index","request_id":"bc291df8-8681-47d3-8e10-bd5d93a021a0","source":"unknown","tags":["request"],"@timestamp":"2017-07-19T09:29:05.970Z","@version":"1"}
```

#### 2.使用Logstash收集日志
Logstash通过一份配置文件描述了数据从哪里来，经过怎样的处理流程，输出到何处这整套流程，分别对应于input,filter,output三个概念。

我们先使用简单的配置来验证一下正确性
```
input {
  file {
    path => "/home/vagrant/blog/log/logstash_development.log"
      start_position => beginning
      ignore_older => 0
    }
}
output {
        stdout {}
}
```
在这份配置中，我们从上一步生成的日志文件中读取，并输出到stdout中，结果如下

```
2017-07-19T09:59:01.520Z precise64 {"method":"GET","path":"/","format":"html","controller":"rails/welcome","action":"index","status":200,"duration":4.85,"view":3.28,"ip":"10.1.1.11","route":"rails/welcome#index","request_id":"27b8e5a5-dd1d-4957-9c91-435347d50888","source":"unknown","tags":["request"],"@timestamp":"2017-07-19T09:59:01.030Z","@version":"1"}
```
然后，修改Logstash的配置文件，将output改为Elasticsearch

```
input {
  file {
    path => "/vagrant/blog/log/logstash_development.log"
      start_position => beginning
      ignore_older => 0
    }
}

output {
  elasticsearch {
    hosts => [ "localhost:9200" ]
    user => 'xxx'
    password => 'xxx'
  }
}
```
可以看到，整个配置文件的可读性是非常高的，input中描述了输入源是我们整理好的日志文件，输出到Elasticsearch中。

然后就可以使用Kibanana来进行日志分析的工作了。

#### 3. Kibana的一些实践

基于Kibana，我们可以定制Elasticsearch的搜索，来查询一些非常有价值的数据

+ 查询某个接口的请求情况
+ 查询耗时在500ms以上的超慢接口
+ 查询线上报500的接口
+ 统计高频接口
......

#### 4.Future

有了ELK提供的数据，我们已经可以比较方便的完成分布式情况下的错误排查，高频接口统计，为下一步的优化提供了指导。我们不必再根据业务逻辑去猜测哪些才是20%的热点，而是有了实实在在的数据支撑。

#### 5. 问题
当然，在使用过程中也遇到过一些问题。在活动期间，访问量暴增的情况下，Elasticsearch吃了很多内存，直接拖垮了两台机器。我们通过临时关闭几台web server上的logstash暂时解决了这个问题。后续还需要对JVM进行一些调优。
