# cassial.site

个人站点总入口（统一门户）+ 独立子应用架构。

## 架构

| 域名 | 内容 | 托管 | 仓库 |
|---|---|---|---|
| `cassial.site` | 门户首页（应用卡片 + 前端密码门） | GitHub Pages（本仓库 `portal/` 目录） | `Ricardo-Rov/cassial.site` |
| `jizhang.cassial.site` | 简账（个人记账，云端同步） | Cloudflare Workers + D1（服务端密码门） | `apps/jianzhang/`（独立部署） |

DNS 由 Cloudflare 管理：`@`/`www` 的 A/CNAME 记录（灰云，仅 DNS）指向 GitHub Pages；`jizhang` 由 Workers 自定义域自动管理。

## 门户（本仓库）

- 纯静态，无构建步骤，源码在 `portal/`。
- 推送到 `main` 后 GitHub Actions（`.github/workflows/pages.yaml`）自动部署 `portal/` 到 Pages。
- **密码门**：初始密码哈希内置于 `portal/index.html` 的 `PASSWORD_SHA256`；用户在页脚"修改密码"改密后，新哈希存在该浏览器的 localStorage（仅当前浏览器生效）。修改初始密码：改哈希值即可。
- **新增应用卡片**：编辑 `portal/index.html`，复制一张 `.app-card` 改链接、名称、简介、徽章。

## 简账（apps/jianzhang/）

源自从 ChatGPT Sites 平台导出的 vinext 应用，已适配为自托管：

- 移除 ChatGPT 登录，改为 Worker 层密码门（`build/sites-worker.ts`）：
  - 初始密码来自 Worker secret `SITE_PASSWORD`
  - 改密后哈希存 D1 `settings` 表，改密后所有会话失效
  - `/login`（GET/POST）、`/logout`、`/change-password`（POST，需登录）
- 账目数据全部归到固定单用户 `owner`（`app/api/ledger/route.ts`）
- 数据库：Cloudflare D1 `jianzhang-db`，迁移文件在 `drizzle/`

### 简账常用命令

```bash
cd apps/jianzhang
export PATH="/c/Program Files/nodejs:$PATH"   # Windows Git Bash

corepack pnpm install        # 装依赖
npm run build                # 构建到 dist/
npm start                    # 本地预览（127.0.0.1，用 .dev.vars 里的 SITE_PASSWORD）

# 本地库迁移（首次）
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0001_settings.sql

# 部署（注意：每次 build 后 wrangler.json 会重置，需重新打补丁）
python -c "
import json
p='dist/server/wrangler.json'
c=json.load(open(p,encoding='utf-8'))
c['name']=c['topLevelName']='jianzhang'
c['d1_databases']=[{'binding':'DB','database_name':'jianzhang-db','database_id':'9e038db8-696c-4b5f-a664-d24e7ae804f4'}]
c['routes']=[{'pattern':'jizhang.cassial.site','custom_domain':True}]
json.dump(c,open(p,'w',encoding='utf-8'),ensure_ascii=False)
"
node ./node_modules/wrangler/bin/wrangler.js deploy --config dist/server/wrangler.json
```

## 新增子应用 SOP

1. 解压 zip 到 `apps/<name>/`，git init
2. 移除平台登录（`app/chatgpt-auth.ts`），API 改为固定 owner
3. 把 `apps/jianzhang/build/sites-worker.ts` 的密码门复制过去
4. `wrangler d1 create <name>-db` → 应用 `drizzle/` 迁移（`--remote`）
5. `npm run build` → 补丁 `dist/server/wrangler.json`（name、database_id、子域名 route）→ `wrangler secret put SITE_PASSWORD` → `wrangler deploy`
6. 门户 `portal/index.html` 加卡片

## 安全说明

- 门户密码门是前端校验，仅防普通访问者（GitHub Pages 无服务端）。
- 简账密码门在 Worker 服务端强制执行，HTML/静态资源/API 全部受保护；密码不明文存储（D1 中为 SHA-256 哈希，初始值来自 Worker secret）。
