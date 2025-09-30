+++
date = '2025-09-30T23:28:12+08:00'
lastmod = '2025-09-30T23:28:12+08:00'
draft = false
title = 'Web Security学习笔记(二): 常见Web攻击方式'
slug = 'web-security-attacks'
description = '个人学习Web Security过程中的笔记，介绍常见的Web攻击方式，包括SQL注入、XSS、CSRF的攻击原理与防范措施'
keywords = ['Web Security', 'SQL 注入', 'SQL Injection', 'XSS', '跨站脚本', 'CSRF', '跨站请求伪造', 'Web 攻击防护', '网络安全']
categories = ['Web Security', '学习笔记']
tags = ['Security', 'SQL Injection', 'XSS', 'CSRF', 'Cookie']
+++

## SQL Injection(SQL 注入)
### 定义
SQL 注入是指：应用程序将不可信的用户输入直接拼接进 SQL 语句并执行，从而改变原语句的逻辑，导致敏感数据泄露、篡改或删除，甚至获取数据库底层权限。

### 示例
对于下面的服务端查询语句：
```go
query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s' AND password = '%s'", username, password)
```
如果攻击者把 `username` 填为 `admin' --`，把 `password` 填为任意字符串，那么拼出来的 SQL 会变成：
```sql
SELECT * FROM users WHERE username = 'admin' -- ' AND password = 'anything'
```
其中 `--` 使后面的密码相等条件变成注释，等同于绕过了密码校验。

### 防范措施
1. **参数化查询/预编译语句**（首选）：把用户输入作为参数传递，而不做字符串拼接，或者使用数据库驱动或 ORM 的预编译接口，例如：
```go
db.QueryRow("SELECT * FROM users WHERE username = ? AND password = ?", username, password)
```
2. **最小权限原则**：应用程序连接数据库的账号应当只授予必须的权限（例如只读账号仅授予 `SELECT` 权限），避免授予 `DROP` 等高级权限。
3. **输入校验与过滤**：对用户输入的长度、格式做基本校验（例如用户名与密码只允许特定字符集、邮箱采取正则校验等）。
4. 避免将数据库内部错误直接返回给客户端，另外日志中也要避免泄露 SQL 字符串与敏感参数。

总之，防范 SQL 注入的核心思想是：永远不信任用户输入，同时配合最小权限原则与合理的输入校验，最后不要将 SQL 语句的信息暴露出去。

## Cross Site Scripting(XSS)
### 定义
XSS 攻击指：攻击者将恶意脚本注入到网页中，当其他用户浏览该网页时，脚本在其浏览器上执行，可能导致页面被篡改、窃取 Cookie、进行钓鱼或触发其他攻击。
XSS 攻击有三种常见形态：反射型（Reflected）、存储型（Stored）、DOM 型（DOM-based）。

### 反射型 XSS
指：恶意脚本作为请求参数被服务端原封不动包含在响应中返回给浏览器，导致脚本被执行，反射型 XSS 通常是诱导用户点击恶意链接实现的。

示例：下面的服务端逻辑没有对查询参数进行任何转义或过滤，如果 `q` 包含了 `<script>...</script>`，那么点击链接后脚本会在浏览器中执行：
```go
q := r.URL.Query().Get("q")
fmt.Fprintf(w, "<p>%s</p>", q)
```

### 存储型 XSS
指：恶意脚本以评论等形式被服务端原封不动地持久化存储，那么后续所有查看该内容的用户浏览器都会触发脚本执行。

### DOM型 XSS
指：前端代码将不可信的内容（如 `location.hash`）通过 `innerHTML` 渲染或传入 `eval()`，导致脚本执行。此过程不涉及服务端，完全发生在前端的 DOM 操作上。

示例：对于下面的 JavaScript 代码：
```html
<div id="msg"></div>
<script>
	document.getElementById('msg').innerHTML = location.hash
</script>
```
点击链接 `https://example.com#<script>...</script>` 就会触发脚本执行。

DOM 型的攻击步骤与反射型很类似，但唯一的区别就是：构造的 URL 参数不需要发送到服务端，因此可以绕过服务端的防护手段。

### 防范措施
<!-- 虽然上文只使用了 `<script>` 标签举例，但攻击者还有其他方法注入恶意代码，比如 `onerror` 属性等。 -->
<!-- 针对 XSS 攻击，常见的防范手段包括： -->
- **转义用户输入**：在把不可信输入写入 HTML、URL 等目标时，首先对输入执行转义
- **前端使用安全的 API**：前端尽量使用 `textContent`、`innerText` 等安全的接口，避免 `innerHTML` 等危险的调用
- **富文本白名单**：如果应用程序允许用户提交富文本，那么建立一个白名单，只允许安全的标签和属性
- **Cookie 属性**：为 Cookie 设置 `HttpOnly` 属性，禁止 JavaScript 读取
- **设置 CSP 策略**：通过 CSP 限制可执行脚本的来源

## Cross Site Request Forgery(CSRF)
### 定义
CSRF（跨站请求伪造）攻击利用了浏览器会在同源请求中自动携带 Cookie 的特性，攻击者诱导已登陆用户在不知情的情况下发起对目标网站的请求，从而冒充用户执行敏感操作。

`GET` 请求的攻击示例：
- 假设银行网站有一个转账接口：`GET https://bank.com/transfer?to=attacker&amount=1000`
- 用户登录银行网站后，浏览器保存了登录态的 Cookie
- 攻击者只要在恶意网站上放一张图片：`<img src="https://bank.com/transfer?to=attacker&amount=1000">`
- 用户访问该恶意网站时，该请求会自动带上银行网站的 Cookie，从而触发转账

即使接口使用 `POST` 请求，攻击者也可以构造自动提交的表单提交请求：
```html
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="1000">
</form>
<script>document.forms[0].submit()</script>
```

### 防范措施
- **CSRF Token**：服务端在表单或页面中嵌入随机的 Token，比如隐藏在 `<form>` 的 `<input type='hidden'>` 里，前端提交请求时一并发送，服务端校验 Token 合法后才会处理请求
- **SameSite Cookie**：将 `SameSite` 属性设置为 `Lax` 或 `Strict`，可以拦截大多数跨站携带 Cookie 的请求
- **验证请求来源**：检查请求头中 `Origin` 或 `Referer` 字段，但是某些情况下可能被绕过，比如代理、隐私插件可能会隐藏 `Referer`
- **Double Submit Cookie**：前端将 CSRF Token 同时放在 Cookie 与请求参数中，服务端判断合法性和一致性
- **敏感操作二次确认**：对关键操作强制二次认证，比如短信验证码等

CSRF Token 方案在服务端渲染的 Web 应用中非常合适，但是在 Single-Page App(SPA) 中会更复杂，因为前端通常不直接用表单提交，而是通过 fetch api 等与服务端交互，此时常见做法有：
- 在网站首次加载时由服务端将 CSRF Token 作为页面数据（或 Cookie 中）返回，后续前端请求将 Token 放入自定义的请求头（如 `X-CSRF-Token`）进行校验