+++
date = '2025-09-30T21:39:21+08:00'
lastmod = '2025-09-30T21:39:21+08:00'
draft = false
title = 'Web Security学习笔记(一): 基本概念'
slug = 'web-security-basics'
description = '个人学习Web Security过程中的笔记，介绍一些基础概念，包括认证与授权、跨域与跨站、Cookie与CSP等知识'
keywords = ['Web Security', '认证与授权', 'Authentication', 'Authorization', '跨域请求', '跨站请求', 'Cookie', 'CSP', 'localStorage']
categories = ['Web Security', '学习笔记']
tags = ['Security', 'Authentication', 'Authorization', 'Cookie']
+++

## Authorization 与 Authentication
Authorization（授权）和 Authentication（认证）是两个密切相关但又有明显区别的概念，它们的区别在于：
| | Authentication | Authorization |
|---|---|---|
| 定义 | 确认“你是谁” | 确认“你能做什么” |
| 作用 | 验证用户身份是否合法 | 决定已认证用户能否访问资源或执行操作 |
| 方式 | 用户名&密码、验证码、人脸识别、token等 | 权限表(ACL)、策略(Policy)等 |

## Regular Web App 与 Single Page App
Regular Web App（传统网页应用）和 Single Page App（单页应用）本质上都是 Web 应用，它们的区别在于：
| | Regular Web App | Single Page App |
|---|---|---|
| 页面加载方式 | 每次操作都会向服务器请求新的 HTML 页面 | 首次加载一个 HTML 页面，后续通过 `fetch` 等动态更新局部内容 |
| 用户体验 | 首屏快，但页面切换较慢 | 首屏较慢，但后续交互流畅 |
| 前后端分工 | 前端只负责展示，后端负责渲染 | 前端负责渲染和状态管理，后端提供 API |
| SEO 友好性 | 对搜索引擎友好 | 需要借助 SSR 或静态生成优化 |
| 适用场景 | 内容简单、交互较少的应用 | 交互复杂、用户体验要求高的应用 |

## 跨域请求
跨域请求（Cross-Origin Request）是指在一个 URL 下请求**不同源**（协议、域名、端口任一不同）的资源。由于浏览器的同源策略（Same-Origin Policy），跨域请求默认会被浏览器拦截，常见解决方案包括：
- 跨域资源共享 CORS（Cross-Origin Resource Sharing）：服务端在响应头中设置 `Access-Control-Allow-Origin` 来允许特定请求
- 代理转发：前端请求同源后端，由后端再去请求目标服务

## 跨站请求
跨站请求（Cross-Site Request）是指在一个注册域名下请求另一个注册域名的资源，注册域名指的是一个可以在域名注册商处直接注册的域名。

只要两个 URL 的注册域名相同，即使子域名不同，也认为它们是同站请求，例如对于 `http://a.example.com` 和 `http://b.example.com`：
- 跨域：因为子域名不同
- 同站：因为注册域名相同，都是 `example.com`

## Origin 与 Referer
`Origin` 和 `Referer` 都是浏览器自动添加的请求头字段，用于标识请求的来源，但它们的内容与用途有所不同：
- Origin：仅包含协议、域名和端口，通常用于跨域请求的验证、防范 CSRF 攻击
- Referer：包含完整的来源 URL，通常用于流量分析、防范 CSRF 攻击

## Cookie
Cookie 是存储在客户端（浏览器）的键值对数据，会在后续请求中自动携带到对应域名的请求中，实现在客户端与服务端之间传递状态信息。
Cookie 的典型用途包括会话管理（登录状态、购物车等）、个性化设置等。

### Cookie 的常见属性
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

### 与 localStorage/sessionStorage 对比
| | Cookie | localStorage | sessionStorage |
|---|---|---|---|
| 存储大小 | KB 级别 | MB 级别 | MB 级别 |
| 生命周期 | 服务端自定义 | 永久存储，除非手动删除 | 关闭标签页删除 |
| 访问方式 | 自动随请求发送或通过 JS 访问(可设置 `HttpOnly` 禁止) | 只能通过 JS 访问 | 只能通过 JS 访问 |
| XSS 风险 | 可设置 `HttpOnly` 避免 | 应用自身防范 | 应用自身防范 |
| CSRF 风险 | 可设置 `SameSite` 降低 | 不存在 | 不存在 |
| 适用场景 | 会话管理、个性化设置 | 长期存储数据 | 临时存储数据 |

## Content Security Policy(CSP)
CSP（内容安全策略）是一种浏览器安全机制，它的核心思想是将内容来源白名单化，服务端通过设置 `Content-Security-Policy` 响应头告知浏览器：页面中哪些外部资源可以被加载、哪些脚本可以被执行，从而降低 XSS 攻击的成功率。

CSP 的常见功能包括：
- 限制脚本来源：比如只允许执行同域名下或某个可信 CDN 的脚本
- 禁止内联脚本：默认禁止 `<script>alert()</script>` 这类内联脚本执行
- 限制图片或 CSS 来源：避免加载恶意的 CSS 与图片
- 禁止 HTTPS 的页面加载 HTTP 协议下的资源