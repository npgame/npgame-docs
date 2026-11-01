# 游戏 UI

游戏 UI 是一个普通的 React 组件，运行在平台提供的 **sandbox iframe** 里。它拿到当前的 `RolePerspective`，把用户交互转换成 `Action` 发回平台。

## GameUIProps

```ts
interface GameUIProps<TState = any> {
  perspective: RolePerspective<TState>;  // 当前角色视角（已过滤隐藏信息）
  onAction: (action: Action) => void;    // 提交动作
  isMyTurn: boolean;                     // 是否轮到你
  readonly: boolean;                     // 只读（观战、游戏结束、回放）
  metadata?: {
    roomId: string;
    roleId: string;
    playerId?: string;
    roleDisplayMapping?: Record<string, { name: string }>;
  };
}
```

组件通常长这样：

```tsx
const MyUI: React.FC<GameUIProps<MyState>> = ({ perspective, onAction, isMyTurn, readonly }) => {
  const state = perspective.current_state;
  const roleId = perspective.your_role.identity;

  return (
    <div>
      <p>{perspective.message}</p>
      <button
        disabled={!isMyTurn || readonly}
        onClick={() => onAction({ action_id: 'pass', role_id: roleId })}
      >
        结束回合
      </button>
    </div>
  );
};
```

要点：

- **`role_id` 必须带上**，用 `perspective.your_role.identity`（或 `metadata.roleId`）。
- 可执行动作从 `perspective.action_space_definition.actions` 取，不要自己另写一套判断。
- 不满足条件时用 `disabled`，但真正的校验仍然在 `applyAction` 里。
- `readonly` 为 `true` 时禁止一切提交。

## iframe 通信

模板里的 `ui/iframe-entry.tsx` 已处理全部通信，你通常不需要改。协议如下：

**父窗口 → iframe**

```ts
{ type: 'SYNC_STATE', payload: { perspective, isMyTurn, readonly, metadata } }
```

**iframe → 父窗口**

```ts
{ type: 'IFRAME_READY' }                    // 加载完成，请求首帧
{ type: 'ACT', payload: Action }            // 提交动作
```

`SYNC_STATE` 会随每次状态变化重复下发，所以组件应当把它当作完整视图重新渲染，而不是增量合并。

## game-ui.html

`worker/public/game-ui.html` 是 iframe 真正加载的页面，内容固定：

```html
<div id="root"></div>
<script src="/_ui.js"></script>
```

`_ui.js` 和 `style.css` 由 `pnpm build:ui` 生成。`game-ui.html` 是随源码提交的静态文件，不要删除。CSS 建议用 CSS Module（`ui/ui.module.css`），会被 vite 打进 `style.css`。

## 本地调试直连模式

用 `npgame-dev perspective <roleId>` 会输出一个调试 URL，带三个参数：

```
?stateUrl=<worker>/perspective?...&actionUrl=<worker>/action&token=<本地引擎令牌>
```

带这些参数时，`iframe-entry.tsx` 不依赖 postMessage，而是直接：

- `GET stateUrl`（`Authorization: Bearer <token>`）拿视角；
- `POST actionUrl` 提交 `{ roleId, action }`。

这样可以在浏览器里直接打开 iframe 页面手工点，不必经过平台。线上部署后这些参数不存在，会自动回到 postMessage 模式。

## 约束

- iframe 有 sandbox 限制：不要依赖顶层导航、`window.open`、localStorage 跨域共享。
- iframe 尺寸由平台决定（通常占满游戏区）。用 `width/height: 100%` 自适应，不要写死像素高度。
- 不要发网络请求获取游戏状态——所有数据都在 `perspective` 里。
- 不要假设 DOM 之外的全局状态；每次 `SYNC_STATE` 都应根据传入的 props 完整渲染。
- 图片等静态资源放在 `worker/public/`，用相对路径引用；它们会随部署一起上传。
