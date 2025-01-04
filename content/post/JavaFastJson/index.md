---
title: Java反序列化-FastJson
description: 本文章详细记录Java反序列化-FastJson的审计全过程和漏洞利用原理。
slug: Intelligent-reporting-system
date: 2025-01-04
image: cover.jpg
categories:
    - cate1
tags:
    - Java
    - CVE
---

## 前言

​	虽然审了不少的Javaweb项目，但是我还是是认为自己的Java反序列化的知识点不够深刻，所以还是决定深度学习一下反序列化的知识点，而FastJson又是比较常见的Java反序列化依赖，所以决定从FastJson着手来学习Java反序列化。

## FastJson简介

​	Fastjson 是 Alibaba 开发的 Java 语言编写的高性能 JSON 库，用于将数据在 JSON 和 Java Object 之间互相转换。

​	提供两个主要接口来分别实现序列化和反序列化操作。

```java
JSON.toJSONString //将 Java 对象转换为 json 对象，序列化的过程。
JSON.parseObject/JSON.parse //将 json 对象重新变回 Java 对象；反序列化的过程
```

​	所以可以简单的把 json 理解成是一个字符串。

​	这里的知识点和PHP的反序列化实际是相同的，不同的只是反序列化的过程中所调用的方法不同，所以可以互相类比，这里贴出我之前所写的PHP反序列化的文章链接，有兴趣的师傅可以去看一下。

[序列化与反序列化基础及反序列化漏洞(附案例)_反序列化漏洞代码-CSDN博客](https://blog.csdn.net/2301_79629995/article/details/142713714?spm=1001.2014.3001.5501)

## 简单的序列化demo

### 依赖引入

​	为了深刻的理解FastJson的漏洞，这里提供一个简单的代码demo。先创建一个springboot项目，然后再pom文件中引入依赖。

```xml
<dependency>  
 <groupId>com.alibaba</groupId>  
 <artifactId>fastjson</artifactId>  
 <version>1.2.24</version>  
</dependency>
```

​	版本其实无所谓，这只是一个简单演示，只是为了方便理解FastJson的工作原理。

### 类定义

​	简单定义一个类，注意：**其中无参构造方法不可省略，否则反序列化过程中将无法正确加载类。**

```java
package com.example.demo;

import java.io.IOException;

public class User {
    private String name;
    private int age;
    public User() {
        System.out.println("无参构造");
    }
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public String getName() throws IOException {
        System.out.println("unser!!!");
        Runtime.getRuntime().exec("calc");
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}

```

### 控制器代码

​	然后再写一个控制器，调用JSON.toJSONString去序列化对象，并添加选项SerializerFeature.WriteClassName，让他写出类名。

```java
package com.example.demo;

import com.alibaba.fastjson.serializer.SerializerFeature;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

@RestController
public class FastJsontest {
    @GetMapping("/")
    public String index(String name,int age) {
        User thisuser = new User(name,age);
        String json = JSON.toJSONString(thisuser, SerializerFeature.WriteClassName);
        return json;
    }
}

```

​	启动项目后来到前端，访问如下URL：[127.0.0.1:8080/?name=123&age=12](http://127.0.0.1:8080/?name=123&age=12)，他会返回如下的JSON字符串

```json
{"@type":"com.example.demo.User","age":12,"name":"123"}
```

​	这样一个简单demo就搭建好了。

## 序列化过程

​	我们设置断点后运行一下。跟踪后来到了这个方法。

```java
    public static String toJSONString(Object object, int defaultFeatures, SerializerFeature... features) {
        SerializeWriter out = new SerializeWriter((Writer)null, defaultFeatures, features);

        String var5;
        try {
            JSONSerializer serializer = new JSONSerializer(out);
            serializer.write(object);
            var5 = out.toString();
        } finally {
            out.close();
        }

        return var5;
    }
```

​	序列化是在serializer.write(object);这一步完成的。

```java
    public final void write(Object object) {
        if (object == null) {
            this.out.writeNull();
        } else {
            Class<?> clazz = object.getClass();
            ObjectSerializer writer = this.getObjectWriter(clazz);

            try {
                writer.write(this, object, (Object)null, (Type)null, 0);
            } catch (IOException e) {
                throw new JSONException(e.getMessage(), e);
            }
        }
    }
```

​	这一步是在获取类。**writer.write(this, object, (Object)null, (Type)null, 0);**中比较关键的参数有两个。一个是this中的typekey，这个是序列化完成后JSON字符串中类的键。一个就是object，这个是类名。之后就是将类中的参数也并入字符串中。

## 反序列化demo

### 	控制器

```java
	@GetMapping("/unser")    
	public String unser(String text) {
        JSONObject user=JSON.parseObject(text);
        System.out.println(user);
        return "success";
    }
```

​	访问：

```json
{
    "name": "123",
    "age": 123
}
```

## 反序列化过程

​	打上断点后运行。

```java
    public static Object parse(String text, int features) {
        if (text == null) {
            return null;
        } else {
            DefaultJSONParser parser = new DefaultJSONParser(text, ParserConfig.getGlobalInstance(), features);
            Object value = parser.parse();
            parser.handleResovleTask(value);
            parser.close();
            return value;
        }
    }
```

​	**Object value = parser.parse();**反序列化是在这一步完成的。

![image-20250104175346873](/image-20250104175346873.png)

​	这个方法代码较长，所以这里就只截图了，该方法首先判断lexer.token()返回的值，返回值为12，对应

![image-20250104175637464](/image-20250104175637464.png)

​	我们传入的字符串也确实'{'开头，然后12的情况会交给这个方法来处理。

![image-20250104175805801](/image-20250104175805801.png)

​	继续跟进，**JSONObject object = new JSONObject(lexer.isEnabled(Feature.OrderedField));**这一步返回了一个空的object。继续跟进**this.parseObject((Map)object, fieldName);**

![image-20250104180537401](/image-20250104180537401.png)

​	这一步又对token进行了几次判断。

![image-20250104181156564](/image-20250104181156564.png)

​	这是一个死循环，从上往下的逻辑为，跳过空格，获取当前字符，AllowArbitraryCommas这里是遇到多个逗号会跳过。

![image-20250104191049113](/image-20250104191049113.png)

​	这一步是获取键，然后再检测':'判断传入的字符串是否合法。

![image-20250104191612383](/image-20250104191612383.png)

​	这一步功能是判断键是否为@type并通过scanSymbol获取类名，再通过loadClass加载类。

![image-20250104192322790](/image-20250104192322790.png)

​	这里的逻辑是，先判断类名是否为空长度是否为0，如果都满足就执行下面的逻辑，先从mappings中加载类，显然这里无法加载成功，再通过中括号判断是否是数组，这里显然也不是，**再判断类名的首字母是否为L，尾字符是否为;，如果是就删除掉，再重新加载**，这里画个小重点。

![image-20250104193041690](/image-20250104193041690.png)

​	这里判断了classLoader是否为null，这里为null所以就执行了这里的代码，调用了单参数loadClass，实际上就是将classloader修改为了false。

![image-20250104193508416](/image-20250104193508416.png)

​	重点再这一步走到contextClassLoader处，通过Thread.currentThread().getContextClassLoader()获取AppClassLoader。接着使用AppClassLoader来加载Student类，将它放到缓存里面，然后返回clazz。

​	![image-20250104194317013](/image-20250104194317013.png)

​	从这一步开始就开始了反序列化的过程，在此之前的操作都仅仅是对JSON字符串的操作。这里的size是放入的参数，我们追溯到这里为止还没有放入任何的参数，所以这里就size自然为0。这一步的重点是通过config.getDeserializer(clazz)获取反序列化器，然后使用这个反序列化器来反序列化。跟进**getDeserializer**![image-20250104203136252](/image-20250104203136252.png)、	

​	这一步还是直接从缓存中获取User类，这里直接就获取到了，反序列化器不影响整体流程deserializer内部无法查看，不过不影响。这里由于直接从缓存中取出了类，所以这里中间一大段也就省略了。

![image-20250104203946113](/image-20250104203946113.png)

​	我们直接跟来到toJSON这个方法，这里是判断一下这个类是不是空、JSON类、Map类或者Collection，这里显然不是。

![image-20250104204101501](/image-20250104204101501.png)

​	如果都不是就判断是不是数字，是不是数组或者是不是原始类型，这里显然也不是。

![image-20250104204613253](/image-20250104204613253.png)

​	这一步先检查了一下反序列化器，不重要，重点在这一步。**javaBeanSerializer.getFieldValuesMap(javaObject);**我们继续追溯。

![image-20250104204807226](/image-20250104204807226.png)

​	可以看到这里循环遍历了对象所属类的getter。到这里我们已经成功的找到了调用了Java类中的getter方法的位置，而这里也就是FastJson产生反序列化漏洞的位置。而我们只需要简单修改一下我们之前所定义的类。

```java
package com.example.demo;

import java.io.IOException;

public class User {
    private String name;
    private int age;
    public User() {
        System.out.println("无参构造!!!");
    }
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public String getName() throws IOException {
        System.out.println("unser!!!");
        Runtime.getRuntime().exec("calc");
        return name;
    }
    public void setName(String name) {
        System.out.println("set!!!");
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

​	这里我修改部分方法的代码，其中getName方法就是一个危险方法，我们再回到前端传入相同的参数，即可弹出计算机。

![image-20250104205247702](/image-20250104205247702.png)

​	可以看到，当代码经过这里之后就会弹出计算机。除此之外set方法也有一条路径，这里由于篇幅限制就不展开了。

## 总结

​	FastJson在反序列化时，会通过键"@type"去寻找类，之后在反序列化的时候会自动的去调用getter方法和无参构造方法。

下面直接引用结论，Fastjson会对满足下列要求的setter/getter方法进行调用：

满足条件的setter：

​	**非静态函数**
​	**返回类型为void或当前类**
​	**参数个数为1个**
满足条件的getter：

​	**非静态方法**
​	**无参数**
​	**返回值类型继承自Collection或Map或AtomicBoolean或AtomicInteger或AtomicLong**

## FastJson漏洞原理及原理

### 原理

​	由前面的审计过程我们知道，Fastjson是自己实现的一套序列化和反序列化机制，不是用的Java原生的序列化和反序列化机制。无论是哪个版本，Fastjson反序列化漏洞的原理都是一样的，只不过不同版本是针对不同的黑名单或者利用不同利用链来进行绕过利用而已。

#### 如何反序列化出恶意类？

​	由代码demo我们可知，parseObject()/parse()在进行反序列化时，可通过键"@type"指定类型，若包含的类型太大，比如object或者Jsonobject就可以反序列化处任意类。例如代码写

```java
Object o = JSON.parseObject(poc,Object.class)
```

就可以反序列化出Object类或其任意子类，而Object又是任意类的父类，所以就可以反序列化出所有类。

#### 如何通过反序列化利用恶意类触发恶意函数？

​	根据审计我们可知，FastJson的反序列化漏洞触发的效果将所有满足条件的setter、getter以及构造方法全部执行一遍，如果这三种方法中存在危险操作，就会导致反序列化漏洞。也就是说，该漏洞的前提是setter、getter以及构造方法中存在存在漏洞才能触发。比如我上面所提供的类中的getName方法。

## 参考文献

[Java反序列化—Fastjson基础_java fastjson-CSDN博客](https://blog.csdn.net/qq_61237064/article/details/128579803?ops_request_misc=%7B%22request%5Fid%22%3A%2208ab84ad2013fe9bdf7d95652b88da50%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=08ab84ad2013fe9bdf7d95652b88da50&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-128579803-null-null.142^v101^pc_search_result_base6&utm_term=Java FastJSON反序列化&spm=1018.2226.3001.4187)

