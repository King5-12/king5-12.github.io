---
title: 概念理解
---

### BaaS

> Backend as a Service, 后端即服务。

### SaaS

> Software as a Service, 软件即服务。

### FaaS

> Function as a Service, 函数即服务。

### BFF

> Back-end For Front-end，服务于前端的服务层。微服务之间的通信都是采用 RPC 接口，但是前端不能直接调用 RPC 接口，所以需要加一层服务对前端提供 http 接口，并且由于前端对于数据的需求，这一层也做数据的组装。

### CSRF

> Cross-site request forgery,跨站请求伪造：攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的注册凭证，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。

一个典型的 CSRF 攻击流程：

- 受害者登录 a.com，并保留了登录凭证（Cookie）。
- 攻击者引诱受害者访问了 b.com。
- b.com 向 a.com 发送了一个请求：a.com/act=xx。浏览器会默认携带 a.com 的 Cookie。
- a.com 接收到请求后，对请求进行验证，并确认是受害者的凭证，误以为是受害者自己发送的请求。
- a.com 以受害者的名义执行了 act=xx。
- 攻击完成，攻击者在受害者不知情的情况下，冒充受害者，让 a.com 执行了自己定义的操作。
