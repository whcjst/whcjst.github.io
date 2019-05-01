---
title: CTF中Openssl常用命令
date: 2017-05-10 20:07:37
tags: 
- CTF
- 密码学
categories: 
- CTF
---

openssl命令详解

http://blog.csdn.net/zzxian/article/details/7739673

### 对称加密


http://www.linuxidc.com/Linux/2016-03/129562.htm  

比如要用des的cbc模式，参数就是des-cbc，非常方便. 

openssl enc -help一下
<!--more-->
 几个最常见的参数   
[in/out]  
这两个参数指定输入文件和输出文件，加密是输入文件是明文，输出文件是密文；解密时输入文件是密文，输出文件是明文。

[pass]  
指定密码的输入方式，共有五种方式：命令行输入(stdin)、文件输入(file)、环境变量输入(var)、文件描述符输入(fd)、标准输入(stdin)。默认是标准输入，及从键盘输入。

[e/d]  
e:加密， d:解密 默认是加密

[-a/-base64]  
由于文件加密后是二进制形式，不方便查看，使用该参数可以使加密后的内容经过base64编码，使其可读；同样，解密时需要先进行base64解编码，然后进行解密操作。


[K/IV]  
默认文件的加密密钥的Key和IV值是有用户输入的密码经过转化生成的，但也可以由用户自己指定Key/IV值，此时pass参数不起作用



1. **只对文件进行base64编码**


对文件进行base64编码  
openssl enc -base64 -in plain.txt -out base64.txt  
对base64格式文件进行解密操作  
openssl enc -base64 -d -in base64.txt -out plain2.txt  
使用diff命令查看可知解码前后明文一样  
diff plain.txt plain2.txt


2. **不同形式输入密码**


命令行输入，密码123456  
openssl enc -aes-128-cbc -in plain.txt -out out.txt -pass pass:123456  
文件输入，密码123456  
echo 123456 > passwd.txt  
openssl enc -aes-128-cbc -in plain.txt -out out.txt -pass file:passwd.txt


3. **对称加密（aes加盐举例）**

openssl enc -aes-128-cbc -in plain.txt -out encrypt.txt -pass pass:123456 -P
salt=32F5C360F21FC12D
key=D7E1499A578490DF940D99CAE2E29EB1
iv =78EEB538897CAF045F807A97F3CFF498

openssl enc -aes-128-cbc -in plain.txt -out encrypt.txt -pass pass:123456 -P
salt=DAA482697BECAB46
key=9FF8A41E4AC011FA84032F14B5B88BAE
iv =202E38A43573F752CCD294EB8A0583E7

openssl enc -aes-128-cbc -in plain.txt -out encrypt.txt -pass pass:123456 -P -S 123
salt=1230000000000000
key=50E1723DC328D98F133E321FC2908B78
iv =1528E9AD498FF118AB7ECB3025AD0DC6

openssl enc -aes-128-cbc -in plain.txt -out encrypt.txt -pass pass:123456 -P -S 123  
salt=1230000000000000  
key=50E1723DC328D98F133E321FC2908B78  
iv =1528E9AD498FF118AB7ECB3025AD0DC6

可以看到，不使用-S参数，salt参数随机生成，key和iv值也不断变化，当slat值固定时，key和iv值也是固定的。



### 非对称加密
1. **利用openssl进行rsa加密解密**

http://www.cnblogs.com/aLittleBitCool/archive/2011/09/22/2185418.html
做ctf，rsa加密加密经常用到openssl，比如这次在安恒杯涉及到的共模攻击。
<!--more-->

生成密钥  
openssl genrsa -out test.key 1024

将这个文件中的公钥提取出来  
openssl rsa -in test.key -pubout -out test_pub.key

生成的公钥加密文件  
openssl rsautl -encrypt -in hello -inkey test_pub.key -pubin -out hello.en

解密文件  
openssl rsautl -decrypt -in hello.en -inkey test.key -out hello.de2



2. **openssl数字签名中的消息摘要用法**

http://www.cnblogs.com/gordon0918/p/5382541.html

rsa密钥进行签名验证操作 
openssl dgst -sign RSA.pem -sha256 -out sign.txt file.txt

rsa密钥来验证签名  
openssl dgst -prverify test.key -sha256 -signature  sign.txt message.txt

利用公钥来验证签名  
openssl dgst -verify  test_pub.key   -sha256 -signature  sign.txt message.txt
