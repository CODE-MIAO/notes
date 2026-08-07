# Junwen's Notes

基于 [Hugo](https://gohugo.io/) + [Hugo Book](https://github.com/alex-shpak/hugo-book) 的个人技术笔记站。

线上地址（部署后）：https://CODE-MIAO.github.io/notes/

## 本地预览

```powershell
hugo server -D
```

浏览器打开 http://localhost:1313/notes/

## 目录结构

```
content/docs/
  llm/        # 大模型
  rag/        # RAG
  agent/      # Agent
  mcp/        # MCP
  protocols/  # 协议与网络
  infra/      # 部署与基建
static/typora-user-images/  # 笔记图片
```

## 部署

推送到 `main` 后，由 GitHub Actions 自动构建并发布到 GitHub Pages。

仓库 Settings → Pages → Source 选择 **GitHub Actions**。
