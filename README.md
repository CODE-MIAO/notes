# Junwen's Notes

基于 [Hugo](https://gohugo.io/) + [Hugo Book](https://github.com/alex-shpak/hugo-book) 的个人技术笔记站。

线上地址：https://CODE-MIAO.github.io/notes/

## 本地预览

```powershell
hugo server -D
```

浏览器打开 http://localhost:1313/notes/

## 项目结构与文件作用

```
blog/
├── hugo.toml                 # 站点总配置（标题、主题、baseURL、Book 参数等）
├── README.md                 # 项目说明（本文件）
├── .gitignore                # Git 忽略规则（public、resources 等）
├── .gitmodules               # Git 子模块声明（hugo-book 主题）
├── .hugo_build.lock          # Hugo 本地构建锁文件（可忽略，勿手改）
│
├── .github/workflows/
│   └── hugo.yml              # GitHub Actions：构建并部署到 GitHub Pages
│
├── content/                  # 网站正文（真正会发布的内容）
│   ├── _index.md             # 首页
│   └── docs/                 # Book 主题侧栏对应的文档区
│       ├── _index.md         # docs 根目录页
│       ├── llm/              # 大模型相关笔记
│       ├── rag/              # RAG 相关笔记
│       ├── agent/            # Agent 相关笔记
│       ├── mcp/              # MCP 相关笔记
│       ├── protocols/        # 协议与网络相关笔记
│       └── infra/            # 部署与基建相关笔记
│
├── static/                   # 静态资源（原样复制到站点根路径）
│   └── typora-user-images/   # 笔记配图（线上访问 /notes/typora-user-images/...）
│
├── assets/                   # 需经 Hugo 处理的资源（SCSS/JS 等）
│   └── _custom.scss          # 自定义样式与 notes 主题配色
│
├── layouts/                  # 覆盖主题的模板（只放自定义部分）
│   ├── _markup/
│   │   └── render-image.html # 修正图片路径（适配 /notes/ 子路径）
│   └── _partials/docs/inject/
│       └── head.html         # 注入字体等 head 资源
│
├── themes/
│   └── hugo-book/            # Book 主题（git submodule，一般不要直接改）
│
├── archetypes/
│   └── default.md            # hugo new 新建文章时的默认模板
│
├── data/                     # 结构化数据目录（当前未用）
├── i18n/                     # 多语言文案目录（当前未用）
│
├── AI Note/                  # 原始笔记备份（迁移前源文件，不参与构建）
└── typora-user-images/       # 原始图片备份（与 static 下目录对应，不参与构建）
```

### 构建产物（本地生成，已加入 .gitignore）

| 路径 | 作用 |
|------|------|
| `public/` | `hugo` 构建输出，最终静态网站 |
| `resources/` | Hugo 资源缓存（编译后的 CSS 等） |

### 日常最常改的文件

| 你想做的事 | 改哪里 |
|------------|--------|
| 写/改笔记 | `content/docs/<主题>/xxx.md` |
| 加图片 | 放到 `static/typora-user-images/`，正文用 `/typora-user-images/文件名.png` |
| 改站点标题、菜单、搜索等 | `hugo.toml` |
| 改外观样式 | `assets/_custom.scss` |
| 改首页文案 | `content/_index.md` |
| 改自动部署流程 | `.github/workflows/hugo.yml` |

## 内容分区说明

| 目录 | 主题 |
|------|------|
| `content/docs/llm/` | 大模型 |
| `content/docs/rag/` | RAG |
| `content/docs/agent/` | Agent |
| `content/docs/mcp/` | MCP |
| `content/docs/protocols/` | 协议与网络 |
| `content/docs/infra/` | 部署与基建 |

## 部署

推送到 `master` / `main` 后，由 GitHub Actions 自动构建并发布到 GitHub Pages。

仓库 Settings → Pages → Source 选择 **GitHub Actions**。
