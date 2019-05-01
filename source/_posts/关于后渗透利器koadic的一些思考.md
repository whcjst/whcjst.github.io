---
title: 关于后渗透利器koadic的一些思考
date: 2017-11-23 16:21:34
tags:
- 远控
categories: 
- 渗透
---
## 文章起因

作为一个最近在学习js远控的萌新，在学习beef过程中碰到了许多问题，然后就请教在做ctf认识的小伙伴，小伙伴推荐了“大宝剑”koadic，在接触koadic之后，发现这东西真是一个对萌新特别友好的神器。本着分享即学习的原则，我就把我最近玩koadic的新的想法写成一篇文章，如果文章有任何不准确的地方，请各位大佬斧正。
<!--more-->

## Koadic的基础用法

前期准备

    # git clone https://github.com/zerosum0x0/koadic.git
    # cd koadic
    # pip install -r requirements.txt

准备完成之后，基本的操作在参考文章的第一篇已经有所介绍，我就不再累述。koadic根据作者在github上的介绍，主要分为这么2个模块  

- **Stagers**  

|  Module        | Description           |
| ------------- |:-------------:| 
| stager/js/mshta      | serves payloads in memory using MSHTA.exe HTML Applications | 
| stager/js/regsvr     | serves payloads in memory using regsvr32.exe COM+ scriptlets |
| stager/js/rundll32_js | serves payloads in memory using rundll32.exe |
| stager/js/disk  | serves payloads using files on disk |

stagers是在你攻击机上生成的payload的种类

- **Implants**


|  Module        | Description           |
| ------------- |:-------------:| 
| implant/elevate/bypassuac_eventvwr     | Uses enigma0x3's eventvwr.exe exploit to bypass UAC on Windows 7, 8, and 10. | 
| implant/elevate/bypassuac_sdclt     | Uses enigma0x3's sdclt.exe exploit to bypass UAC on Windows 10. |
| implant/fun/zombie | Maxes volume and opens The Cranberries YouTube in a hidden window. |
| implant/fun/voice  | Plays a message over text-to-speech. |
| implant/gather/clipboard  | Plays a message over text-to-speech. |
| implant/gather/hashdump_sam | Retrieves hashed passwords from the SAM hive.|
| implant/gather/hashdump_dc | Domain controller hashes from the NTDS.dit file. |
| implant/inject/mimikatz_dotnet2js | Injects a reflective-loaded DLL to run powerkatz.dll (@tirannido DotNetToJS). |
| implant/inject/shellcode_excel | Runs arbitrary shellcode payload (if Excel is installed). |
| implant/manage/enable_rdesktop | Enables remote desktop on the target. |
| implant/manage/exec_cmd | Run an arbitrary command on the target, and optionally receive the output. |
| implant/pivot/stage_wmi | Hook a zombie on another machine using WMI.|
| implant/pivot/exec_psexec | Run a command on another machine using psexec from sysinternals. |
| implant/scan/tcp | Uses HTTP to scan open TCP ports on the target zombie LAN. |
| implant/utils/download_file | Downloads a file from the target zombie. |
| implant/utils/upload_file | Uploads a file from the listening server to the target zombies.|

implant是在koadic能执行的操作，想命令执行exec_cmd，上传下载文件download_file，upload_file，抓hash  hashdump_sam和hashdump_dc等等。


但是有几点补充：
我在本机测试的时候
1.用python2运行，会出现UnicodeDecodeError：ascii的错误，第一篇文章里面有了[解决方案](http://blog.csdn.net/qq_20125305/article/details/44562901)
2.在更新过微软的补丁，bypassuac_eventvwr 和/bypassuac_sdclt win10提权均不成功，所以抓hash的一些操作不了。
3.在implant/fun 模块下 voice操作全部成功，但是cranberry出现AttributeError: 'ThunderstruckImplant' object has no attribute 'load_script'错误，跟了一遍代码，发现不了问题，而且github上面也没有issue，很苦恼，各位大佬看出问题，可以联系小弟。


## 结合office的DDE命令执行来钓鱼

实验环境：
攻击机  kali（192.168.189.137）
靶机    win7家庭版 （192.168.189.139）

参考第二篇参考文章，利用dde做一个带有恶意命令的word

![](http://oohnuejim.bkt.clouddn.com/dde.png)

诱导靶机用户点击这个word，并且允许进行外部链接引用，当用户点击点击链接，我们攻击已经生成了一个session

![](http://oohnuejim.bkt.clouddn.com/%E5%8F%8D%E5%BC%B92.png)

我们可以使用 bypassuac_evenvwr提权，提完权可以进行其他骚骚的操作
![](http://oohnuejim.bkt.clouddn.com/%E6%8F%90%E6%9D%83.png)

扫描tcp端口
![](http://oohnuejim.bkt.clouddn.com/tcp%E6%89%AB%E6%8F%8F.png)
也可以进行内网扫描（偷了个懒，扫了kaili，kali是扫不到的）
![](http://oohnuejim.bkt.clouddn.com/tcp%E6%89%AB%E6%8F%8F2.png)


也可以上传文件（这个我在win10上成功了，win7上没成功）
![](http://oohnuejim.bkt.clouddn.com/%E4%B8%8A%E4%BC%A02.png)
![](http://oohnuejim.bkt.clouddn.com/%E4%B8%8A%E4%BC%A0.png)

功能还是挺强大的



## 参考文章

http://www.freebuf.com/sectool/145674.html
http://www.freebuf.com/articles/terminal/150285.html