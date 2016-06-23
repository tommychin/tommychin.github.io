---
tags:
  - security
layout: post
title: WEB网站常见漏洞及预防
category: 
---

### 注入漏洞
注入漏洞是利用某些输入或者资料输入特性以导入某些资料或者代码，造成目标系统操作崩溃的电脑漏洞。通常这些漏洞安全隐患是由不充分的输入确认及其他种种因素造成的。

根据注入的目标，常见的危害比较大的有以下几种：

#### SQL注入
SQL注入是指在用户输入的字符串之中注入SQL指令，在设计不良的程序当中忽略了检查，那么这些注入进去的指令就会被数据库服务器误认为是正常的SQL指令而运行，因此遭到破坏。

** 简单例子：**

```sql
SELECT * FROM users WHERE name = '" + userName + "' and pw = '"+ passWord +"';
```

其中 userName 和 passWord是用户输入的内容，且如果sql语句返回的行数大于0，我们认为此用户存在，登录成功。

假设我们不对用户输入做检查的话，那么用户输入以下内容

```sql
userName = "1' OR '1'='1";
passWord = "1' OR '1'='1";
```

最终生成的SQL语句为

```sql
SELECT * FROM users WHERE name = '1' OR '1'='1' and pw = '1' OR '1'='1';
```

这样恶意攻击者就能成功的登录了系统。

以上的例子只是最简单的情况，便于大家理解。现在的网站已经基本不会有这种问题了，但是如果开发人员不注意，仍有可能犯类似的错误。

**避免方法：**

对用户的输入内容做转义，尤其是单引号这种危险字符。另外建议不要自己去拼SQL中的参数，而是用框架或API中的原生处理方式，一般的框架都会考虑到了SQL注入的问题。

以JDBC为例，尽量不要自己去拼SQL的参数，而使用Preparestatement预编译占位符传参。

```sql
--正确
SELECT * FROM users WHERE name = ? and pw = ?;

--错误
SELECT * FROM users WHERE name = '" + userName + "' and pw = '"+ passWord +"';
```


以MyBatis为例，则要注意不能使用`$`传参，而是用`#`传单，`$` 意味着原样输出，相当于JDBC的自己拼接SQL参数，`#` 类似于Preparestatement占位符传参。

当然不是说MyBatis绝对不能使用`$`，原则是，对于用户输入内容作为参数时，绝对不能使用`$`传参。

```sql
--正确
SELECT * FROM users WHERE name = #{userName} and pw =#{userName};

--错误
SELECT * FROM users WHERE name = ${userName} and pw = ${userName};
```
对于非用户输入的内容，一般是程序自定义的动态内容，则可以放心使用$，例如：

```sql
--正确
SELECT * FROM ${table_name}  
```

在MyBatis中还有一点需要注意的是like的用法。因为like匹配一般是用户的输入内容，同时又需要添加百分号或下划线去模糊匹配，极易用错造成SQL注入，例如：

```sql
--正确 ORACLE语法
SELECT * FROM users WHERE name  like '%'||'#{param}'||'%';
--正确 MySQL语法
SELECT * FROM users WHERE name  like CONCAT('%',#{param},'%');
--错误语法
SELECT * FROM users WHERE name like '%${param}%';
 ```

#### XSS注入
XSS 即 Cross-site scripting，通常简称为XSS或跨站脚本或跨站脚本攻击 是一种网站应用程序的安全漏洞攻击，是代码注入的一种。

它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了HTML以及用户端脚本语言。

简单例子：

```html
<input type="text" name="username" value="<%=request.getParameter("username")%>" />
```

或者

```html
<div><%=request.getAttribute("user_input_content")%></div>
```


上面的两行代码分别展示了HTML展示用户数据的2种情况，对于这2种情况，假设用户输入的分别是：

```javascript
"/><script>alert(document.cookie)</script><input value="
```

 和
 
 ```javascript
<script>alert(document.cookie)</script>
```
 
则Javascript脚本被执行。

**主要危害：**

盗用 cookie ，获取敏感信息。
利用植入 Flash ，通过 crossdomain 权限设置进一步获取更高权限；或者利用Java等得到类似的操作。
利用 iframe、frame、XMLHttpRequest或上述Flash等方式，以（被攻击）用户的身份执行一些管理动作，或执行一些一般的如发微博、加好友、发私信等操作。
利用可被攻击的域受到其他域信任的特点，以受信任来源的身份请求一些平时不允许的操作，如进行不当的投票活动。
在访问量极大的一些页面上的XSS可以攻击一些小型网站，实现DDoS攻击的效果。
避免方法：

对用户的输入内容对危险字符做转义，尤其是 `<` 或者 `>` 这种危险字符。

对用户的输入内容全部转码。如果是在HTML显示，则全部转为HTML编码。java中可以使用ESAPI统一做转码处理。

例如 JSTL 中可以使用函数：`${fn:escapeXml(user_input_content)}` 或者使用  `<c:out value="${user_input_content}"/>` 不建议直接使用 `${user_input_content}`

```html
<!--正确-->
<div>${fn:escapeXml(user_input_content)}</div> 
<!--正确-->
<div><c:out value="${user_input_content}"/></div> 
<!--危险-->
<div>${user_input_content}</div>
```

上面的2种正确处理方式会转义危险字符 如 < 变成 &lt; > 变成&gt;等;

同时，在java中则可以直接使用ESAPI框架做处理，直接将所有的内容转换为HTML编码：

```html
<!--正确-->
<div><%=ESAPI.encoder().encodeForHTML(request.getAttribute("user_input_content"))%></div>
```


### 请求伪造漏洞
#### CSRF漏洞
CSRF（Cross-site request forgery），中文名称：跨站请求伪造，也被称为：one click attack/session riding，缩写为：CSRF/XSRF。

攻击者盗用了你的身份，以你的名义发送恶意请求。

CSRF能够做的事情包括：以你名义发送邮件，发消息，盗取你的账号，甚至于购买商品，虚拟货币转账等等。

**简单例子：**

银行网站A，它以GET请求来完成银行转账的操作，如：`http://www.mybank.com/Transfer.php?toBankId=11&money=1000`

危险网站B，它里面有一段HTML的代码如下：

`<img src="http://www.mybank.com/Transfer.php?toBankId=11&money=1000">`

首先，你登录了银行网站A，在没有退出登录银行A的情况下，然后访问危险网站B，这时你会发现你的银行账户少了1000块......（现在的网银都已经很安全了，绝大部分会在重要操作时进行二次验证,这里只是以此举个简单例子）

原因是银行网站A违反了HTTP规范，使用GET请求更新资源(但实际上即使POST请求，仍不能根本杜绝)。

在访问危险网站B的之前，你已经登录了银行网站A，而B中的<img>以GET的方式请求第三方资源（这里的第三方就是指银行网站了，原本这是一个合法的请求，但这里被不法分子利用了），

所以你的浏览器会带上你的银行网站A的Cookie发出Get请求，去获取资源`http://www.mybank.com/Transfer.php?toBankId=11&money=1000`

结果银行网站服务器收到请求后，认为这是一个更新资源操作（转账操作），所以就立刻进行转账操作。

**防御方法：**

简单方法是根据HTTP Header 里的Rerfer判断

防御CSRF方式方法很多样，但总的思想都是一致的，就是在客户端和服务器的通信增加令牌验证。主要原理如下：

服务器生成一个令牌；
客户端请求页面时，将生成的令牌下放到客户端页面；
客户端提交内容时，带上令牌；
服务器验证令牌的有效性；如果不正确不予处理；
令牌的产生方式通常有：

cookie的hash值，随机值,UUID ，验证码（太重量级，普通业务不适用）等。

 

### 其他注意事项
以下为在以前的开发过程中遇到的若干问题，在此总结一下。

#### 重复提交

对一些重要的业务要防止用户重复提交数据。

重复提交业务数据一般是由于用户误操作引起，例如：后退，前进，刷新页面，双击Button等。

防范方法：对于非Ajax请求，现在的浏览器一般能提示用户，防止重复提交数据。对于Ajax请求，注意用户的双击操作不要造成两次提交业务。

#### URL规范

根据HTTP1.1协议RFC2616规范第9章，HTTP的提交方法有多种，在这里重点介绍以下4种方法。

这里只根据HTTP协议，建议使用如下的RESTful的URL设计原则。

以用户管理的CURD为例。

|方法|URL规范|说明
| :------------ | :------------ |
|GET|/users|查看全部用户，列表页
|GET|/users/1|查看ID为1的用户，用户详情页
|GET|/users/name:tommy/age:21,30|查询名称是tommy，年龄在21-30岁之间的用户
|POST|/users/add|新增用户
|PUT|/users/1|修改ID为1的用户
|DELETE|/users/1|删除ID为1的用户



#### 密码加盐

盐（Salt），在密码学中，是指通过在密码任意固定位置插入特定的字符串，让散列后的结果和使用原始密码的散列结果不相符，这种过程称之为“加盐”。

网站数据库里面不能直接存放用户的密码的明文，大家应该都已经有共识了，目前一般的方式是对密码明文做散列值，例如：

密码 abcd1234       MD5之后   e19d5cd5af0378da05f63f891c7467af

但是，这个散列值很容易被彩虹表破解(现在在线的MD5彩虹表很多，如：http://www.md5online.org/ 在这个网站很容易就能查到 `e19d5cd5af0378da05f63f891c7467af` 对应的原始值 为  `abcd1234`)

应对方式就是对md5方法添加自己的规则或密钥，这个过程一般被称为加盐，这里的密钥一般也被叫做salt。
常见的加盐方法：(这里的salt是某个特定字符串，可根据业务自行指定)

```php
md5( $password + $salt )
md5( $password + md5($salt))
md5( md5($password) + md5($salt))
```


#### 帐号保护

在网站的验证码失效（被OCR识别，经测试，目前很多网站的验证码可以被轻松识别，有兴趣的可以研究一下开源OCR库tesseract）的情况下，

需要设计帐户的锁定保护，用于最大程度的保护用户的帐号安全。

同时也要注意限制用户的密码复杂度，例如，长度必须6位以上，必须是字母数字组合，必须大写等。

理想的方式是登录失败N次后，帐号自动锁定一段时间，这样不需人工干预，减少管理员的工作量，同时也起到了保护帐号的作用。

事实上，一个帐户自动锁定30分钟，基本上暴力破解就没戏了。