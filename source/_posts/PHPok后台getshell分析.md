---
title: PHPok后台getshell分析
date: 2017-07-31 11:14:40
tags:
- 渗透测试
categories: 
- 漏洞
---


这个是夏令营师傅现场审计的一个0day，但是感觉这个cms很小众，直接利用这个漏洞的话会感觉很鸡肋，毕竟先要进后台。但是getshell的思路很nice。
<!--more-->

## 漏洞利用 ：


- 方法一：

以admin账户登入后台（admin密码怎么来，方法很多，如sql注入等），在程序升级模块压缩包上传处上传一个webshell压缩包

![](https://ws1.sinaimg.cn/large/00699yRSly1fi2whv2j9uj30sk0jqtaw.jpg)

直接访问根目录webshell文件就行了

![](https://ws1.sinaimg.cn/large/00699yRSly1fi2wi97jkbj31gw0hxwgf.jpg)

- 方法二：

条件竞争型的发升级包，和请求data/update/目录下被解包的php，这个php要能够在北执行的时候生成一个新的php文件，也就是我们的shell。



## 漏洞分析


在以管理员身份登上后台后，以一个渗透测试者的角度来看，要想拿到服务器的shell，肯定得找上传的地方，要么你在什么地方放个存储型xss，盗个管理员cookie啥的，说不定就有一些服务器信息什么的。在后台找到一个上传插件的地方，在上传的抓包分析下，可以从url看到是这个是通过url参数来调用功能的cms，上传的内容是一句话和phpinfo()

![](https://ws1.sinaimg.cn/large/00699yRSly1fi2wkl2oboj30xp0o1whf.jpg)

查看源码，从admin.php开始查看代码流程，admin.php定义了一堆宏，主要是框架在framework/admin中定义，转到该目录下，一下子就看见update_control.php。既然上传的是zip，肯定会有压缩包的解压，要想getshell，就要找到你zip解包过后文件的位置和文件名

~~~php
    //解压zip
    public function unzip_f()
    {
        $zipfile = $this->get('zipfile');
        if(!$zipfile){
            $this->error(P_Lang('未指定附件文件'));
        }
        if(strpos($zipfile,'..') !== false){
            $this->error(P_Lang('不支持带..上级路径'));
        }
        if(!file_exists($this->dir_root.$zipfile)){
            $this->error(P_Lang('ZIP文件不存在'));
        }
        $this->lib('phpzip')->unzip($this->dir_root.$zipfile,'data/update/');//把zip解压到data/update/目录下
        $info = $this->update_load();//根据包升级
        if(!$info || (is_array($info) && $info['status'] == 'error')){
            $this->error($info['content']);
        }
        $this->success();
    }
~~~

可以看到对压缩包进行解包，解包后把文件放在网站根目录的data/update/目录下，然后进行包的升级，到这里，应该可以算找到上传的路径了把，尝试一下，看看你能不能访问到，他返回的是一个cache地址，访问不到，应该是删除了。

文件升级其实跟上面zip升级差不太多
~~~php
    //文件升级
    public function file_f()
    {
        $file = $this->get('file','int');
        if(!$file){
            $this->json(P_Lang('升级失败，未指定文件'));
        }
        $urlext = 'file='.rawurlencode($file);
        $rs = $this->service(5,$urlext);
        $rs = $this->lib('json')->decode($rs);
        if($rs['status'] != 'ok'){
            $this->json($rs['content']);
        }
        if(!$rs['content']){
            $this->json(P_Lang('升级失败，升级包内容为空'));
        }
        $info = base64_decode($rs['content']);
        file_put_contents($this->dir_root.'data/tmp.zip',$info);
        $this->lib('phpzip')->unzip($this->dir_root.'data/tmp.zip','data/update/'); //把tmp.zip解压到update目录底下
        $this->lib('file')->rm($this->dir_root.'data/tmp.zip');
        $verinfo = substr($file,0,1).".".substr($file,1,1).".".substr($file,2);
        $info = $this->update_load($verinfo);
        if(!$info || (is_array($info) && $info['status'] == 'error')){
            if(!$info['content']) $info['content'] = '升级失败';
            $this->json($info['content']);
        }
        $this->json('ok',true);
    }
~~~

跟进update_load()函数里看看这个函数实现的功能

1. 把data/update目录底下的文件放入list的数组  

2. 比对升级升级信息                 
如果为delete.txt、run.php、framework/等等进行其他操作
如果不存在tmp.zip就把zip内的文件在根目录生成

3. 更新sql                  

~~~php
//升级文件
    private function update_load($verinfo='')
    {
        $list = array(); //升级文件内容列表
        $this->lib('file')->deep_ls($this->dir_root.'data/update/',$list);
        if(!$list || count($list) < 1){
            return array('status'=>'error','content'=>P_Lang('没有升级文件内容'));
        }
        $strlen = strlen($this->dir_root."data/update/");
        $delfile = false;
        $sqlfile = array();
        $cfile = array();
        foreach($list AS $key=>$value){
            $value = trim($value);
            if(!$value){
                continue;
            }
            $tmp = substr($value,$strlen);
            if($tmp == 'version.txt'){
                $verinfo = trim(file_get_contents($value));
                continue;
            }
            if($tmp == 'delete.txt'){
                $delfile = $value;
                continue;
            }
            if($tmp == 'run.php'){
                continue;
            }
            if(substr($tmp,-3) == 'sql' && $tmp != 'table.sql'){
                $sqlfile[] = $value;
                continue;
            }
            if(substr($tmp,0,17) == 'framework/config/'){
                $cfile[] = $value;
                continue;
            }
            if(substr($tmp,0,10) == 'framework/'){
                $tmp1 = substr($tmp,10);
                if(is_file($value)){
                    $this->lib('file')->mv($value,$this->dir_phpok.$tmp1);
                    continue;
                }
                if(is_dir($value) && !is_dir($this->dir_phpok.$tmp1)){
                    $this->lib('file')->make($this->dir_phpok.$tmp1,'folder');
                    continue;
                }
            }
            if(is_file($value) && $tmp != 'table.sql'){
                $this->lib('file')->mv($value,$this->dir_root.$tmp);
                continue;
            }
            if(is_dir($value) && !is_dir($this->dir_root.$tmp)){//如果tmp目录不存在，就在根目录下生成文件或目录
                $this->lib('file')->make($this->dir_root.$tmp,'folder');
                continue;
            }
        }
        //现在执行删除
        if($delfile){
            $dlist = file($delfile);
            if(!$dlist){
                $dlist = array();
            }
            foreach($dlist AS $key=>$value){
                if(!$value && !trim($value)){
                    continue;
                }
                $value = trim($value);
                if($value && is_file($this->dir_root.$value)){
                    $this->lib('file')->rm($this->dir_root.$value);
                    continue;
                }
                if($value && is_dir($this->dir_root.$value)){
                    $this->lib('file')->rm($this->dir_root.$value,'folder');
                    continue;
                }
            }
        }
        //执行table.sql操作
        $this->update_table();
        //执行新的扩展
        foreach($sqlfile AS $key=>$value){
            if(!$value || !is_file($value)){
                continue;
            }
            $info = trim(file_get_contents($value));
            if($this->db->prefix != 'qinggan_'){
                $info = str_replace('qinggan_',$this->db->prefix,$info);
            }
            if($info){
                $this->sql_run($info);
            }
        }
        //更新配置文件
        foreach($cfile AS $key=>$value){
            $base = basename($value);
            $this->lib('file')->mv($value,$this->dir_phpok.'config/'.$base);
        }
        //运行PHP文件，以实现高级的PHP更新操作
        if(file_exists($this->dir_root."data/update/run.php")){
            include($this->dir_root.'data/update/run.php');
        }
        $this->lib('file')->rm($this->dir_root.'data/update/');
        $list = $this->lib('file')->ls($this->dir_root.'data/update/');
        if($list && count($list)>0)
        {
            foreach($list as $key=>$value){
                $this->lib('file')->rm($value,'folder');
            }
        }
        //更新升级文件
        $this->success_version($verinfo);
        return array('status'=>'ok','content'=>P_Lang('升级成功'));
    }
~~~

利用方式一就是通过上传不存在的更新zip，来使更新的内容（webshell）被放入根目录。
