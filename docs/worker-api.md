# Worker HTTP API

游戏 worker 是一个 Cloudflare Worker（Hono），由模板的 `worker/src/index.ts` 把你的 `logic/index.ts` 包装成下面这些端点。**这套端点对所有游戏都一样，通常不需要改动。**

平台通过 HTTP 调用其中带 🔒 的端点。`/metadata` 在注册和选游戏时被调用，`/__nexus_worker_verify` 用来确认这是一个合法的游戏 worker。

## 端点一览

| 方法 | 路径 | 调用方 |
| --- | --- | --- |
| GET | `/metadata` | 平台 🔒 / 调试 |
| GET | `/__nexus_worker_verify` | 平台 🔒 |
| GET | `/game-ui.html` | 浏览器 iframe |
| GET | `/game-ui.js` | 浏览器 iframe |
| GET | `/style.css` | 浏览器 iframe |
| POST | `/init` | 平台 🔒 |
| POST | `/current-role` | 平台 🔒 |
| POST | `/legal-actions` | 平台 🔒 |
| POST | `/action` | 平台 🔒 |
| POST | `/is-terminal` | 平台 🔒 |
| POST | `/perspective` | 平台 🔒 |
| POST | `/state-prompt` | 平台 🔒 |

## GET /metadata

返回 `getMetadata()` 的结果，并注入 UI 地址：

```json
{
  "id": "my-game",
  "name": "我的游戏",
  "description": "...",
  "minPlayers": 2,
  "maxPlayers": 4,
  "roleIds": ["player_1", "player_2", "player_3", "player_4"],
  "ui": { "mode": "url", "url": "https://<worker>/game-ui.html" }
}
```

## GET /__nexus_worker_verify

返回纯文本：

```
NEXUS_GAME_WORKER_VERIFIED_V1
```

平台用它确认 worker 是有效的游戏实现。

## POST /init

请求：

```json
{ "players": ["player_1", "player_2"], "options": {} }
```

返回：初始 `GameState`。人数不合法时返回 `400 { "error": "..." }`。

## POST /current-role

请求：

```json
{ "state": { "...": "..." } }
```

返回：

```json
{ "roleId": "player_1" }
```

## POST /legal-actions

请求：

```json
{ "state": {}, "roleId": "player_1" }
```

返回 `ActionSpec`：

```json
{
  "actions": [
    { "action_id": "place", "description": "落子", "params_schema": { "type": "object" } }
  ]
}
```

## POST /action

请求：

```json
{ "state": {}, "action": { "action_id": "pass", "role_id": "player_1", "params": {} } }
```

返回 `ActionResult`：

```json
{ "success": true, "nextState": {}, "events": [], "commands": [] }
```

失败时：

```json
{ "success": false, "error": "不是你的回合" }
```

如果逻辑声明了 `actionSchemas`，参数会先经过 Zod 校验。

## POST /is-terminal

请求：

```json
{ "state": {} }
```

返回：

```json
{ "isTerminal": true, "winners": ["player_1"] }
```

`winners` 只在 `isTerminal` 为 `true` 时非空。

## POST /perspective

请求：

```json
{
  "state": {},
  "roleId": "player_1",
  "wholeHistory": [],
  "diffHistory": []
}
```

返回 `RolePerspective`：

```json
{
  "global_rules": "...",
  "whole_history": [],
  "diff_history": [],
  "current_state": {},
  "your_role": { "identity": "player_1", "goal": "...", "is_current": true },
  "action_space_definition": { "actions": [] },
  "message": "轮到你了"
}
```

观战者传入 `roleId: null`。

## POST /state-prompt

请求：

```json
{ "perspective": { "...": "..." } }
```

返回：

```json
{ "statePrompt": "..." }
```

仅在 LLM 回合使用。

## 静态资源端点

- `/game-ui.html`：iframe 承载页，引用 `/_ui.js` 与 `/style.css`。
- `/game-ui.js`：代理到 `_ui.js` 并补上 CORS 与 `Content-Type`。
- `/style.css`：vite 产物，同样补 CORS。

这三个端点由模板处理，不需要你改。`worker/public/` 下的其他文件（图片、字体、封面）由 `[assets]` 直接托管。

## 本地手测

```bash
curl http://localhost:8788/metadata
curl http://localhost:8788/__nexus_worker_verify
curl -X POST http://localhost:8788/init \
  -H 'Content-Type: application/json' \
  -d '{"players":["player_1","player_2"]}'
```
