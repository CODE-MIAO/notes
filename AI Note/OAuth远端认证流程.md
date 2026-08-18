# OAuth2.0标准远端认证流程

> 图示已内嵌到博客正文：`content/docs/protocols/OAuth远端认证流程.md`  
> 独立打开：https://CODE-MIAO.github.io/notes/diagrams/oauth-remote-auth.html

前置：Cursor 已通过探测拿到授权端点与 Token 端点 URL。浏览器只经手短命 `code`，`access_token` 只走 Cursor ↔ Auth 的 HTTPS 后台通道。

---

# 本地回环微型服务器（解析 code）

独立打开：https://CODE-MIAO.github.io/notes/diagrams/oauth-loopback.html

Cursor 在本机随机端口监听；云端把浏览器 302 回 `localhost`，桌面端从回调 URL 直接取出 `code`。
