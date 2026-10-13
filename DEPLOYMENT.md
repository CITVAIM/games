# 小游戏合集首页 部署流程（Cloudflare Pages + GitHub 自动部署）

> 记录把「游戏首页」部署到 Cloudflare Pages 并实现 push 自动部署的全过程与关键参数，供后续 AI/维护者复现或排障。
> 本文档**不要提交到仓库**（含敏感信息，见 §5）。流程与 2048 / 扫雷 项目完全一致。

## 1. 最终架构

```
本地 repo  /Users/lvhengchang/Documents/games
   │  git push origin main
   ▼
GitHub  CITVAIM/games  (public)
   │  repo webhook  (event: push)
   ▼
Cloudflare Pages Deploy Hook  (URL 见 §5)
   │  POST
   ▼
Cloudflare Pages 项目 "games"  →  https://games-4n6.pages.dev
```

首页卡片指向两个游戏的线上地址：
- 2048 → https://2048-cjo.pages.dev
- 扫雷 → https://minesweeper-dz8.pages.dev

要点：**不用 Cloudflare 原生 Git 集成**（对 API 创建的项目不触发构建），改用「GitHub 仓库 webhook → CF Deploy Hook」。无需 GitHub Actions、无需 workflow 文件。

## 2. 关键标识

| 项 | 值 |
| --- | --- |
| Cloudflare 账号 | ChangZhou Account |
| Cloudflare Account ID | `72d370a3ad2667dd62f989df6bd7cc14` |
| CF 项目名 | `games` |
| 生产域名 | https://games-4n6.pages.dev |
| 生产分支 | `main` |
| GitHub 仓库 | `CITVAIM/games` |
| GitHub repo_id | `1417206197` |
| GitHub 登录账号 | `CITVAIM` |
| wrangler 凭据 | `~/Library/Preferences/.wrangler/config/default.toml`（OAuth Token） |

## 3. 本地 git 独立仓库

`/Users/lvhengchang/Documents/games` 是**独立 git 仓库**，与父目录及兄弟目录（2048、minesweeper）的 git 无关。

```bash
git init /Users/lvhengchang/Documents/games
```

`.gitignore`：`.DS_Store`、`.wrangler/`

## 4. 部署步骤（复现用）

### 4.1 创建 GitHub 仓库并推送

```bash
cd /Users/lvhengchang/Documents/games
gh repo create games --public --source=. --remote=origin --push
```

> 注：此命令推送成功后可能短暂卡住，属正常，`ls-remote` 确认 main 已在远端即可。

### 4.2 创建 Cloudflare Pages 项目（GitHub 源）

```bash
CFG="$HOME/Library/Preferences/.wrangler/config/default.toml"
TOKEN=$(sed -n 's/.*oauth_token *= *"\(.*\)"/\1/p' "$CFG")
ACC=72d370a3ad2667dd62f989df6bd7cc14

curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/pages/projects" \
  -d '{
    "name": "games",
    "production_branch": "main",
    "source": {"type": "github", "config": {
      "owner": "CITVAIM", "repo_name": "games", "production_branch": "main",
      "deployments_enabled": true, "production_deployments_enabled": true,
      "pr_comments_enabled": true, "preview_deployment_setting": "all"
    }},
    "build_config": {"build_command": "", "destination_dir": "/", "root_dir": ""}
  }'
```

### 4.3 解决「push 不触发构建」：Deploy Hook + 仓库 webhook

```bash
# 创建 deploy hook，拿到 hook_id
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/pages/projects/games/deploy_hooks" \
  -d '{"name":"github-actions","branch":"main"}'
# hook 触发 URL：https://api.cloudflare.com/client/v4/pages/webhooks/deploy_hooks/<hook_id>

# 在 GitHub 仓库上挂 webhook，指向 hook URL（事件：push）
gh api repos/CITVAIM/games/hooks -X POST \
  -f "name=web" -F "active=true" -F "events[]=push" \
  -f "config[url]=<DEPLOY_HOOK_URL>" \
  -f "config[content_type]=json"
```

## 5. 敏感值（不要提交/外泄）

- Deploy Hook id：`a97f224d-d74e-4076-aa7f-0a0d06ba4332`
- Deploy Hook URL（**等同部署凭据，任何人拿到即可触发构建**）：
  `https://api.cloudflare.com/client/v4/pages/webhooks/deploy_hooks/a97f224d-d74e-4076-aa7f-0a0d06ba4332`
- GitHub webhook id：`696786345`
- 仓库 secret：`CF_DEPLOY_HOOK`（内容即上面的 hook URL）

## 6. 常用操作

```bash
# 手动触发一次部署
curl -fsS -X POST "https://api.cloudflare.com/client/v4/pages/webhooks/deploy_hooks/a97f224d-d74e-4076-aa7f-0a0d06ba4332"

# 查看部署状态
CFG="$HOME/Library/Preferences/.wrangler/config/default.toml"
TOKEN=$(sed -n 's/.*oauth_token *= *"\(.*\)"/\1/p' "$CFG")
ACC=72d370a3ad2667dd62f989df6bd7cc14
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$ACC/pages/projects/games/deployments" \
  | python3 -m json.tool

# 查看 webhook 投递记录
gh api repos/CITVAIM/games/hooks/696786345/deliveries \
  --jq '.[] | {event, status: .status_code, at: .delivered_at}'
```

## 7. 新增游戏到首页

1. 部署好新游戏，拿到其线上地址。
2. 在 `index.html` 的 `<main class="grid">` 里复制一张 `<a class="card">`，改 `href`、`--accent`、标题与描述。
3. `git commit && git push`，webhook 会自动重新部署首页。

## 8. 坑与注意事项

1. **本机普通 `git push` 会失败**（无凭据助手）。内联凭据方案：
   ```bash
   GH_TOKEN_PUSH=$(gh auth token) git -c credential.helper='!f() { echo username=x-access-token; echo "password=$GH_TOKEN_PUSH"; }; f' push origin main
   ```
2. **`gh` token 缺 `workflow` scope**，无法推送 `.github/workflows/*`；本项目不需要。
3. **`wrangler pages project create` 要加 `--force`**，否则被委派到 Workers 而失败；建好后不要再用。
4. Deploy Hook 若泄露，删除并重建，再更新 webhook 的 `config[url]`。
5. 修改仓库/域名/hook 后，同步更新本文档参数。
