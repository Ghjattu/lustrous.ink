+++
date = '2025-09-29T15:35:11+08:00'
lastmod = '2025-09-30T23:45:11+08:00'
draft = false
title = 'Web Security学习笔记(三): Authentication'
slug = 'web-security-authentication'
description = '个人学习Web Security过程中的笔记，介绍Web Authentication的常见方式，包括Basic Auth、Cookie-Based、Token-Based以及JWT的原理与优缺点'
keywords = ['Web Security', 'Authentication', 'Basic Authentication', 'Cookie Authentication', 'Token Authentication', 'JWT', 'JSON Web Token']
categories = ['Web Security', '学习笔记']
tags = ['Security', 'Authentication', 'Cookie', 'JSON Web Token']
+++

## Basic Authentication
Basic Authentication 是 HTTP 协议中定义的一种最简单的认证方式。
### 认证流程
它的认证流程如下：
1. 客户端发送请求至服务端
2. 服务端检查请求头中 `Authorization` 字段的值，由于第一次请求时为空，故服务端返回 401 状态码，并在响应头中设置 `WWW-Authenticate: Basic`
3. 客户端识别出 Basic Auth，弹出一个引导输入用户名与密码的弹窗
4. 用户输入并提交后，客户端将 `username:password` 进行 Base64 编码，并添加 `Basic` 前缀后作为请求头中 `Authorization` 字段的值，重新发送请求
5. 服务端解码用户名与密码，验证通过后返回资源

### 优缺点
优点：
- 实现非常简单
- 跨平台兼容性好，遵循 HTTP 协议的客户端都可以使用

缺点：
- 弹窗样式不支持自定义
- Base64 编码不是加密算法，因此 Basic Auth 并不安全，建议配合 HTTPS 协议进行加密传输
- 不支持多因素认证等高级特性

所以，Basic Authentication 仅适用于测试环境、低安全需求的简单系统。

## Cookie Based Authentication
Cookie Based Authentication 通常指 Cookie 与 Session 结合使用的认证方式：用户登录成功后，服务端会在内存或数据库中创建一个 session 保存用户的信息，并将 session_id 通过 `Set-Cookie` 响应头返回给客户端，后续客户端的请求会自动携带该 Cookie，服务端通过解析 Cookie 中的 session_id 来识别用户身份。

这种方式在 Regular Web App 中应用较广泛，因为一般前后端同源，可以为 Cookie 设置合适的 `SameSite` 属性来降低 CSRF 攻击的风险，同时又不会影响 Cookie 的自动携带。

### 优缺点
优点：
- 用户身份信息存储在服务端，更安全
- 便于服务端执行复杂的登录状态管理

缺点：
- 增加了服务端的存储压力
- 在分布式系统中需要处理 session 共享的问题
- 不适用于移动端应用与桌面应用：这些应用没有自动管理 Cookie 的机制，需要开发者自己实现 Cookie 的存储与请求携带，增加了复杂性
- 在前后端不同源的应用中配置较复杂：因为客户端默认不会跨域携带 Cookie，必须通过设置 `Access-Control-Allow-Credentials` 和 `withCredentials` 来实现，增加了配置复杂性；如果设置 `SameSite=None`，又会增加 CSRF 攻击的风险，需要额外的防护措施

## Token Authentication
Token Authentication 是一种基于令牌(Token)的认证方式：用户登录成功后，服务端会签发一个 Token 返回给客户端，后续客户端在请求中携带该 Token（通常放在请求头的 `Authorization` 字段中，并加上 `Bearer` 前缀），服务端验证 Token 来识别用户身份。

### Token 的存储位置
这里指的是 Token 在**浏览器端**的存储位置，常见的有：
1. 内存：
   - 特点：存储在应用运行时的内存变量中（如 React 的 `state` ），安全性最高
   - 缺点：刷新或关闭页面后丢失，且无法在不同标签页中共享
2. Cookie：
   - 特点：在特定请求中自动携带，并可以设置 `HttpOnly` 属性防止 XSS 攻击
   - 缺点：容易受到 CSRF 攻击，需要结合 `SameSite` 属性或其他机制防范；另外可能需要处理跨域携带的问题
3. localStorage：
   - 特点：存储容量大、存储时间长；JS 可读写，且跨标签页共享；没有 CSRF 风险
   - 缺点：存在 XSS 风险
4. sessionStorage：
	- 特点：存储容量大、仅在当前标签页有效；JS 可读写；没有 CSRF 风险
	- 缺点：存在 XSS 风险；关闭页面后丢失

## JSON Web Token
JSON Web Token(JWT) 是一种轻量级的认证标准，该标准允许前后端使用一个 JSON 形式的 Token 传递信息，JWT 包含了一些声明和验证所需要的信息，通常在用户身份验证通过后签发。

### JWT 的结构
JWT 由三部分组成，分别是 Header、Payload 和 Signature，中间用 `.` 分隔：
- Header：描述 JWT 的元数据，包括：
  - `typ`：Token 类型，一般为定值 `JWT`
  - `alg`：数字签名的算法，包括 HS256(HMAC with SHA-256, 对称加密)和 RS256(RSA with SHA-256, 非对称加密)等
- Payload：存放实际传递的信息，称为声明(Claims)，包括：
  - 预定义的声明(Registered Claims)：如 `iss`(签发者)、`sub`(主题)、`exp`(绝对过期时间)、`iat`(签发时间)、`jti`(JWT Id)等
  - 自定义的声明(Custom Claims)：根据业务需求自定义的字段，如 `user_id` 等
- Signature：数字签名，用于验证 JWT 的完整性与真实性，生成方式如下：
  - 将 Header 与 Payload 分别进行 Base64URL 编码，并用 `.` 连接
  - 使用 `alg` 指定的算法对连接后的字符串进行签名，生成 Signature

最后，将上述三个部分经过 Base64URL 编码后，用 `.` 连接，形成最终的 JWT 字符串。

### Signature 的生成与验证流程
Signature 的生成与验证流程会因为签名算法的不同而有所区别，下面简单介绍 RS256 和 HS256 两种算法的流程：

#### RS256
- 生成流程：
  1. 首先对 Base64URL(Header) + `.` + Base64URL(Payload) 执行 SHA-256 生成一个哈希值
  2. 接着对该哈希值使用私钥进行 RSA 签名，生成 Signature
- 验证流程：
  1. 与生成流程的第一步类似，对收到的 JWT 的前两部分执行 SHA-256 生成哈希值 A
  2. 从 JWKS(JSON Web Key Set) 或其他可信来源获取公钥
  3. 对 Signature 经过 Base64URL 解码后，再使用公钥解密，得到哈希值 B
  4. 比较哈希值 A 和 B 是否相等

#### HS256
- 生成流程：
  1. 对 Base64URL(Header) + `.` + Base64URL(Payload) 使用对称密钥进行 HMAC-SHA256 签名，生成 Signature
- 验证流程：
  1. 若 JWT 的签发节点与验证节点不同，那么首先需要通过安全的渠道获取对称密钥
  2. 对收到的 JWT 的前两部分使用对称密钥生成哈希值 A
  3. 对 Signature 经过 Base64URL 解码后，得到哈希值 B
  4. 比较哈希值 A 和 B 是否相等

在分布式架构中（Token 签发和验证不是同一个节点），更推荐使用 RS256 算法，因为非对称加密的特性使得公钥可以公开分发给多个验证节点（当然是在私钥不泄漏的前提下）；而在单体架构中，HS256 对称加密的效率更高，也可以作为一种选择。

<!-- ### JSON Web Key Set -->

<!-- ### 加密的 JWT -->

### JWT 优缺点
优点：
- 适用于分布式系统
- Token 自身包含了验证所需要的全部信息，以及其他应用需要的数据

缺点：
- Token 在过期之前始终有效，除非额外实现黑名单
- Payload 过大时会增加带宽消耗
- 服务端不存储状态，因此难以主动向在线用户推送消息

### JWT 最佳实践
- 始终通过 HTTPS 传输，且避免在 Payload 中存储敏感信息
- 设置较短的 Token 有效期，并配合 Refresh Token 使用
- 验证 Token 时不仅要校验 Signature，还要检查 Payload 中 `exp` 等声明是否合法
- 利用 `jti` 声明在服务端实现黑名单，及时吊销存在风险的 Token
