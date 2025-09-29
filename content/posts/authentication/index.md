+++
date = '2025-09-29T15:35:11+08:00'
lastmod = '2025-09-29T15:35:11+08:00'
draft = false
title = 'Authentication&Authorization(一): 基本概念'
slug = 'authentication-and-authorization-basic-concepts'
description = '个人学习Authentication&Authorization过程中的笔记，整理了部分基本概念，以及Basic Auth、Cookie Auth与Token Auth的介绍与对比'
keywords = ["Authentication", "Authorization", "Cookie", "Cookie Auth", "Token Auth", "JSON Web Token", "JWT", "Basic Auth"]
categories = ["Web安全", "学习笔记"]
tags = ["Authentication", "Authorization", "Cookie", "JSON Web Token"]
+++

## 基本概念
### Authorization 与 Authentication
Authorization（授权）和 Authentication（认证）是两个密切相关但又有明显区别的概念，它们的区别在于：
| | Authentication | Authorization |
|---|---|---|
| 定义 | 确认“你是谁” | 确认“你能做什么” |
| 作用 | 验证用户身份是否合法 | 决定已认证用户能否访问资源或执行操作 |
| 方式 | 用户名&密码、验证码、人脸识别、token等 | 权限表(ACL)、策略(Policy)等 |

### Regular Web App 与 Single Page App
Regular Web App（传统网页应用）和 Single Page App（单页应用）本质上都是 Web 应用，它们的区别在于：
| | Regular Web App | Single Page App |
|---|---|---|
| 页面加载方式 | 每次操作都会向服务器请求新的 HTML 页面 | 首次加载一个 HTML 页面，后续通过 `fetch` 等动态更新局部内容 |
| 用户体验 | 首屏快，但页面切换较慢 | 首屏较慢，但后续交互流畅 |
| 前后端分工 | 前端只负责展示，后端负责渲染 | 前端负责渲染和状态管理，后端提供 API |
| SEO 友好性 | 对搜索引擎友好 | 需要借助 SSR 或静态生成优化 |
| 适用场景 | 内容简单、交互较少的应用 | 交互复杂、用户体验要求高的应用 |

### 跨域请求
跨域请求（Cross-Origin Request）是指在一个 URL 下请求**不同源**（协议、域名、端口任一不同）的资源。由于浏览器的同源策略（Same-Origin Policy），跨域请求默认会被浏览器拦截，常见解决方案包括：
- 跨域资源共享 CORS（Cross-Origin Resource Sharing）：服务端在响应头中设置 `Access-Control-Allow-Origin` 来允许特定请求
- 代理转发：前端请求同源后端，由后端再去请求目标服务

### 跨站请求
跨站请求（Cross-Site Request）是指在一个注册域名下请求另一个注册域名的资源，注册域名指的是一个可以在域名注册商处直接注册的域名。

只要两个 URL 的注册域名相同，即使子域名不同，也认为它们是同站请求，例如对于 `http://a.example.com` 和 `http://b.example.com`：
- 跨域：因为子域名不同
- 同站：因为注册域名相同，都是 `example.com`

### Origin 与 Referer
`Origin` 和 `Referer` 都是浏览器自动添加的请求头字段，用于标识请求的来源，但它们的内容与用途有所不同：
- Origin：仅包含协议、域名和端口，通常用于跨域请求的验证、防范 CSRF 攻击
- Referer：包含完整的来源 URL，通常用于流量分析、防范 CSRF 攻击

### Cookie
Cookie 是存储在客户端（浏览器）的键值对数据，会在后续请求中自动携带到对应域名的请求中，实现在客户端与服务端之间传递状态信息。
Cookie 的典型用途包括会话管理（登录状态、购物车等）、个性化设置等。

#### Cookie 的常见属性
| 属性 | 含义 | 示例 |
|---|---|---|
| Name=Value | Cookie 的名称与对应的值 | session_id=abc123 |
| Domain | 指定作用域，包括该域与子域 | Domain=example.com |
| Path | 指定生效路径 | Path=/example |
| Expires | 绝对过期时间 | Expires = Wed, 21 Oct 2015 07:28:00 GMT |
| Max-Age | 秒数有效期，优先级高于 `Expires` | Max-Age=3600 |
| Secure | 限制仅通过 HTTPS 传输(localhost 除外) | - |
| HttpOnly | 禁止 JS 访问 Cookie，防止 XSS 攻击 | - |
| SameSite | 控制 Cookie 是否随跨站请求发送，避免 CSRF 攻击 | SameSite=Strict / Lax / None |

`SameSite` 属性的取值说明：
- Strict：完全禁止跨站请求携带 Cookie，最安全，但可能影响用户体验
- Lax：默认取值，允许部分 Get 跨站请求携带（比如通过搜索结果跳转）
- None：允许所有跨站请求携带，但必须同时设置 `Secure` 属性，否则会报错

#### 与 localStorage/sessionStorage 对比
| | Cookie | localStorage | sessionStorage |
|---|---|---|---|
| 存储大小 | KB 级别 | MB 级别 | MB 级别 |
| 生命周期 | 服务端自定义 | 永久存储，除非手动删除 | 关闭标签页删除 |
| 访问方式 | 自动随请求发送或通过 JS 访问(可设置 `HttpOnly` 禁止) | 只能通过 JS 访问 | 只能通过 JS 访问 |
| XSS 风险 | 可设置 `HttpOnly` 避免 | 应用自身防范 | 应用自身防范 |
| CSRF 风险 | 可设置 `SameSite` 降低 | 不存在 | 不存在 |
| 适用场景 | 会话管理、个性化设置 | 长期存储数据 | 临时存储数据 |

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
