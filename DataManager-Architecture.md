# DataManager 架构分析：Workspace 切换与本地/云端数据模块对比

## 一、DataManager 概述

`DataManager` 是 Beekeeper Studio 的数据层中枢，定义在 `apps/studio/src/components/data/DataManager.vue` 中。它以一个空模板的 Vue 组件形式存在，不渲染任何 UI，仅负责：

1. 在组件挂载时初始化所有数据模块
2. 监听 `workspaceId` 变化，切换本地/云端模块
3. 以固定间隔轮询云端数据
4. 组件销毁时注销所有已注册模块

---

## 二、数据模块注册与注销机制

### 2.1 DataModules 注册表

在 [DataModules.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/DataModules.ts) 中定义了模块注册表：

```ts
export const DataModules = [
  { path: 'data/queries',         local: UtilQueryModule,           cloud: CloudQueryModule },
  { path: 'data/connections',     local: UtilConnectionModule,      cloud: CloudConnectionModule },
  { path: 'data/queryFolders',    local: LocalQueryFolderModule,    cloud: CloudQueryFolderModule },
  { path: 'data/connectionFolders', local: LocalConnectionFolderModule, cloud: CloudConnectionFolderModule },
  { path: 'data/usedQueries',     local: UtilUsedQueryModule,       cloud: CloudUsedQueryModule },
  { path: 'data/usedconnections', local: UtilUsedConnectionModule,  cloud: UtilUsedConnectionModule },
]
```

每个条目包含：
- `path`：Vuex 模块路径（命名空间）
- `local`：本地工作空间使用的模块实现
- `cloud`：云端工作空间使用的模块实现

### 2.2 mountAndRefresh：注册过程

当 `workspaceId` 变化或组件初次挂载时，`mountAndRefresh` 被调用：

```ts
mountAndRefresh() {
  if (!this.workspace) return
  const scope = this.$store.getters.isUltimate ? this.workspace.type : 'local'
  DataModules.forEach((module) => {
    const choice = module[scope]   // 根据 scope 选择 local 或 cloud
    if (this.$store.hasModule(module.path)) {
      this.$store.unregisterModule(module.path)  // 先注销旧模块
    }
    this.$store.registerModule(module.path, choice) // 再注册新模块
    this.$store.dispatch(`${module.path}/load`)     // 触发 load
  })
}
```

**关键步骤：**
1. **判定 scope**：根据 `workspace.type` 或是否为 Ultimate 用户选择 `local` 或 `cloud` 版本
2. **注销旧模块**：`unregisterModule(module.path)` —— 移除 Vuex store 中已有的模块（同时清空 state、mutations、actions、getters）
3. **注册新模块**：`registerModule(module.path, choice)` —— 注册选中的模块实现
4. **触发 load**：调用 `dispatch('modulePath/load')` 从数据源加载数据

### 2.3 beforeDestroy：全局注销

组件销毁时统一注销所有模块：

```ts
beforeDestroy() {
  if (this.interval) clearInterval(this.interval)
  DataModules.forEach((module) => {
    if (this.$store.hasModule(module.path)) {
      this.$store.unregisterModule(module.path)
    }
  })
}
```

同时停止轮询定时器。

### 2.4 Workspace 切换流程

```
用户切换 Workspace
    │
    ▼
watch: { workspaceId() }
    │
    ▼
mountAndRefresh()
    │
    ├─ 遍历 DataModules
    │     │
    │     ├─ hasModule? ──yes──► unregisterModule (清除旧 state/旧 actions)
    │     │
    │     └─ registerModule(local 或 cloud 版本)
    │
    ├─ dispatch('data/usedconnections/load')
    │
    └─ dispatch('pinnedConnections/loadPins')
```

---

## 三、本地 vs 云端：行为对比

### 3.1 架构对比表

| 维度 | 本地模块 (utilActionsFor) | 云端模块 (actionsFor) |
|------|--------------------------|----------------------|
| 数据源 | `Vue.prototype.$util.send('appdb/...')` (Electron 主进程 SQLite) | `cloudClient[scope].list/upsert/delete/reorder` (REST API) |
| load | `appdb/{type}/find` 查询本地数据库 | `cli[scope].list()` 从云端 API 拉取 |
| poll | **空操作** (no-op) | `cli[scope].list()` + `replace` mutation |
| save | `appdb/{type}/save` 同步写入 | `cli[scope].upsert()` + 乐观更新 |
| remove | `appdb/{type}/remove` | `cli[scope].delete()` |
| mutation | `upsert` (直接覆盖) | `replace` (保护 pendingSaveIds) |
| reorder | 本地计算所有兄弟项顺序，逐个 `appdb/save` | `cli[scope].reorder()` 批量 API，返回所有受影响项 |

### 3.2 load 行为对比

**本地模块**（[DataModuleBase.ts#L136-L146](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L136-L146)）：

```ts
async load(context) {
  context.commit("error", null)
  await safely(context, async () => {
    const items = await Vue.prototype.$util.send(`appdb/${type}/find`, { options: loadOptions })
    if (context.rootState.workspaceId === LocalWorkspace.id) {
      context.commit('upsert', items)  // 直接 upsert，不保护
    }
  })
}
```

- 直接从本地 `appdb` 查询
- 数据只属于本地 workspace，不存在竞态问题
- 使用 `upsert` mutation（简单的 Object.assign 覆盖）

**云端模块**（[DataModuleBase.ts#L210-L220](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L210-L220)）：

```ts
async load(context) {
  context.commit("error", null)
  await safelyDo(context, async (cli) => {
    const items: any[] = await cli[scope].list()
    const rightItems = items.filter((i) => i.workspaceId === context.rootState.workspaceId)
    if (rightItems.length === items.length) {
      context.commit('replace', rightItems)  // 使用 replace，保护 pendingSaveIds
    }
  })
}
```

- 从云端 API `cli[scope].list()` 拉取
- 过滤出当前 workspace 的数据
- 使用 `replace` mutation（而非 `upsert`），保护有 pending save 的项目

### 3.3 poll 行为对比

**本地模块**（[DataModuleBase.ts#L147-L150](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L147-L150)）：

```ts
async poll() {
  // do nothing, locally we don't need to poll.
  // nothing else can change anything.
}
```

本地数据是单用户的，没有外部变更源，所以完全无需轮询。

**云端模块**（[DataModuleBase.ts#L221-L240](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L221-L240)）：

```ts
async poll(context) {
  await havingCli(context, async (cli) => {
    try {
      const items = await cli[scope].list()
      const rightItems = items.filter((item) => item.workspaceId === context.rootState.workspaceId)
      if (rightItems.length === items.length) {
        context.commit('replace', rightItems)   // 关键：使用 replace，会跳过 pendingSaveIds
      }
      context.commit('pollError', null)
    } catch (ex) {
      context.commit('pollError', ex)
    }
  })
}
```

云端数据可能被其他用户修改，需要定期轮询。使用 `replace` mutation 确保不会覆盖本地正在保存的乐观更新。

**特殊模块的 poll 覆盖**：

| 模块 | poll 行为 |
|------|-----------|
| CloudQueryFolderModule | 空操作（显式覆盖） |
| CloudConnectionFolderModule | 空操作（显式覆盖） |
| CloudUsedQueryModule | 空操作（显式覆盖） |
| CloudQueryModule | 继承 actionsFor 默认实现（执行轮询） |
| CloudConnectionModule | 继承 actionsFor 默认实现（执行轮询） |

文件夹和已使用记录不参与轮询，因为它们的变更频率低或不影响用户体验。

### 3.4 save 行为对比

**本地模块**（[DataModuleBase.ts#L163-L167](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L163-L167)）：

```ts
async save(context, item: T) {
  const updated = await Vue.prototype.$util.send(`appdb/${type}/save`, { obj: item })
  context.commit('upsert', updated)
  return updated.id
}
```

简单的保存后 upsert，无并发保护需求。

**云端模块**（[DataModuleBase.ts#L241-L247](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L241-L247)）：

```ts
async save(context, item: T): Promise<T> {
  return await havingCli(context, async (cli) => {
    const updated = await cli[scope].upsert(item)
    context.commit('upsert', updated)
    return updated.id
  })
}
```

云端 `save` 是悲观写入（先等 API 返回再更新）。而 `saveMany` 和 `reorder` 使用乐观更新策略（见下一节）。

### 3.5 reorder 行为对比

**本地模块**（以 [UtilityQueryModule.ts#L28-L96](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/UtilityQueryModule.ts#L28-L96) 为例）：

1. 计算目标文件夹内所有兄弟项的新顺序
2. 为每项分配连续的 `position` 值 (1, 2, 3, ...)
3. 乐观更新：`commit('upsert', updates)`
4. 逐个调用 `appdb/query/save` 保存
5. 失败时用快照回滚

```
本地 reorder 流程：
┌─────────────────────────────────────────────────┐
│ 1. 构建兄弟项排序数组 (newOrder)                  │
│ 2. 分配 position: idx + 1                        │
│ 3. 快照 pre-mutation 状态                         │
│ 4. commit('upsert', updates)  ← 乐观更新          │
│ 5. Promise.all(每个 item 单独 save)               │
│ 6. 成功：commit('upsert', saved)                  │
│    失败：commit('upsert', snapshot)  ← 回滚        │
└─────────────────────────────────────────────────┘
```

**云端模块**（以 [CloudQueryModule.ts#L44-L110](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/CloudQueryModule.ts#L44-L110) 为例）：

1. 计算乐观 position（使用小数如 `0.5` 表示"在 A 和 B 之间"）
2. `addPendingSave` 标记保护
3. 乐观更新
4. 调用 `cli[scope].reorder()` 批量 API
5. API 返回所有受影响项的新 position，逐项更新
6. 失败回滚，finally 中移除 pending 标记

```
云端 reorder 流程：
┌─────────────────────────────────────────────────────┐
│ 1. 计算 optimisticPosition (小数: 0.5 / 1.5 / ...)   │
│ 2. snapshot = { ...existing }                        │
│ 3. addPendingSave(item.id)  ← 保护标记               │
│ 4. commit('upsert', optimistic)  ← 乐观更新           │
│ 5. cli.queries.reorder(id, position, folderId)       │
│ 6. 成功：逐项 upsert 受影响项                         │
│    失败：commit('upsert', snapshot)  ← 回滚           │
│ 7. finally: removePendingSave(id)                    │
└─────────────────────────────────────────────────────┘
```

---

## 四、pendingSaveIds 保护机制详解

### 4.1 DataState 中的定义

[DataModuleBase.ts#L22-L29](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L22-L29)：

```ts
export interface DataState<T> {
  items: T[]
  loading: boolean
  error: ClientError
  pollError: ClientError
  filter?: string
  pendingSaveIds?: number[]   // ← 核心：正在保存的项目 ID 列表
}
```

### 4.2 相关 Mutations

[DataModuleBase.ts#L76-L86](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L76-L86)：

```ts
addPendingSave(state, id: number) {
  if (!state.pendingSaveIds) state.pendingSaveIds = []
  if (!state.pendingSaveIds.includes(id)) {
    state.pendingSaveIds.push(id)
  }
},
removePendingSave(state, id: number) {
  if (state.pendingSaveIds) {
    state.pendingSaveIds = state.pendingSaveIds.filter((i) => i !== id)
  }
}
```

- `addPendingSave`：添加 ID 到 pending 列表（去重）
- `removePendingSave`：从 pending 列表移除 ID

### 4.3 replace mutation：保护逻辑的核心

[DataModuleBase.ts#L101-L118](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts#L101-L118)：

```ts
replace(state, items: T[]) {
  const pendingIds = state.pendingSaveIds || []
  const itemIds = items.map((i) => i.id)
  const stateIds = state.items.map((i) => i.id)

  // 1. 只更新不在 pendingSaveIds 中的已有项
  const toUpdate = items.filter((i) =>
    stateIds.includes(i.id) && !pendingIds.includes(i.id)
  )

  // 2. 插入全新的项（不在 state 中的）
  const toInsert = items.filter((i) => !stateIds.includes(i.id))

  // 3. 只删除不在 pendingSaveIds 中的项
  const stateItems = _.reject(state.items, (item) =>
    !itemIds.includes(item.id) && !pendingIds.includes(item.id)
  )

  const upsertable = [...toUpdate, ...toInsert]
  upsertable.forEach((i) => upsert(stateItems, i))
  // ...排序
}
```

**三条保护规则：**

| 规则 | 逻辑 | 目的 |
|------|------|------|
| 不更新 pending 项 | `!pendingIds.includes(i.id)` | 保留本地乐观更新版本，不让云端旧数据覆盖 |
| 不删除 pending 项 | `!pendingIds.includes(item.id)` | 防止正在保存的项目被当作"已删除"而清掉 |
| 允许插入新项 | `!stateIds.includes(i.id)` | 其他人新建的项目可以正常添加 |

### 4.4 竞态场景分析

**场景：用户编辑一个查询，同时云端轮询返回**

```
时间线：
T1: 用户在 UI 中修改查询标题为 "新标题"
T2: dispatch('save', item)
    ├─ addPendingSave(item.id)
    └─ commit('upsert', item)  ← 乐观更新，state 中标题已更新为 "新标题"
T3: (同时) 定时器触发 poll()
    ├─ cli.queries.list() 返回旧数据（标题还是 "旧标题"）
    └─ commit('replace', oldItems)
        ├─ 检查 pendingSaveIds，发现 item.id 在其中
        └─ 跳过该项，保留本地乐观版本 ✓
T4: save API 返回
    ├─ commit('upsert', savedItem)
    └─ removePendingSave(item.id)
T5: 下次 poll 时，该 ID 不再在 pendingSaveIds 中，正常更新
```

如果没有 `pendingSaveIds` 保护，T3 时刻 `replace` 会用云端旧数据覆盖本地的乐观更新，造成用户看到的内容"跳回"旧版本。

### 4.5 saveMany 中的应用

[CloudQueryModule.ts#L29-L41](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/CloudQueryModule.ts#L29-L41)：

```ts
async saveMany(context, items: ISavedQuery[]) {
  // 标记所有项为 pending
  items.forEach(item => context.commit('addPendingSave', item.id))
  context.commit('upsert', items)   // 乐观更新
  try {
    return await havingCli(context, async (cli) => {
      const saved = await Promise.all(items.map(item => cli.queries.upsert(item)))
      context.commit('upsert', saved)
    })
  } finally {
    // 无论成功失败，都清除 pending 标记
    items.forEach(item => context.commit('removePendingSave', item.id))
  }
}
```

关键点：`finally` 块确保即使 API 失败，pending 标记也会被清除，避免永久"锁定"。

### 4.6 reorder 中的应用

[CloudQueryModule.ts#L69-L108](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/CloudQueryModule.ts#L69-L108)：

```ts
// 标记为 pending 以防止 poll 覆盖
context.commit('addPendingSave', item.id)

// 乐观提交
context.commit('upsert', { ...existing, position: optimisticPosition })

try {
  // 调用云端 reorder API
  const affectedItems = await cli.queries.reorder(...)
  // 更新受影响项
} catch (e) {
  // 回滚到快照
  context.commit('upsert', snapshot)
  throw e
} finally {
  // 清除 pending 标记
  context.commit('removePendingSave', item.id)
}
```

在 reorder 中，正在拖拽的项目被标记为 pending，防止轮询返回的旧位置数据覆盖乐观更新的临时位置。

---

## 五、为什么本地模块基本不 poll

### 5.1 核心原因

本地模块存储在 Electron 主进程的 SQLite 数据库（`appdb`）中，**只有当前应用实例能修改它**，不存在外部变更源。

- 没有多用户协作
- 没有其他设备修改
- 数据变更全部来自当前应用的用户操作

因此不需要轮询，所有变更都由本地操作直接触发。

### 5.2 具体体现

```ts
// DataModuleBase.ts 中 utilActionsFor 的 poll
async poll() {
  // do nothing, locally we don't need to poll.
  // nothing else can change anything.
}
```

即使 `DataManager` 的定时器仍然在调用 `dispatch('modulePath/poll')`，本地模块的 poll 是一个空操作，不会产生任何副作用。

### 5.3 对比：为什么云端必须 poll

云端模块连接的是远程 API：
1. 其他用户可能在另一个设备上修改了数据
2. 当前用户可能在网页端修改了数据
3. 管理员可能修改了团队共享的内容

这些变更不会主动推送到当前应用，所以需要定时轮询来获取最新状态。

---

## 六、数据流全景图

```
┌─────────────────────────────────────────────────────────────┐
│                       DataManager.vue                       │
│                                                             │
│   mounted() ──► mountAndRefresh() ──► 注册所有模块 + load    │
│                                                             │
│   interval: 每 N 秒                                         │
│   ┌──────────────────────────────────────────────────┐      │
│   │  DataModules.forEach:                            │      │
│   │    dispatch('data/queries/poll')                 │      │
│   │    dispatch('data/connections/poll')             │      │
│   │    dispatch('data/queryFolders/poll')  ── (空)   │      │
│   │    ...                                           │      │
│   └──────────────────────────────────────────────────┘      │
│                                                             │
│   watch: workspaceId ──► mountAndRefresh()                  │
│                    └─► unregisterModule + registerModule    │
└─────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴────────────────┐
          ▼                                ▼
   ┌──────────────┐                 ┌──────────────┐
   │  本地模块     │                 │  云端模块     │
   │              │                 │              │
   │ load:        │                 │ load:        │
   │  appdb/find  │                 │  cli.list()  │
   │  → upsert    │                 │  → replace   │
   │              │                 │              │
   │ poll: no-op  │                 │ poll:        │
   │              │                 │  cli.list()  │
   │              │                 │  → replace   │
   │              │                 │  (保护       │
   │              │                 │  pendingIds) │
   │              │                 │              │
   │ save:        │                 │ save:        │
   │  appdb/save  │                 │  cli.upsert()│
   │  → upsert    │                 │  → upsert    │
   │              │                 │              │
   │ reorder:     │                 │ reorder:     │
   │  本地计算    │                 │  cli.reorder │
   │  逐个save    │                 │  + pendingId │
   │  + 快照回滚  │                 │  + 快照回滚  │
   └──────────────┘                 └──────────────┘
```

---

## 七、文件索引

| 文件 | 作用 |
|------|------|
| [DataManager.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/data/DataManager.vue) | 数据管理层组件，注册/注销模块，轮询调度 |
| [DataModules.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/DataModules.ts) | 模块注册表，定义每个 path 的 local/cloud 版本 |
| [DataModuleBase.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/DataModuleBase.ts) | 基类：`utilActionsFor`（本地）、`actionsFor`（云端）、`replace` mutation、`pendingSaveIds` 逻辑 |
| [StoreHelpers.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/StoreHelpers.ts) | `havingCli`、`safelyDo`、`safely`、`upsert` 工具函数 |
| [UtilityConnectionModule.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/connection/UtilityConnectionModule.ts) | 本地连接模块，reorder 使用本地计算 |
| [CloudConnectionModule.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/connection/CloudConnectionModule.ts) | 云端连接模块，reorder 使用批量 API + pendingId |
| [UtilityQueryModule.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/UtilityQueryModule.ts) | 本地查询模块 |
| [CloudQueryModule.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/modules/data/query/CloudQueryModule.ts) | 云端查询模块 |
