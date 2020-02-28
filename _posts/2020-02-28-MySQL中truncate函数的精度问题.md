---
layout:     post                    # 使用的布局（不需要改）
title:      MySQL中truncate函数自动转型的精度问题         # 标题 
subtitle:   MySQL  #副标题
date:       2020-02-28          # 时间
author:     MistRay                      # 作者
# header-img: img/post-bg-7.jpg 这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - MySQL
---
## MySQL中truncate函数的精度问题

#### truncate函数

TRUNCATE(X,D) 是MySQL自带的一个系统函数。  
其中，X是数值，D是保留小数的位数。  
其作用就是按照小数位数，进行数值截取（此处的截取是按保留位数直接进行截取，没有四舍五入）。  

现在有这样一个需求，某个表有一个varchar类型的字段，字段内存了一串浮点数。
现在我想要截取到该字段的小数点后两位。


```sql
select truncate('5.025849038590438543',2);
```

得到的结果毫无疑问，应该是**5.02**，结果也确实为**5.02**。

那下面这条SQL的结果，你能猜到么？

```sql
select truncate('5.02',2);
```

你可能会说，这还不简单，肯定还是**5.02**啊。

可是，神奇的一幕出现了，该SQL的执行结果为**5.01**。
为什么会少了0.1呢，这时就要说到MySQL的TRUNCATE函数做了什么。
因为TRUNCATE函数会对varchar类型做自动转型，产生了精度损失。
在转型后得到的结果其实是5.019999999。然后再做截取，那么结果自然是5.01了。


下面来执行这条SQL
```sql
select truncate(5.02,2);
```
由于没有使用varchar类型，所以TRUNCATE函数没有做自动转型操作，所以并没有产生精度损失，结果为5.02.

#### cast函数
cast是一种数据类型转换的函数，函数将任何类型的值转换为具有指定类型的值，语法格式如下所示：

**CAST ( expression  AS  data_type)**  

* expression：任何有效的MySQL表达式或者一些字符串数据。  

* AS：用于分隔两个参数，在AS之前的是要处理的数据，AS之后是要转换的数据类型。  

* data_type：系统所提供的数据类型，这里不能使用用户定义的数据类型。

* MySQL所能使用的可以是以下类型之一：CHAR(字符型)、DATE(日期)、TIME(时间)、DATETIME(日期时间型)、DECIMAL(浮点数 float)、SIGNED(整数 int)。


依然用5.02这个数字举例:

```sql
select cast('5.02' as decimal(18,2));
```

看似没什么问题，结果也确实是5.02。
但如果使用的是5.029呢？

```sql
select cast('5.029' as decimal(18,2));
```
你会发现，结果变成5.03了，没错，四舍五入了。
执行出来的结果和truncate函数的结果不尽相同，所以也并不能完美满足一开始提出的需求。

#### 总结
这个问题产生的根本问题是使用了varchar类型来存浮点数，并且还需要在数据库内做运算导致的。  
我认为解决这个问题的最好时机是在设计时，使用浮点类型来存储该字段。


## 转载
本文遵循 [CC 4.0 by-sa](https://creativecommons.org/licenses/by-sa/4.0/) 版权协议,转载请附上原文出处链接和本声明。