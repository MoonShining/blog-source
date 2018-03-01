---
title: 使用influxdb构建运营数据Metrics系统
date: 2018-03-01 14:25:40
tags:
    - 后端
    - influxdb
    - golang
---
目前我们需要收集一些用户行为数据。

influxdb内置了HTTP API,免去了编写接入代码的繁琐，并带有数据查询和展示的组件，非常适合。

安装过程略去不提，参考[官方文档](https://docs.influxdata.com/influxdb/v1.4/introduction/getting_started/)

### 目录

+ Influxdb
+ Chronograf
+ 例子

## Influxdb
### 概念

一条influxdb的记录的结构是这样的

```
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]

例如

cpu,host=serverA,region=us_west value=0.64
payment,device=mobile,product=Notepad,method=credit billed=33,licenses=3i 1434067467100293230
stock,symbol=AAPL bid=127.46,ask=127.48
temperature,machine=unit42,type=assembly external=25,internal=37 1434067467000000000
```

类比于RDMBS

influxdb            | rdbms
----                | ---
unix-nano-timestamp | primary key
measurement         | table
tag                 | indexed column
fields              | column

### HTTP API
influxdb的所有操作都可以通过HTTP API来完成

[写数据](https://docs.influxdata.com/influxdb/v1.4/guides/writing_data/)
```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
```

### Authentication & Authorization

通过HTTP API可以进行DDL操作，我们需要有一些[权限机制](https://docs.influxdata.com/influxdb/v1.4/query_language/authentication_and_authorization/#authorization)来保证安全.

1. 修改配置文件**/etc/influxdb/influxdb.conf**并重启

```
[http]
  ...
  auth-enabled = true
  ...
```

2. 创建用户

创建超级用户admin

```
CREATE USER admin WITH PASSWORD '123456' WITH ALL PRIVILEGES
```

创建普通用户

```
CREATE USER reader WITH PASSWORD '123456'
GRANT READ ON "mydb" TO "reader"
```

### Rentention Policy

时序数据的量可能会非常大，需要定义保存数据的[策略](https://docs.influxdata.com/influxdb/v1.4/query_language/database_management/#retention-policy-management)

```
CREATE RETENTION POLICY "one_day_only" ON "mydb" DURATION 1d REPLICATION 1 DEFAULT
```

## Chronograf

官方有一个叫TICK的技术栈推荐，但对于我们的场景，只要结合其中的IC就可以了，即influxdb和chronograf.

[安装启动过程](https://docs.influxdata.com/influxdb/v1.2/introduction/installation/)不提

在使用它之前，需要先学习一下基本的[查询语法](https://docs.influxdata.com/influxdb/v1.4/query_language/data_exploration/)

[TO BE CONTINUE]