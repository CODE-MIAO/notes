---
title: "Node.js介绍"
weight: 1
---

**Node.js 不是一门语言，也不是 Express 那种 Web 框架。** 它是一个让 JavaScript 能在浏览器之外跑起来的**运行时**（runtime）：读文件、起 HTTP 服务、跑构建工具，都靠它。

浏览器里的 JS 由 Chrome / Edge 的引擎执行；服务器和终端里的 JS，通常就是 Node.js 在执行。语言还是 JavaScript（现在基本写 TypeScript，再编译成 JS），换的是「在哪儿跑、能调用哪些系统能力」。

## 它在 Web 里站哪

就算业务后端是 Go，你也很可能每天都在用 Node.js。`web` 里那篇 Nginx 笔记里的 Vite、`pnpm build`，本身就是 Node 程序。

```
你写的 Vue / TS
        │
   Vite / webpack / eslint     ← 这些工具跑在 Node.js 上
        │
   浏览器能执行的 JS / CSS
        │
     用户浏览器
        │
      Nginx
        ├─ /     → 前端打包后的 dist
        └─ /api  → 后端（Go，或也可以是 Node.js）
```

所以 Node.js 在站点里常见有两种身份，别混：

| 身份 | 干什么 | 你现在更可能碰到 |
| ---- | ------ | ---------------- |
| **工具链** | 安装依赖、启动 Vite、打包、跑测试 | 前端目录里的 `pnpm`、`vite` |
| **应用服务器** | 自己监听端口，处理 `/api` | Express / Fastify / Nest / Next.js |

Nginx 仍然是门口：静态文件自己给，`/api` 转给后面的进程。后面那个进程可以是 Go，也可以是 `node server.js`。

## 为什么会有 Node.js

JavaScript 诞生在浏览器里，一开始没有正式的「在服务器跑 JS」的标准方式。2009 年 Ryan Dahl 把 Chrome 的 **V8** 引擎拿到浏览器外面，接上操作系统的文件、网络、进程接口，就成了 Node.js。

可以记成：

- **Chrome**：V8 + DOM + 网页 API
- **Node.js**：V8 + 文件 / 网络 / 进程（通过底层的 **libuv**）

同一门语言，两套宿主。这也是「前端工程师能写后端」这条路能走通的原因：语法熟，换成另一套 API 即可。

## 三个零件：V8、libuv、事件循环

一次 `node app.js` 大致是：

1. **V8** 解析并执行你的 JS。
2. 遇到读文件、听端口、等定时器这类事，交给 **libuv** 去异步做，JS 线程不等死。
3. 做完后把回调丢回 **事件循环**（event loop），再由那一条 JS 线程执行。

这就是那句常被误解的话：**Node.js 的 JS 是单线程的，但 I/O 不是傻等。**

```
请求进来
    │
    ▼
 JS 线程很快做完「登记工作」     ← 不要在这里写死循环或超重计算
    │
    ▼
 libuv 去读盘 / 等数据库 / 等网卡
    │
    ▼
 完成后回调回到 JS 线程，写响应
```

适合 Node 的负载：大量等待（HTTP、数据库、文件）。不适合：在这个线程里做长时间 CPU 计算（大图处理、视频转码）——会堵住事件循环，所有请求一起卡。那种活更适合 Go 多线程、Worker Threads，或丢给别的服务。

## 最小的一个 HTTP 服务

不装框架也能起服务，标准库 `http` 就够：

```js
const http = require("http");

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain; charset=utf-8" });
  res.end("hello from node");
});

server.listen(3000, () => {
  console.log("http://127.0.0.1:3000");
});
```

保存为 `server.js`，执行 `node server.js`。浏览器访问该地址，就是 Node 在当 Web 服务器。

生产里很少直接用 `http` 模块，会换成 Express、Fastify、Nest 这类框架：路由、中间件、JSON 解析都替你收好。但原理仍是上面这个：进程监听端口，每个请求走回调。

## npm 和 package.json

装上 Node 一般会带 **npm**（Node Package Manager）。前端项目里你更常见 **pnpm** 或 **yarn**，干的是同一类事：按清单装依赖。

一个 Node 项目的身份证是根目录的 `package.json`：

```json
{
  "name": "demo",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "express": "^5.0.0"
  },
  "devDependencies": {
    "vite": "^6.0.0"
  }
}
```

| 字段 | 含义 |
| ---- | ---- |
| `scripts` | 快捷命令：`pnpm dev` 其实是跑这里的字符串 |
| `dependencies` | 上线还需要的包 |
| `devDependencies` | 只在开发/打包时需要的包（Vite、类型、测试） |

依赖装进 `node_modules/`。不要手改这个目录，也不用把整棵树提交进 Git；锁文件（`package-lock.json` / `pnpm-lock.yaml`）要提交，用来让别人装到同一组版本。

`npx` / `pnpm dlx` 是「临时跑一个包里的命令」，不一定先全局安装。比如 `npx --yes serve dist`。

## 模块：require 和 import

两套模块语法现在会同时见到：

| | CommonJS | ES Module |
| --- | --- | --- |
| 写法 | `const fs = require("fs")` | `import fs from "fs"` |
| 导出 | `module.exports = ...` | `export` / `export default` |
| 如何启用 | 默认（`.cjs` 或旧项目） | `"type": "module"` 或 `.mjs` |

新项目优先 ESM，和浏览器、Vite、TypeScript 一致。看旧教程时，看到 `require` 不要慌，那是 CommonJS。

## 和「框架」的边界

容易混的几个词：

| 名字 | 它是什么 |
| ---- | -------- |
| **JavaScript / TypeScript** | 语言 |
| **Node.js** | 运行语言的引擎 + 系统 API |
| **Express / Koa / Fastify** | 在 Node 上写 HTTP API 的库 |
| **NestJS** | 更重的 Node 后端框架（模块、依赖注入） |
| **Next.js / Nuxt** | 全栈框架：页面 + API，底层仍常跑在 Node 上 |
| **Deno / Bun** | 另外的 JS 运行时，目标类似 Node，生态和兼容性不同 |

学 Node 不是学某一个框架。先分清「谁在执行 JS」，再选框架。

## 几个必须认识的词

| 词 | 含义 |
| --- | ---- |
| runtime | 运行时：执行 JS 并提供 `fs`、`http`、`process` 这些能力 |
| event loop | 事件循环：单线程调度回调，不阻塞在 I/O 上 |
| npm registry | 包仓库，默认是 npmjs.com；pnpm 也从这类仓库拉包 |
| `node_modules` | 安装后的依赖目录 |
| LTS | 长期支持版本，生产优先用 LTS，不要追最新实验版 |
| N-API / addon | 用 C/C++ 给 Node 写原生扩展；一般业务用不到 |
| shebang `#!/usr/bin/env node` | 让 `.js` 文件能当命令执行 |

版本用 [nvm](https://github.com/nvm-sh/nvm) 或 Windows 上的 nvm-windows / fnm 切换。`node -v` 看当前运行时，`npm -v` 看包管理器，两者不是一回事。

## 现在学到什么程度就够

值得先搞懂：

- Node 是运行时，不是语言，也不是 Web 框架
- 为什么 Vite / pnpm 需要先装 Node
- 事件循环：I/O 可以并发，CPU 重活会堵
- `package.json`、锁文件、`node_modules` 各自干什么
- 能写一个 `http` 小服务，并知道框架只是在它上面加路由

可以后学：流（Stream）、集群 / 多进程、Worker Threads、自己写中间件、和 Nginx 反代 Node 的调优。

**一句话：** Node.js 是浏览器外的 JavaScript 发动机——前端工具链靠它转，后端也可以用它听端口；Nginx 仍然站在门口，决定静态走磁盘、接口转给哪一个进程。
