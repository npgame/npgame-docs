# 架构与核心概念

## 一句话模型

> 游戏逻辑是一个无状态的纯函数：输入 `(state, action)`，输出 `(nextState | error)`。引擎保存权威 state，把 action 转发给游戏 worker，再把过滤后的视角推给每个角色。

## 一次回合发生了什么

```
玩家点击 UI
  → iframe postMessage { type: 'ACT', payload: action }
  → 平台 WebSocket
  → 引擎 GameDO（权威 GameState）
  → POST <gameWorker>/action { state, action }
  → 你的 applyAction(state, action) → { success, nextState }
  → 引擎保存 nextState
  → POST <gameWorker>/current-role { state }        # 现在轮到谁
  → POST <gameWorker>/perspective { state, roleId } # 每个角色能看到什么
  → SYNC_STATE 推送给各端（人类 UI / LLM）
```

引擎在三个时机调用 `/current-role`：游戏开始时、每次成功动作后、LLM 玩家行动后。返回的 roleId 如果映射到 LLM 玩家，平台就触发 LLM 回合；否则等待人类玩家提交。

## State 与 Perspective

- **`GameState`**：权威状态，包含所有隐藏信息（全部手牌、真实身份、牌堆）。它只存在于引擎和你的 `logic` 里，永远不会原样下发给玩家。
- **`RolePerspective`**：某个角色能看到的视图，由 `toRolePerspective()` 生成。引擎把每个角色各自的视角单独推送。

因此隐藏信息不是靠“前端不显示”来保护的，而是根本不下发。`toRolePerspective` 是唯一的信息出口，也是最容易出安全问题的地方。

观战者的 `roleId` 为 `null`，可以返回全量信息。

## roleId 与 userId

游戏逻辑里操作的是**角色 ID（roleId / 座位）**，不是用户 ID。

- `initState(ctx)` 收到的 `ctx.players` 是 `roleIds` 数组（例如 `['player_1','player_2']`），不是用户。
- state 里的 `currentRole`、action 里的 `role_id`、perspective 里的 `your_role.identity` 都是 roleId。
- `roleId → userId` 的映射（`role_mapping`）由引擎保管。逻辑层既不知道也不需要知道谁坐在哪个座位上。

这带来一个直接好处：同一套逻辑可以让人和 LLM 任意混坐，也可以让观战者进入。

## 纯函数的边界

`GameLogic` 的每个方法都必须是纯函数：

- 不做网络/文件/数据库 I/O；
- 不读当前时间（需要时间就把它写进 state）；
- 不使用 `Math.random()` 改变结果（洗牌应在 `initState` 里完成并把结果写进 state）；
- 不修改入参——`applyAction` 必须先复制再改，返回新对象。

平台可能重放、序列化、跨进程传输 state，任何副作用都会让状态不可复现。

## state 必须能 JSON 往返

state 在引擎和 worker 之间以 JSON 传输，也会被深拷贝。不要放：

- `Map` / `Set` / `Date` / `BigInt` / 函数 / 类实例 / `undefined` 作为关键字段。

用普通对象、数组、字符串、数字、布尔值。时间用 ISO 字符串。

## 游戏结束

- `isTerminal(state)` 返回 `true` 后，引擎不再接受 action。
- `getWinners(state)` 返回获胜的 roleId 数组；平局或无人获胜返回 `null`。
- `getCurrentRole` 在终局可以返回任意值，引擎会以 `isTerminal` 为准。

## 文件职责

```
logic/index.ts          # 你的全部游戏规则（纯函数）
ui/ui.tsx               # 你的 UI（React，跑在 sandbox iframe 里）
ui/iframe-entry.tsx     # 平台通信外壳，一般不用改
worker/src/index.ts     # 把 logic 包成 HTTP API，不要改
worker/public/          # game-ui.html（iframe 承载页）、构建产物、封面
nexus.game.json         # 平台元数据，注册时由平台写入
```
