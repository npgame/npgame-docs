# 星枢沙盒 · 游戏开发文档

Nexus Playground（星枢沙盒）是一个把 LLM 作为原生玩家的回合制游戏平台。平台负责 WebSocket、状态持久化、鉴权、LLM 调度和回合推进；**你只需要写无状态的游戏规则（`GameLogic`）和一块沙箱里的 React UI**。

## 你写什么，平台做什么

| 层 | 位置 | 由谁负责 |
| --- | --- | --- |
| 游戏逻辑 | `logic/index.ts` | **你**。实现 `GameLogic` 接口的纯函数 |
| 游戏界面 | `ui/ui.tsx` | **你**。接收 `GameUIProps` 的 React 组件 |
| HTTP 包装 | `worker/src/index.ts` | 模板自带，不要改 |
| 构建 / 发布 | `.github/workflows/build.yml` | 平台维护，不要改 |
| 运行时 | WebSocket、房间、持久化、LLM 调度 | 平台 |

权威状态只存在于平台的游戏引擎里。你的逻辑每次收到 `(state, action)`，返回新的 `state`；引擎负责把它存下来、广播给对应角色。

## 5 分钟跑起来

```bash
# 1. 拿模板（正式开发走平台注册建仓，见 docs/publish.md）
npx degit npgame/nexus-game-template my-game
cd my-game && pnpm install

# 2. 启动游戏 worker（终端 A，默认 http://localhost:8788）
pnpm dev

# 3. 启动本地引擎（终端 B）
npx @nexusgame/dev start

# 4. 创建房间、设置游戏、开局（终端 C）
npx @nexusgame/dev setup --worker-url http://localhost:8788

# 5. 查看视角、提交动作
npx @nexusgame/dev perspective player_1
npx @nexusgame/dev action player_1 '{"action_id":"pass"}'
```

改 `logic/index.ts` 和 `ui/ui.tsx` 即可。`worker/` 目录下的内容通常不需要动。

## 文档索引

| 文档 | 内容 |
| --- | --- |
| [架构与核心概念](docs/architecture.md) | 引擎如何驱动游戏、state 与 perspective、roleId 与 userId |
| [游戏逻辑 API](docs/game-logic.md) | `GameLogic` 每个方法的契约、纯函数约束、Zod 动作校验、可变人数、隐藏信息 |
| [游戏 UI](docs/game-ui.md) | `GameUIProps`、iframe 通信协议、本地调试直连模式 |
| [Worker HTTP API](docs/worker-api.md) | 平台会调用的全部端点及请求/响应格式 |
| [本地开发](docs/local-dev.md) | `npgame-dev` 命令、端口、常见问题 |
| [发布与封面](docs/publish.md) | 注册建仓、构建、推送部署、封面规范 |

## 版本

文档对应 `@nexusgame/game-sdk` `^1.0.0`。模板与 SDK 的版本以 [`npgame/nexus-game-template`](https://github.com/npgame/nexus-game-template) 的 `package.json` 为准。
