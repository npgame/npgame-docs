# 本地开发

`@nexusgame/dev`（命令名 `npgame-dev`）提供一个跑在 Miniflare 上的本地引擎，不依赖后端、前端或 LLM 服务。它只做三件事：起引擎、把房间接到你的本地 worker、驱动回合。

它**不生成脚手架、不做认证、不负责发布**。

## 前置条件

- Node.js >= 18
- pnpm
- 一个已经 `pnpm install` 的游戏仓库（模板见 [`npgame/nexus-game-template`](https://github.com/npgame/nexus-game-template)）

## 三个终端

```bash
# 终端 A：游戏 worker
cd my-game && pnpm dev            # wrangler dev，默认 http://localhost:8788

# 终端 B：本地引擎
npx @nexusgame/dev start          # 默认 http://localhost:56512

# 终端 C：驱动游戏
npx @nexusgame/dev setup --worker-url http://localhost:8788
```

`start` 会把状态持久化到当前目录的 `.nexus/`。删掉它就能从干净状态开始。

## 命令

### `npgame-dev start`

```bash
npgame-dev start [-p, --port <port>]
```

前台运行本地引擎，默认端口 `56512`。Ctrl+C 停止。

### `npgame-dev setup`

```bash
npgame-dev setup -w http://localhost:8788 [-p <port>] [-n <players>]
```

一条龙完成：创建房间 → `set-game` → `start-game`。

- `-w, --worker-url`：游戏 worker 地址（必填）。
- `-n, --players`：指定人数，适用于 UNO 这类支持可变人数的游戏；不传则由游戏元数据决定。
- 如果房间已经在进行中，会先停止再重新开局——改完逻辑重新 `setup` 即可。

### `npgame-dev state`

```bash
npgame-dev state
```

打印房间全局状态，包括权威 `gameState`、角色映射和完整历史。

### `npgame-dev perspective`

```bash
npgame-dev perspective <roleId>
```

打印该角色视角（`RolePerspective`）以及可执行动作。末尾会输出一个调试 URL：

```
📺 调试 GUI 页面:
http://localhost:8788/game-ui.html?stateUrl=...&actionUrl=...&token=...
```

在浏览器打开它，可以直接点 UI 手动操作，不需要经过平台。这里用的是 `iframe-entry.tsx` 的本地直连模式。

### `npgame-dev action`

```bash
npgame-dev action <roleId> '{"action_id":"place","params":{"row":7,"col":7}}'
```

以指定角色提交动作。`role_id` 由命令参数填入，JSON 里只需要 `action_id` 和 `params`。

## 典型调试循环

1. 改 `logic/index.ts`。
2. 重新 `setup`：
   ```bash
   npgame-dev setup -w http://localhost:8788
   ```
3. `state` 看当前轮到谁。
4. `perspective <roleId>` 检查视角里有没有泄漏隐藏信息。
5. `action <roleId> '<json>'` 推进一步。
6. 改 `ui/ui.tsx` 时，`pnpm build:ui` 后刷新调试 GUI 页面即可。

## 常见问题

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `setup` 报 `ECONNREFUSED` | 本地引擎没启动 | 另开终端 `npgame-dev start` |
| `set-game` 失败 | worker 没启动或地址不对 | 确认 `pnpm dev` 在跑，`-w` 用 `http://localhost:8788` |
| 端口冲突 | `56512` 被占用 | 所有命令加同一个 `--port` |
| 本地能玩、线上不能 | 构建/模板差异 | 本地跑 `pnpm build`；确认没有依赖 `worker/wrangler.toml` 的本地特有配置 |
| `action` 无效 | JSON 写错或漏了 `action_id` | 检查引号和字段名 |

## 本地引擎做了什么

本地引擎就是 `nexus-engine` 的 GameDO 单文件 bundle。它按真实平台的顺序调用你的 worker：`/metadata` → `/init` → `/current-role` → `/perspective` → `/action` → `/is-terminal`。因此本地能跑通，基本可以确定线上一致；唯一差异是发布时的构建环境。
