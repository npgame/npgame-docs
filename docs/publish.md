# 发布与封面

## 发布模型

游戏不需要自己执行 `wrangler deploy`，也不需要持有 Cloudflare 凭据。发布链路固定：

```
git push origin main
  → .github/workflows/build.yml
      pnpm install --frozen-lockfile
      pnpm build
      打包 worker → manifest.json + index.js + public/**
  → GitHub Actions OIDC（短期身份）
  → 平台校验身份与 artifact
  → Cloudflare Direct Upload
  → 游戏自动上线
```

与此相关的两个文件由平台维护，**不要修改**：

- `.github/workflows/build.yml`：固定 workflow。平台签发的 git 凭据没有 `workflows: write` 权限，改了也推不上去。
- `nexus.game.json`：仓库标记，注册时由平台写入。`id` / `name` / `repo` 不要手改。

## 注册并拿到仓库

在平台的创作中心注册游戏（或调用平台提供的注册命令）。平台会：

1. 从 [`npgame/nexus-game-template`](https://github.com/npgame/nexus-game-template) 生成一个专属仓库；
2. 写入真实的 `nexus.game.json`；
3. 配置好仓库的部署凭据。

然后 clone 下来开发：

```bash
git clone <repo-url> my-game
cd my-game
pnpm install
```

> 平台生成的仓库只接受平台签发的短期 Git 凭据，普通 GitHub 账号密码 / 个人 token 无法推送。按平台提示获取即可。

## 构建

```bash
pnpm run build
```

- `build:logic`（tsup）→ `dist/`
- `build:ui`（vite）→ `worker/public/_ui.js` + `style.css`

CI 使用 `pnpm install --frozen-lockfile`，所以 **`pnpm-lock.yaml` 必须提交**。如果改了依赖却没更新 lockfile，构建会直接失败。

## 推送部署

```bash
git add -A
git commit -m "feat: ..."
git push origin main
```

推送后在创作中心查看部署状态：

- `queued` / `uploading` / `verifying`：进行中
- `succeeded`：已上线
- `failed`：失败，会附带错误码和原因

常见失败原因：

| 错误 | 原因 |
| --- | --- |
| `INVALID_ARGUMENT` + 封面比例 | 封面不是 16:9，或超过 5MB |
| 构建失败 | `pnpm-lock.yaml` 未更新、`tsc` 报错、依赖安装失败 |
| 身份校验失败 | 推送的不是 `main`，或 workflow 被改动 |
| artifact 校验失败 | 构建产物缺少 `index.js` 或 `public/` |

## 验证线上实例

```bash
curl https://<worker-url>/metadata
curl https://<worker-url>/__nexus_worker_verify   # NEXUS_GAME_WORKER_VERIFIED_V1
```

然后在平台建房、选游戏、开局，确认端到端可玩。

## 封面

游戏列表展示的封面由仓库里的静态文件决定，约定：

- 路径：`worker/public/cover.{png,jpg,jpeg,webp,svg,gif}`
- 多个同时存在时，按上面的顺序取第一个
- 比例：**16:9**（允许 ±2%），推荐 **1280×720** 或 1920×1080
- 大小：≤ 5MB
- 模板自带一张 1280×720 的占位图，替换成自己的美术即可

封面在构建时随 `public/` 一起进入 artifact。比例不合规会让整次部署失败，并且**在调用 Cloudflare 之前就会被拒绝**，不会留下半成品。

不想用封面就删掉这个文件再部署，游戏会回到无封面状态。

## 更新与下线

- 改代码：继续推 `main`，每次推送都会触发一次部署。
- 改封面：替换 `cover.png` 后推送即可。
- 下线：在创作中心将游戏设为不可见或删除。

## 本地预览

发布前建议先在本地完整跑一遍：

```bash
pnpm dev                                          # worker
npx @nexusgame/dev start                          # 本地引擎
npx @nexusgame/dev setup -w http://localhost:8788 # 开局
```

详见 [本地开发](local-dev.md)。
