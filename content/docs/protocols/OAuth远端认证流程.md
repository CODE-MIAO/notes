---
title: "OAuth远端认证流程"
weight: 5
---

# OAuth2.0标准远端认证流程

前置：Cursor 已探测到授权端点与 Token 端点。浏览器只经手短命 `code`；`access_token` 只走 Cursor ↔ Auth 的 HTTPS 后台通道。

{{< diagram "oauth-remote-auth" 920 >}}

---

# 本地回环微型服务器（解析 code）

Cursor 在本机随机端口监听；云端把浏览器 302 回 `localhost`，桌面端从回调 URL 直接取出 `code`。

{{< diagram "oauth-loopback" 720 >}}
