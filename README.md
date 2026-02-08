# cloudnative-cookbook

云原生技术原理解析站点（Hugo/Thulite）。

## 本地开发

```bash
npm install
npm run dev
```

## Cloudflare Pages 部署

### 1. 一次性准备

1. 在 Cloudflare Dashboard 创建 Pages 项目，项目名保持为 `cloudnative-cookbook`（或自行修改 `CF_PAGES_PROJECT_NAME`）。
2. 本地登录 Cloudflare：

```bash
npm run cf:whoami
```

### 2. 本地部署（手动）

```bash
npm run cf:deploy
```

### 3. GitHub Actions 自动部署

仓库已内置工作流 `/Users/yulei/workspace/cloudnative-cookbook/.github/workflows/deploy.yml`，推送 `main` 分支会自动部署到 Cloudflare Pages。

请在 GitHub 仓库 Secrets 中设置：

- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`

如需修改 Pages 项目名，请编辑工作流中的 `CF_PAGES_PROJECT_NAME`。
