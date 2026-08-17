**Nginx 是一台高性能的 Web 服务器**，生产环境里最常干两件事：把前端静态文件发给浏览器，以及把 `/api` 转给后端。它不是业务框架，也不查数据库。

你们开发时是：浏览器 → Vite `:5173` → 页面由 Vite 给，`/api` 再被 Vite 转到 Go `:8888`。上线后 Vite 一般不跑了，这个「门口」常常换成 Nginx。

## 它在架构里站哪

```
用户浏览器
    │  只访问 http://医院内网地址  （80 或 443）
    ▼
  Nginx
    ├─ /          → 前端打包后的 dist（index.html、JS、CSS）
    └─ /api       → Go 后端 :8888
```

浏览器只认一个入口。页面还是 SPA（Single Page Application）：Nginx 把 `index.html` 发出去，Vue Router 在浏览器里决定显示登录页还是总览；点按钮发 `/api/login` 时，由 Nginx 转给 Go。

## 最常用的三个角色

**1. 静态文件服务器**  
`pnpm build` 之后，前端变成一堆文件。Nginx 直接读磁盘返回，比让 Go 去夹带前端更合适。

**2. 反向代理**  
用户以为在跟 Nginx 说话，Nginx 再把请求转给后面的 Go。开发时 `vite.config.ts` 里的 `proxy` 就是这件事的缩小版：

```23:31:frontend/vite.config.ts
  server: {
    host: true,
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8888',
        changeOrigin: true,
      },
    },
  },
```

**3. 统一入口 / 隐藏端口**  
用户不用记 `:5173` 和 `:8888`，只访问 80（HTTP）或 443（HTTPS）。

另外它也常做 HTTPS 证书、gzip 压缩、限制请求大小。这些是加分项，不是入门第一课。

## 几个必须认识的词

| 词               | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| 正向代理         | 帮客户端出去上网（公司翻墙网关那种），Nginx 入门很少用       |
| 反向代理         | 帮服务器接客：外面打到 Nginx，再转到 Go                      |
| `server`         | 一个站点，通常对应一个端口、一个域名                         |
| `location`       | 按 URL 路径分流：`/` 静态，`/api` 转后端                     |
| `root` / `alias` | 静态文件在磁盘上的目录                                       |
| `proxy_pass`     | 转到后面哪台服务                                             |
| `try_files`      | SPA 刷新 `/login` 时，磁盘没有 `login.html`，就回退到 `index.html`，交给前端路由 |

最后这条很关键：你之前问过「第一次访问 `/login` 是谁返回页面」。生产环境里，这个「回退到 `index.html`」往往就是 Nginx 的 `try_files` 在做。

## 对照你们项目的一份最小配置

概念示例（不是你们仓库里现成的文件）：

```nginx
server {
    listen 80;
    server_name localhost;

    # 前端打包产物
    root /usr/share/nginx/html;
    index index.html;

    # 页面路由：刷新 /login、/overview 时仍返回 index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 转给 Go（相当于开发时的 Vite proxy）
    location /api/ {
        proxy_pass http://127.0.0.1:8888;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

读请求时可以记：

1. 看 `listen`：接哪个端口  
2. 看 `location`：这个 URL 走哪条规则  
3. 静态就 `root` + `try_files`，接口就 `proxy_pass`

## 一次请求怎么走（生产）

用户打开 `http://xxx/login`：

1. Nginx 发现没有名叫 `login` 的文件  
2. `try_files` 返回 `index.html`  
3. 浏览器加载 JS，Vue Router 渲染 `Login.vue`  
4. 点登录，前端 `POST /api/login`  
5. Nginx 把这条转到 Go `:8888`  
6. Go 查库、返回 JSON，Nginx 原样回给浏览器  

方法（GET/POST/DELETE）Nginx **默认原样转发**，不会改成另一种。

## 和开发环境的对应关系

| 开发                        | 生产（常见）                   |
| --------------------------- | ------------------------------ |
| Vite 提供页面和热更新       | Nginx 提供打包后的静态文件     |
| Vite `proxy /api` → `:8888` | Nginx `location /api` → Go     |
| 访问 `localhost:5173`       | 访问 `80/443`，用户看不到 8888 |

Go 仍然只负责 API 和业务；Nginx 不写你们的登录逻辑。

## 现在学到什么程度就够

值得先搞懂：

- 反向代理是什么  
- `location /` 和 `location /api` 为什么要分开  
- SPA 为什么必须 `try_files ... /index.html`  
- 它和 Vite proxy 是同一类事，只是用在上线后  

可以后学：负载均衡、缓存、限流、复杂 HTTPS、把 Nginx 调到极致。

**一句话：** Nginx 是站点门口的接待员——静态页面自己给，`/api` 转给 Go。你们开发用 Vite 当临时门口；正式部署时，这个门口通常换成 Nginx。