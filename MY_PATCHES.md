# 我的 Hermes 本地修改说明

> 本文件说明我在 `wociaozhongyunonghaole/hermes-agent`（fork 自
> `NousResearch/hermes-agent`）中保留的两处修改。
>
> **给阅读本仓库的 AI / 维护者**：每个修改都写明了「症状 / 根因 / 改法 / 边界」，
> 并附真实文件路径与行号。修改均已通过 vitest 单元测试。
> **这两处修改尚未提交给上游官方仓库。**

---

## 修改一：折叠网关分组时，遗留会话收不起来

**分支**：`fix/sidebar-connectionless-gateway-rows`
**提交**：`64dd10f3fb`
**涉及文件**：
- `apps/desktop/src/app/chat/sidebar/gateway-group-model.ts`
- `apps/desktop/src/app/chat/sidebar/gateway-groups.tsx`
- 对应测试：`gateway-group-model.test.ts`、`gateway-groups.test.tsx`

### 症状（用户实际遇到）

侧边栏里，会话按「网关 → 档案」两级分组：

```
🖥 我的电脑
    ├─ default
    └─ default
☁ 远程网关
    └─ default
```

点击一级标题（如「我的电脑」）折叠时，**有几个二级组折叠不进去**，
仍然摊开在列表里。用户原话：「一级折叠二级的时候，有几个二级没有被折叠到一级里」。

### 根因

折叠的唯一开关在 `gateway-groups.tsx`：

```ts
const open = !collapsed.includes(group.id)   // :178
...
{open && ( ...{renderRows(sessions)}... )}   // :310
```

**折叠完全依靠 `group.id` 精确匹配。**

而这个 id 由数据侧生成（`gateway-group-model.ts` 上游原版 :22）：

```ts
const connectionId = session.connection_id || null          // ← 关键
const id = JSON.stringify([connectionId, profile])
```

部分会话没有 `connection_id` 字段（旧版本生成的记录，或网关多路复用
切换间隙产生的记录），于是它们的 id 是：

```
[null, "default"]
```

而网关分组头的 id 是：

```
["gateway", "local"]
```

两者对不上。更关键的是 `gateway-groups.tsx` 里有这一段：

```ts
if (nested || embedded || !group.connectionId) {
  sections.push(group)
  continue        // ← 无归属的组被推到顶层，永远不进网关子节点表
}
```

所以这些遗留会话**从来没有被挂到任何一级标题底下**。
折叠一级标题时自然够不着它们 —— 不是「折叠失效」，是它们压根不在那个容器里。

### 改法

```ts
const fallbackConnectionId = registry?.primary || 'local'
const connectionId = session.connection_id || fallbackConnectionId
```

把无归属的行锚到注册表的 **primary connection**（缺省 `'local'`），
使它们的 id 变成 `["local","default"]`，从而落进对应网关分组头之下，
折叠命令即可覆盖到。

### 边界（重要）

此改动**不猜测、也不写入会话的真实归属**：

- 会话自身的 `connection_id` 字段**不被修改**
- **显式设置了**归属的会话**永不被改写**
- 仅在「分组 / 折叠」的**渲染层**做锚定

因此远程网关里的同名档案仍保持各自独立的分组，不会被错误吞并。

### 与上游设计的关系

上游曾刻意让无归属行保持 unassigned，代码注释为：

> `Unknown legacy ownership stays unassigned; never guess a local gateway.`

并有测试锁定（`leaves legacy rows without a connection unassigned
instead of guessing a gateway`）。

本改动不否定「不要臆造归属」这一原则，但它修掉了该设计的副作用：
**因为折叠按 id 精确匹配，这些行无法被任何一级折叠隐藏**。
原测试已随之更新，以描述修正后的行为。

---

## 修改二：机器人花名册拖拽箭头不会变 / 拖不动

**分支**：`fix/roster-drag-global-dnd`
**提交**：`5cafae9c89`
**涉及文件**：
- `apps/desktop/src/plugins/hermes-bots/roster-drag.ts`（新增）
- `apps/desktop/src/plugins/hermes-bots/user-sections-ui.tsx`
- `apps/desktop/src/plugins/hermes-bots/bot-row.tsx`
- 新增测试：`roster-drag.test.tsx`（101 行）

### 症状

在机器人花名册（roster）里拖动一个 bot 或分组到别的分区时：

- 鼠标光标显示「禁止」符号，而不是「移动」
- 或者拖过去了但**没有任何反应**

### 根因

花名册用的是**浏览器原生 HTML5 拖拽**（`dataTransfer` + `dragover/drop`），
而侧边栏文件树同时注册了 **dnd-kit 的全局 backend**。

原生拖拽事件会一路**冒泡**上去，被 dnd-kit 的全局监听接手。
dnd-kit 不认识这套自定义的 MIME（`application/x-hermes-bot-key`），
于是把 `dropEffect` 重置为 `none` —— 表现就是「禁止」光标，
或拖拽被判定为无效而被吞掉。

上游原代码（`bot-row.tsx`）在 `onDragStart` 里设置了数据和效果：

```ts
event.dataTransfer.setData(BOT_DRAG_MIME, rosterKey)
event.dataTransfer.effectAllowed = 'move'
$draggingBot.set(rosterKey)
```

**但没有调用 `stopPropagation()`**，事件继续冒泡给全局 backend。

### 改法

抽出统一入口 `roster-drag.ts`：

```ts
export function startRosterDrag(event, key) {
  event.stopPropagation()               // ← 新增：拦住冒泡
  event.dataTransfer.setData(BOT_DRAG_MIME, key)
  event.dataTransfer.effectAllowed = 'move'
  $draggingBot.set(key)
}
```

`bot-row.tsx` 的两个 `onDragStart`（`BotRow` 和 `GroupRow`）改为调用它。

同时在 `SectionDropZone` 的 `onDragEnter` / `onDragOver` / `onDrop`
三个处理里补 `stopPropagation()`，防止落点判定也被全局 backend 抢走。

另补 robustness：拖拽状态 `$draggingBot` 增加 `dragend` / `drop` /
`blur` 三个兜底清理监听 —— 原先只有 `Escape` 键能取消，
一旦拖拽在别处结束，状态会卡住不复位。

### 边界

只作用于机器人插件自身的拖拽路径，不触碰 dnd-kit 的
通用配置，也不影响侧边栏其他部分的拖拽行为。

---

## 测试状态

| 修改 | 涉及测试文件 | 结果 |
|---|---|---|
| 修改一 | `gateway-group-model.test.ts`、`gateway-groups.test.tsx` | 5 passed |
| 修改二 | `roster-drag.test.tsx` | 新增 101 行，通过 |

运行方式（在 `apps/desktop` 目录下）：

```bash
npx vitest run --project ui \
  src/app/chat/sidebar/gateway-group-model.test.ts \
  src/app/chat/sidebar/gateway-groups.test.tsx \
  src/plugins/hermes-bots/roster-drag.test.tsx
```

---

## 上游是否需要知道

两条都可独立提 PR 给 `NousResearch/hermes-agent`。
截至 2026-10-04，上游对这两个文件的最近改动为 2026-09-24，
**无人抢先修复**；`gateway-group-model.ts` 全文仅被改过 2 次，属活跃新代码。

若提 PR，修改一的描述需正面回应上游那句
`never guess a local gateway` —— 说明本改动不臆造归属，只修复折叠覆盖不到的问题。
