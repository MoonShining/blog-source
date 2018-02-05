---
title: MySQL查询优化
date: 2016-12-05 09:47:41
tags: 后端
---
2017.12.11更新
[The Secret Life of SQL: How to Optimize Database Performance by Bryana Knight](https://www.youtube.com/watch?v=BuDWWadCqIw)


### 查询性能低下的原因是访问了太多的数据

* 多表连接时返回了所有的列


```
select * from sakila.actor 
inner join sakila.file_actor using(actior_id)
inner join sakila.film using(film_id)
where sakila.film.title = 'AronMan'
```

<font color = red>正确的做法是这样</font>

```
select sakila.actor.* from sakila.actor 
inner join sakila.file_actor using(actior_id)
inner join sakila.film using(film_id)
where sakila.film.title = 'AronMan'
```

* 分解连接技术

```
select * from tag
join tag_post on tag_post.tag_id=tag.id
join post on tag_post.post_id=post.id
where tag.tag='mysql'
```

分解连接之后

```
select * from tag where tag='mysql'
select * from tag_post where tag_id=1234
select * from post where post.id in(123,456,789)
```

分解连接看上去比较浪费，但是有巨大优势

1. 缓存效率高
2. MyISAM引擎下，锁住表的时间短
3. 在应用程序端连接可以更方便扩展数据库，把表放在不同的数据库服务器上
4. 查询本身更高效
5. 减少多余行的访问

**什么时候使用分解连接？**

1. 可以缓存大量查询
2. 使用了多个MyISAM表
3. 数据分布在不同服务器
4. 对于大表使用in替换连接
5. 一个连接引用了同一个表多次

### 优化连接
1. 确保on或者using的列有索引
2. 确保group by 或者order by只引用一个列，这样可以使用索引

### 悲观锁
```
select chairid from seat where booked is null for update
update seat set booked='x' where chairid=1
commit
```



## 索引及查询优化
摘取部分自[mysql性能优化-慢查询分析、优化索引和配置 ](http://blog.chinaunix.net/uid-11640640-id-3426908.html)
 

### 索引的类型

Ø 普通索引：这是最基本的索引类型，没唯一性之类的限制。

Ø 唯一性索引：和普通索引基本相同，但所有的索引列值保持唯一性。

Ø 主键：主键是一种唯一索引，但必须指定为”PRIMARY KEY”。

Ø 全文索引：MYSQL从3.23.23开始支持全文索引和全文检索。在MYSQL中，全文索引的索引类型为FULLTEXT。全文索引可以在VARCHAR或者TEXT类型的列上创建。

使用多列索引 要注意最左前缀问题

有时MySQL不使用索引，即使有可用的索引。一种情形是当优化器估计到使用索引将需要MySQL访问表中的大部分行时。(在这种情况下，表扫描可能会更快些）。然而，如果此类查询使用LIMIT只搜索部分行，MySQL则使用索引，因为它可以更快地找到几行并在结果中返回。


### 合理的建立索引的建议：

(1)  越小的数据类型通常更好：越小的数据类型通常在磁盘、内存和CPU缓存中都需要更少的空间，处理起来更快。 

(2)  简单的数据类型更好：整型数据比起字符，处理开销更小，因为字符串的比较更复杂。在MySQL中，应该用内置的日期和时间数据类型，而不是用字符串来存储时间；以及用整型数据类型存储IP地址。

(3)  尽量避免NULL：应该指定列为NOT NULL，除非你想存储NULL。在MySQL中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值

### 这部分是关于索引和写SQL语句时应当注意的一些琐碎建议和注意点。

1. 当结果集只有一行数据时使用LIMIT 1

2. 避免SELECT *，始终指定你需要的列

从表中读取越多的数据，查询会变得更慢。他增加了磁盘需要操作的时间，还是在数据库服务器与WEB服务器是独立分开的情况下。你将会经历非常漫长的网络延迟，仅仅是因为数据不必要的在服务器之间传输。

3. 使用连接（JOIN）来代替子查询(Sub-Queries)。 连接（JOIN）之所以更有效率一些，是因为MySQL不需要在内存中创建临时表来完成这个逻辑上的需要两个步骤的查询工作。

4. 使用ENUM、CHAR 而不是VARCHAR，使用合理的字段属性长度

5. 尽可能的使用NOT NULL

6. 固定长度的表会更快

7. 拆分大的DELETE 或INSERT 语句

8. 查询的列越小越快

### Where条件

在查询中，WHERE条件也是一个比较重要的因素，尽量少并且是合理的where条件是很重要的，尽量在多个条件的时候，把会提取尽量少数据量的条件放在前面，减少后一个where条件的查询时间。

有些where条件会导致索引无效：

Ø where子句的查询条件里有！=，MySQL将无法使用索引。

Ø where子句使用了Mysql函数的时候，索引将无效，比如：select * from tb where left(name, 4) = ‘xxx’

Ø 使用LIKE进行搜索匹配的时候，这样索引是有效的：select * from tbl1 where name like ‘xxx%’，而like ‘%xxx%’ 时索引无效

## 技巧整理
1、应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

2、对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。

3、应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，如：

```
select id from t where num is null
```

可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：

```
select id from t where num=0
```

4、尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

```
select id from t where num=10 or num=20
```

可以这样查询：

```
select id from t where num=10
union all
select id from t where num=20
```

5、下面的查询也将导致全表扫描：(不能前置百分号)

```
select id from t where name like '%abc'
```

若要提高效率，可以考虑全文检索。

6、in 和 not in 也要慎用，否则会导致全表扫描，如：


```
select id from t where num in(1,2,3)
```

对于连续的数值，能用 between 就不要用 in 了：

```
select id from t where num between 1 and 3
```

7、如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然 而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：


```
select id from t where num=@num
```

可以改为强制查询使用索引：


```
select id from t with(index(索引名)) where num=@num
```

8、应尽量避免在 where 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。如：


```
select id from t where num/2=100
```

应改为:


```
select id from t where num=100*2
```

9、应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：


```
select id from t where substring(name,1,3)=’abc’
```

应改为:


```
select id from t where name like ‘abc%’
select id from t where createdate>=’2005-11-30′ and createdate<’2005-12-1′
```

10、不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用索引。

11、在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使 用，并且应尽可能的让字段顺序与索引顺序相一致。

12、不要写一些没有意义的查询，如需要生成一个空表结构：

```
select col1,col2 into #t from t where 1=0
```

这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：

```
create table #t(…)
```

13、很多时候用 exists 代替 in 是一个好的选择：

```
select num from a where num in(select num from b)
```

用下面的语句替换：


```
select num from a where exists(select 1 from b where num=a.num)
```

14、并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段 sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。

15、索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，若太多则应考虑一些不常使用到的列上建的索引是否有 必要。

16.应尽可能的避免更新 clustered 索引数据列，因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。

17、尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

18、尽可能的使用 varchar/nvarchar 代替 char/nchar ，因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

19、任何地方都不要使用 select * from t ，用具体的字段列表代替“*”，不要返回用不到的任何字段。

20、尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。

21、避免频繁创建和删除临时表，以减少系统表资源的消耗。

22、临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用表中的某个数据集时。但是，对于一次性事件，最好使 用导出表。

23、在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，避免造成大量 log ，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。

24、如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。

25、尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过1万行，那么就应该考虑改写。

26、使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更有效。

27、与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，尤其是在必须引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时 间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

28、在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ，在结束时设置 SET NOCOUNT OFF 。无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。

29、尽量避免向客户端返回大数据量，若数据量过大，应该考虑相应需求是否合理。

30、尽量避免大事务操作，提高系统并发能力。
