---
title: 关于Liunx提权的思考
date: 2017-8-10 19:57:49
tags:
- 提权
categories: 
- 渗透
---

http://www.freebuf.com/articles/system/129549.html

2个思路，一个是直接用提权的exp，一种是看看shadow文件可不可读，可读就直接爆破


提权exp整理
http://exploit.linuxnote.org/
https://github.com/SecWiki/linux-kernel-exploits

<!--more-->
### 一、提权

两个在靶机本地运行的脚步




- LinEnum



git clone https://github.com/rebootuser/LinEnum.git

直接运行sh脚本就行了， ./LinEnum.sh





- Linux Exploit Suggester


注意是perl写的，所以需要有环境

git clone https://github.com/PenturaLabs/Linux_Exploit_Suggester.git



攻击机kali




- exp-db


searchsploit 


### 二、明文root密码提权

遇到的概率很小

大多linux系统的密码都和/etc/passwd和/etc/shadow这两个配置文件息息相关。passwd里面储存了用户，shadow里面是密码的hash。出于安全考虑passwd是全用户可读，root可写的。shadow是仅root可读写的。

这里是一个典型的passwd文件

    root:x:0:0:root:/root:/bin/bashdaemon:x:1:1:daemon:
	usr/sbin:/bin/shbin:x:2:2:bin:/bin:/bin
	shsys:x:3:3:sys:/dev:/bin/shsync:x:4:65534:sync:/bin:
	bin/syncgames:x:5:60:games:/usr/games:/bin
	shman:x:6:12:man:/var/cache/man:/bin/shlp:x:7:7:lp:
	var/spool/lpd:/bin/shmail:x:8:8:mail:/var/mail:/bin
	shnews:x:9:9:news:/var/spool/news:/bin
	shuucp:x:10:10:uucp:/var/spool/uucp:/bin
	shproxy:x:13:13:proxy:/bin:/bin/sh
	www-data:x:33:33:www-data:/var/www:/bin
	shbackup:x:34:34:backup:/var/backups:/bin
	shlist:x:38:38:Mailing List Manager:/var/list:/bin/sh
	irc:x:39:39:ircd:/var/run/ircd:/bin
	shnobody:x:65534:65534:nobody:/nonexistent:/bin
	shibuuid:x:100:101::/var/lib/libuuid:/bin/
	shsyslog:x:101:103::/home/syslog:/bin
	falsesshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin

passwd由冒号分割，第一列是用户名，第二列是密码，x代表密码hash被放在shadow里面了（这样非root就看不到了）。而shadow里面最重要的就是密码的hash
