---
title: awk like a boss!
date: 2018-04-03 12:34:38
tags:
    - awk
    - linux
---

linux提供了很多文本处理的屠龙刀，awk就是其中的佼佼者，称为一门面向文本的编程语言也不为过(awk中可以定义变量、进行运算，处理分支条件等等)。

如果早点学会awk，财务系统就不会写的这么费劲了，直接导出一份csv，写写awk脚本就好。

### 执行流程

完整的awd命令，可以由BEGIN BODY END三部分组成，其中BEGIN和END只会在awk执行前后运行一次，而BODY部分会每行不断处理。

![QQ截图20180403123540.png](http://7xqlni.com1.z0.glb.clouddn.com/QQ截图20180403123540.png)


### Example

下面是几个几个awk的例子

```sh
# 只有 body 区域:
awk -F: '{print $1}' /etc/passwd

# 同时具有 begin,body 和 end 区域:
awk –F: 'BEGIN{printf "username\n-------\n"}\
{ print $1 }\
END {print "----------" }' /etc/passwd

# 只有 begin 和 body 区域:
awk –F: 'BEGIN {print "UID"} {print $3}' /etc/passwd
```

我们以`awk -F: '{print $1}' /etc/passwd`为例来看分析一下

1. -F:指定了以：作为分隔符
2. '{ print $1 }' 表示输出第一个
3. /etc/passwd是输入的文件

这个命令会输出

```sh
root
daemon
bin
sys
sync
games
man
lp
mail
news
...
```

### 指定分隔符


默认的输入分隔符是空格，可以通过-F选项来指定**输入字段分隔符**，例如

```sh
awk -F: '{print $2}' /etc/passwd

awk -F, '{print $2}' example.csv
```

也可以通过设置OFS变量来指定**输出字段分隔符**

```sh
awk -F, 'BEGIN {OFS=":"} {print $2, $3}' employee.txt
```

当一行容纳了多组数据时，awk允许你指定数据之间的分隔符，**输入记录分隔符**

```sh
awk -F, 'BEGIN { RS=":" } {print $2}' employee-one-line.txt
```

在输出时，记录直接允许指定**输出记录分隔符**

```
awk 'BEGIN {FS=",";ORS="\n---\n"} {print $2,$3}' employee.txt
```

$1、$2这样的变量代表分割后的字符串数组中的位置， $0代表整行

### Pattern Maching

当然，我们可以只处理一行中的某些部分，通过正则实现

```sh
# employee.txt
# Jason Smith IT Manager
# Jane Miller Sales Manager
# vagrant@z:~/awk$ cat employee.txt
# 101,John Doe,CEO
# 102,Jason Smith,IT Manager
# 103,Raj Reddy,Sysadmin
# 104,Anand Ram,Developer
# 105,Jane Miller,Sales Manager

awk -F, '/Manager/ {print $2, $3}' employee.txt
```

### awk变量、运算

一旦涉及变量定义这种比较复杂的功能，我就偏向于写awk脚本，然后`awk -f xx.awk somefile`这样来执行，常见的数学操作(+-*/%，++ --)都支持。

```sh
BEGIN {
        FS=",";
        total=0;
}
{
        print $2 "'s salary is: " $4;
        total+=$4;
}
END {
        print "----\nTotal Company Salary=$" total;
}
```

### awk比较

awk支持以下比较

![QQ截图20180403134602.png](http://7xqlni.com1.z0.glb.clouddn.com/QQ截图20180403134602.png)

可以理解成sql中的where条件
```sh
 awk -F, '$5<=5' items.txt
 awk -F "," '$4 < 900 || $5 <= 5' items.txt
 awk -F ':' '$3 > maxuid { maxuid = $3; maxline = $0 } END { print maxuid,maxline }' /etc/passwd
 awk -F ':' '$3 >= 100 && $NF ~ /\/bin\/sh/ ' /etc/passwd # $NF 代表最后一列， ~表示正则匹配， !~也就是不匹配咯

```

### awk控制流

awk支持if,if\else,while、do-while都常见控制流，for(;;)当然也不在话下，连break、continue、exit都有哦。

```sh
cat dowhile.awk

{
 i=2;
 total=0;
 do {
 total = total + $i;
 i++;
 }
 while(i<=NF)
print "Item",$1,":",total,"quantities sold";
}
```

```sh
echo "1 2 3 4" | awk '{ for (i=1;i<=NF;i++) total = total + $i } END { print total }'
```

### awk关联数组
没错，这个就是lua中的关联数组的概念

```sh
$ cat array-assign.awk
BEGIN {
 item[101]="HD Camcorder";
 item[102]="Refrigerator";
 item[103]="MP3 Player";
 item[104]="Tennis Racket";
 item[105]="Laser Printer";
 item[1001]="Tennis Ball";
 item[55]="Laptop";
 item["na"]="Not Available";
 print item["101"];
 print item[102];
 print item["103"];
 print item[104];
 print item["105"];
 print item[1001];
 print item["na"];
}
$ awk -f array-assign.awk
HD Camcorder
Refrigerator
MP3 Player
Tennis Racket
Laser Printer
Tennis Ball
Not Available
```

数组遍历

```sh
$ cat array-for-loop.awk
BEGIN {
 item[101]="HD Camcorder";
 item[102]="Refrigerator";
 item[103]="MP3 Player";
 item[104]="Tennis Racket";
 item[105]="Laser Printer";
 item[1001]="Tennis Ball";
 item[55]="Laptop";
 item["no"]="Not Available";
 for(x in item)
 print item[x]
}
$ awk -f array-for-loop.awk
Not Available
Laptop
HD Camcorder
Refrigerator
MP3 Player
Tennis Racket
Laser Printer
Tennis Ball
```

数组删除

```sh
$ cat array-delete.awk
BEGIN {
 item[101]="HD Camcorder";
 item[102]="Refrigerator";
 item[103]="MP3 Player";
 item[104]="Tennis Racket";
 item[105]="Laser Printer";
 item[1001]="Tennis Ball";
 item[55]="Laptop";
 item["no"]="Not Available";
 delete item[102]
 item[103]=""
 delete item[104]
 delete item[1001]
 delete item["na"]
 for(x in item)
 print "Index",x,"contains",item[x]
}
$ awk -f array-delete.awk
Index no contains Not Available
Index 55 contains Laptop
Index 101 contains HD Camcorder
Index 103 contains
Index 105 contains Laser Printer
```

awk支持的功能还有很多，这里不一一介绍，总之，凡是文本数据处理，用awk就没错了！

### 参考

[https://www.thegeekstuff.com/sed-awk-101-hacks-ebook/](https://www.thegeekstuff.com/sed-awk-101-hacks-ebook/)