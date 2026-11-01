# 游戏逻辑 API

`logic/index.ts` 默认导出一个实现 `GameLogic<TState>` 的实例：

```ts
import type { GameLogic } from '@nexusgame/game-sdk';

class MyGameLogic implements GameLogic<MyState> { /* ... */ }

export default new MyGameLogic();
```

也可以继承 `BaseGameLogic<TState>`，它提供了 `serializeState` / `deserializeState` / `cloneState` 的默认实现。

## 方法总览

| 方法 | 时机 | 返回 |
| --- | --- | --- |
| `getMetadata()` | 注册、选游戏、开局前 | 元数据（含人数、角色、UI） |
| `initState(ctx)` | 开局 | 初始 `GameState` |
| `getCurrentRole(state)` | 每次状态变化 | 当前行动者的 roleId |
| `getLegalActions(state, roleId)` | 生成视角、给 LLM/UI 用 | 该角色可执行的动作 |
| `applyAction(state, action)` | 提交动作 | `{ success, nextState }` 或 `{ success: false, error }` |
| `isTerminal(state)` | 每次状态变化 | 是否结束 |
| `getWinners(state)` | 终局 | 获胜 roleId 数组或 `null` |
| `toRolePerspective(...)` | 每次状态变化 | 该角色能看到的信息 |
| `generateStatePrompt(perspective)` | LLM 回合 | 给 LLM 的自然语言状态 |

## getMetadata

```ts
interface GameMetadata {
  id: string;
  name: string;
  description: string;
  minPlayers: number;
  maxPlayers: number;
  /** 角色列表；可变人数时用 { 人数: roleIds[] } */
  roleIds?: string[] | Record<number, string[]>;
  /** 每个可选人数的展示文案，如 { 2: '双人', 3: '三人' } */
  playerCountLabels?: Record<number, string>;
  /** 是否让 LLM 记住跨回合的记忆 */
  enable_llm_memory?: boolean;
  auto_save_mode?: 'enabled' | 'disabled';
  /** 房间底栏显示的状态文案 */
  getStatusText?: (perspective: RolePerspective) => string;
  /** 由 worker 注入，不要自己填 */
  ui?: { mode: 'url'; url: string };
}
```

- `id` / `name` 直接读 `nexus.game.json`，不要硬编码。平台注册时会写入真实值。
- `roleIds` 的顺序就是座位顺序，也是 `initState` 里 `ctx.players` 的来源。
- 固定人数用 `string[]`；支持多种人数用 `Record<number, string[]>`，例如狼人杀：

```ts
roleIds: {
  5: ['werewolf', 'werewolf', 'seer', 'villager', 'villager'],
  6: ['werewolf', 'werewolf', 'seer', 'witch', 'villager', 'villager'],
}
```

## initState

```ts
initState(ctx: InitContext): TState {
  const players = ctx.players; // roleId 数组
  // 洗牌等随机性必须发生在这里，并把结果写进 state
  const deck = shuffleWithSeed(/* ... */);
  return { deck, hands: deal(deck, players), currentRole: players[0], turn: 1 };
}
```

- `ctx.options` 是平台透传的可选参数，可忽略。
- 人数不合法时直接 `throw new Error('...')`，平台会把错误展示给房主。

## getCurrentRole

返回当前该行动的角色 ID。这是回合推进的唯一真相来源。

```ts
getCurrentRole(state: MyState): string {
  return state.currentRole;
}
```

如果需要跳过已出局玩家，在这里决定“下一个是谁”，而不是让 UI 去猜。

## getLegalActions

```ts
getLegalActions(state: MyState, roleId: string | null): ActionSpec {
  if (roleId === null) return { actions: [] };        // 观战者
  if (state.currentRole !== roleId) return { actions: [] };
  return {
    actions: [
      { action_id: 'place', description: '落子', params_schema: { /* JSON Schema */ } },
    ],
  };
}
```

`ActionSpec` 会原样进入 `perspective.action_space_definition`，UI 和 LLM 都用它来决定能做什么。`description` 是给 LLM 看的，写清楚。

## applyAction

```ts
applyAction(state: MyState, action: Action): ActionResult<MyState> {
  if (action.role_id !== state.currentRole) {
    return { success: false, error: '不是你的回合' };
  }
  if (action.action_id !== 'place') {
    return { success: false, error: `未知行动：${action.action_id}` };
  }

  const nextState = structuredClone(state); // 或 JSON 深拷贝
  // ... 修改 nextState

  return {
    success: true,
    nextState,
    // 可选：写入历史事件，会出现在 perspective 的 diff_history 里。
    // timestamp 用 state 里的回合序号等确定性值，不要读系统时钟。
    events: [{ turn: state.turn, role_id: action.role_id, action, timestamp: state.turn }],
  };
}
```

- 必须返回**新** state，不能原地修改入参。
- `success: false` 时平台不会推进回合，错误信息会回传给提交方。
- `events` 可选，用来生成人类可读的对局记录。
- `commands` 可选，支持 `{ type: 'SAVE_STATE', name }` 和 `{ type: 'CLEAR_HISTORY' }`。

### 用 Zod 自动校验参数

在类上声明 `actionSchemas`，worker 的 `/action` 会先校验参数再调用 `applyAction`：

```ts
import { z } from '@nexusgame/game-sdk';

export class GomokuLogic extends BaseGameLogic<GomokuState> {
  actionSchemas = {
    place: z.object({
      row: z.number().int().min(0).max(14),
      col: z.number().int().min(0).max(14),
    }),
  };
}
```

校验失败会直接返回 `{ success: false, error: '参数校验失败: ...' }`，不会进入你的逻辑。注意只有 `BaseGameLogic` 提供了 `validateAndApply`，worker 通过它自动接入校验。

## isTerminal / getWinners

```ts
isTerminal(state: MyState): boolean {
  return state.winner !== null || state.boardFull;
}

getWinners(state: MyState): string[] | null {
  return state.winner ? [state.winner] : null;
}
```

`isTerminal` 返回 `true` 后引擎停止接受动作；`getWinners` 只在终局被调用。

## toRolePerspective

```ts
toRolePerspective(
  state: MyState,
  roleId: string | null,
  wholeHistory: HistoryEvent[],
  diffHistory: HistoryEvent[]
): RolePerspective<MyState> {
  const spectator = roleId === null;

  return {
    global_rules: this.getMetadata().description,
    whole_history: wholeHistory,
    diff_history: diffHistory,
    current_state: spectator ? state : redact(state, roleId),
    your_role: {
      identity: spectator ? 'Spectator' : roleId,
      goal: spectator ? '观战' : goalFor(roleId),
      is_current: !spectator && state.currentRole === roleId,
    },
    action_space_definition: this.getLegalActions(state, roleId),
    message: spectator ? '观战中' : state.currentRole === roleId ? '轮到你了' : '等待其他玩家',
  };
}
```

**铁律**：`current_state` 只放这个角色有权知道的信息。别的角色的手牌、真实身份、牌堆顺序都不能出现。观战者 `roleId === null` 可以返回全量。

常见的泄漏方式：

- 直接把整个 `state` 塞进 `current_state`；
- 把“所有玩家的私有字段”作为一个对象下发，只靠 UI 隐藏；
- 在 `message` 里透露别人的身份。

## generateStatePrompt

仅 LLM 回合使用，返回给模型的文本。

```ts
generateStatePrompt(perspective: RolePerspective<MyState>): string {
  const s = perspective.current_state;
  return [
    '# 当前局面',
    `回合：${s.turn}`,
    `你的身份：${perspective.your_role.identity}`,
    `目标：${perspective.your_role.goal}`,
    `可选动作：${perspective.action_space_definition.actions.map(a => a.action_id).join(', ')}`,
    `状态：${JSON.stringify(s)}`,
  ].join('\n');
}
```

不实现时 `BaseGameLogic` 会退化成 `JSON.stringify(perspective)`。建议手写，给 LLM 的提示越结构化，行动质量越高。

## 常见设计模式

| 场景 | 做法 |
| --- | --- |
| 卡牌、手牌 | 牌堆和所有手牌放 state；`toRolePerspective` 只下发本人手牌 |
| 隐藏身份、多阶段 | `phase` + `pendingRoles` 记录子阶段；按阶段过滤可见信息 |
| 可变人数 | `roleIds` 用 `Record<number, string[]>`，在 `initState` 里按实际人数取角色 |
| 复杂走子规则 | 抽成 `computeLegalMoves(state)`，逻辑、UI、LLM 三处共用 |
| 随机性 | 只在 `initState` 洗牌并写入 state，其余逻辑保持确定性 |
