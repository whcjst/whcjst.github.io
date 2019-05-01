---
title: SQL注入进阶
date: 2017-12-01 15:29:22
tags:
- SQL注入
categories: 
- 漏洞
---

在最近CTF学习过程中，特别是sql注入（mysql）的学习中，碰到过很多坎（“waf”）,CTF的waf不是特别的强大，比如说在联合注入中不会变态到把你的select给过滤掉，但是会过滤一些关键词。在报错注入中即使把你常用的报错函数给过滤掉了，也会留给你一线生机，就比如上次LCTF的entrance.php，过滤了information_shcema，columns，tables，database()等等，题目本意是让你把当前表下的某个字段最后一条内容给dump出来。具体的题目writeup可以看chamd5微信公众号的wp还有chebeta师傅的博客。
现在我把我最近学习关于sql注入的姿势分享出来，有可能不适合实战环境，大佬看了请轻喷。
<!--more-->

## 不常见函数绕过waf

有时候在CTF中，出题者的waf不会把waf写死，可以多fuzz几个函数。
比如下面的这些函数，这些函数均为MySQL中的空间数据类型的函数，要求几何字段非空。



- SQL空间计算函数


|  描述      | 函数    |
| ------------- |:-------------:| 
|   环    | geometrycollection() | 
|  多点    |  multipoint() |
|  面      |  polygon()    |
|  多面     |  multipolygon() |
|  线   |   linestring()|
|  多线  | multilinestring()|

我经常用的就是直接用这些函数爆出当前字段所在的库和表。

![](http://112.74.59.223/image/sql_injection/Image8.png)

之前拜读了安云luan师傅一篇文章，发现用法不仅仅只有爆当前而已，还可以跟普通的注入一样，换payload来爆出想要的信息，但是有限制，在Mysql5.5的版本下可以，在之前和之后的版本都不可以

![](http://112.74.59.223/image/sql_injection/Image9.png)

- Mysql低版本

NAME_CONST()

![](http://112.74.59.223/image/sql_injection/Image14.png)


- Mysql5.7新特性

Mysql在5.7版本引入几个新函数，可用作报错注入

![](http://112.74.59.223/image/sql_injection/Image10.png)

## 别名构造（过滤当前表字段数据库名关键字）

这个在上次LCTF就碰到了，当时注到库名表名和字段名了，但是用获取最后一个字段名下的值时显示该值被过滤了，然后想到就用这个方法，利用**别名构造**的方法


select * from (select 1)a,(select 2)b,(select 3)c union select * from users;
![](http://112.74.59.223/image/sql_injection/Image11.png)


select (select d.3 from (select * from (select 1)a,(select 2)b,(select 3)c union select * from users)d limit 1,1)e,(select 1)f,(select 1)g;

![](http://112.74.59.223/image/sql_injection/Image12.png)


## join报错

相信CTF赛棍们对这个报错注入应该都很熟悉了，不做累述。

![](http://112.74.59.223/image/sql_injection/Image15.png)