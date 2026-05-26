# Tab 生命周期流程分析

## 流程概述

本文档分析以下完整流程的内部机制：
```
打开 table tab → 切换为 active → 关闭 → 重新打开最近关闭 tab → 再次从 sidebar 打开同一张表
```

## 核心概念与数据结构

### 1. 关键模型字段

**数据库层 (OpenTab Entity)** - [OpenTab.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\common\appdb\models\OpenTab.ts)

| 字段 | 类型 | 作用 |
|------|------|------|
| `active` | boolean | 标记当前是否为活动tab |
| `lastActive` | Date | 最后一次被激活的时间戳 |
| `deletedAt` | Date (DeleteDateColumn) | TypeORM软删除标记，null表示未删除 |
| `tableName` | string | 表名（table类型tab） |
| `schemaName` | string | schema名（table类型tab） |
| `entityType` | string | 实体类型（table/view等） |
| `tabType` | TabType | tab类型（'table'/'query'等） |

**传输层 (TransportOpenTab)** - [TransportOpenTab.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\common\transport\TransportOpenTab.ts)

与数据库层字段基本一致，但增加了前端运行时字段 `isRunning`、`isTransaction`。

### 2. 内存状态管理

**TabModule State** - [TabModule.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\store\modules\TabModule.ts)

```typescript
interface State {
  tabs: TransportOpenTab[],        // 当前打开的tabs
  active?: TransportOpenTab,       // 当前活动tab
  lastClosedTabs: TransportOpenTab[] // 内存中最近关闭的tabs栈
}
```

---

## 逐步流程分析

### 阶段 1：打开 Table Tab

**触发点**：[CoreTabs.vue](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\components\CoreTabs.vue#L1015-L1033) `openTable()` 方法

```typescript
async openTable({ table, filters }) {
  let tab = {} as TransportOpenTab;
  tab.tabType = 'table';
  tab.title = table.name;
  tab.tableName = table.name;
  tab.schemaName = table.schema;
  tab.entityType = table.entityType;
  
  // 关键：先检查是否已有匹配的tab
  let existing = this.tabItems.find((t) => matches(t, tab));
  if (existing) {
    // 已存在，激活即可
    await this.$store.dispatch('tabs/setActive', existing);
  } else {
    // 不存在，新建
    await this.addTab(tab);
  }
}
```

**状态变化**：
- 内存 `tabs` 数组新增一个tab
- 数据库插入新记录：`active=true`, `deletedAt=null`, `lastActive=当前时间`

---

### 阶段 2：切换为 Active

**触发点**：[TabModule.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\store\modules\TabModule.ts#L249-L260) `setActive` action

```typescript
async setActive(context, tab: TransportOpenTab) {
  const oldActive = context.state.active;
  context.commit('setActive', tab);
  
  if (oldActive) {
    oldActive.active = false;
  }
  tab.active = true;
  tab.lastActive = new Date();  // 更新最后活动时间
  tab.deletedAt = null;         // 确保deletedAt为null
  
  await context.dispatch('save', [tab, oldActive].filter(x => !!x));
}
```

**状态变化**：
- 新激活tab：`active=true`, `lastActive=当前时间`
- 原活动tab：`active=false`
- 数据库更新两条记录

---

### 阶段 3：关闭 Tab

**触发点**：[CoreTabs.vue](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\components\CoreTabs.vue#L1060-L1089) `close()` → [TabModule.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\store\modules\TabModule.ts#L218-L228) `remove` action

```typescript
// TabModule remove action
async remove(context, rawItems) {
  const items = _.isArray(rawItems) ? rawItems : [rawItems];
  items.forEach((tab) => {
    tab.deletedAt = new Date();  // 设置软删除时间
    tab.position = 99;
    context.commit('remove', tab);  // 从tabs移除，加入lastClosedTabs
  });
  
  // 保存到数据库（软删除）
  if (usedConfig?.id) {
    await Vue.prototype.$util.send('appdb/tabs/save', { obj: items });
  }
}

// TabModule remove mutation
remove(state, tabs) {
  if (!_.isArray(tabs)) tabs = [tabs];
  state.tabs = _.without(state.tabs, ...tabs);
  // 关键：复制一份加入lastClosedTabs栈
  state.lastClosedTabs.push(...tabs.map((tab) => duplicate(tab)));
}
```

**状态变化**：
- 内存 `tabs` 数组移除该tab
- 内存 `lastClosedTabs` 数组 **push** 该tab的副本（使用 `duplicate()`）
- 数据库记录：`deletedAt=当前时间`, `position=99`（软删除，记录仍保留）

---

### 阶段 4：重新打开最近关闭 Tab

**触发点**：[TabModule.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\store\modules\TabModule.ts#L179-L192) `reopenLastClosedTab` action

```typescript
async reopenLastClosedTab(context) {
  const { usedConfig, workspaceId } = context.rootState;
  
  // 从数据库获取最近软删除的tab
  const tab = await Vue.prototype.$util.send(
    'appdb/tabhistory/getLastDeletedTab', 
    { workspaceId, connectionId: usedConfig.id }
  );
  
  if (tab) {
    tab.deletedAt = null;  // 清除删除标记
    await context.dispatch('add', { item: tab });  // 重新添加
    await context.dispatch('setActive', tab);
  }
}
```

**数据库查询** - [OpenTab.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\common\appdb\models\OpenTab.ts#L219-L232) `getClosedHistory()`

```typescript
static async getClosedHistory(connectionIds): Promise<TransportOpenTab> {
  return await this.findOne({
    where: {
      deletedAt: Not(IsNull()),  // 查找已软删除的
      connectionId,
      workspaceId
    },
    order: {
      deletedAt: 'DESC'  // 按删除时间倒序，取最新的
    },
    withDeleted: true  // 包含软删除记录
  });
}
```

**状态变化**：
- 数据库记录：`deletedAt=null`（恢复），`active=true`
- 内存 `tabs` 重新加入该tab
- 内存 `lastClosedTabs` **不直接操作**，但在 `add` mutation 中会检查并移除匹配项

**关键点**：[TabModule.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\store\modules\TabModule.ts#L84-L91) `add` mutation

```typescript
add(state, nu: TransportOpenTab) {
  state.tabs.push(nu);
  
  // 关键去重逻辑：如果新tab在lastClosedTabs中有匹配项，移除之
  const existingTabInClosedTabs = state.lastClosedTabs.find((tab) => matches(tab, nu));
  if (existingTabInClosedTabs) {
    state.lastClosedTabs = _.without(state.lastClosedTabs, existingTabInClosedTabs);
  }
}
```

---

### 阶段 5：再次从 Sidebar 打开同一张表

**触发点**：[CoreTabs.vue](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\components\CoreTabs.vue#L1015-L1033) `openTable()` 方法

```typescript
async openTable({ table, filters }) {
  let tab = {} as TransportOpenTab;
  tab.tabType = 'table';
  tab.tableName = table.name;
  tab.schemaName = table.schema;
  tab.entityType = table.entityType;
  
  // 使用matches()检查是否已有匹配的tab
  let existing = this.tabItems.find((t) => matches(t, tab));
  
  if (existing) {
    // 找到匹配的，直接激活（不会新建）
    await this.$store.dispatch('tabs/setActive', existing);
  } else {
    await this.addTab(tab);
  }
}
```

**状态变化**：
- 通过 `matches()` 找到已恢复的tab，直接激活
- 不会创建新的tab记录
- 数据库仅更新 `active` 和 `lastActive` 字段

---

## matches() 去重逻辑深度解析

### 核心匹配规则

**对于 `table` 类型 tab**：

```typescript
// 匹配条件（需同时满足）
1. tableName 相等
2. schemaName 相等（都为null也算相等）
3. entityType 相等（都为null也算相等）
```

**注意**：`table` 类型匹配 **不比较 tabType**，而 `table-properties` 类型匹配 **需要比较 tabType**。

### Transport 层 vs TypeORM Model 层 matches() 差异

#### TypeORM Model 层 - [OpenTab.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\common\appdb\models\OpenTab.ts#L157-L190)

```typescript
matches(other: OpenTab): boolean {
  if (other.workspaceId && this.workspaceId && this.workspaceId !== other.workspaceId) {
    return false;
  }

  switch (other.tabType) {
    case 'table-properties':
      return this.tableName === other.tableName &&
        (this.schemaName || null) === (other.schemaName || null) &&
        (this.entityType || null) === (other.entityType || null) &&
        (this.tabType || null) === (other.tabType || null);  // 需要匹配tabType
    case 'table':
      return this.tableName === other.tableName &&
        (this.schemaName || null) === (other.schemaName || null) &&
        (this.entityType || null) === (other.entityType || null);  // 不匹配tabType
    case 'query':
      return this.queryId === other.queryId;  // 仅匹配queryId
    // ...其他类型
  }
}
```

#### Transport 层 - [TransportOpenTab.ts](file:///c:\Users\10244\Desktop\0508-under\beekeeper-studio\apps\studio\src\common\transport\TransportOpenTab.ts#L154-L187)

```typescript
export function matches(obj: TransportOpenTab, other: TransportOpenTab): boolean {
  if (other.workspaceId && obj.workspaceId && obj.workspaceId !== other.workspaceId) {
    return false;
  }

  switch (other.tabType) {
    case 'table-properties':
      return obj.tableName === other.tableName &&
        (obj.schemaName || null) === (other.schemaName || null) &&
        (obj.entityType || null) === (other.entityType || null) &&
        (obj.tabType || null) === (other.tabType || null);
    case 'table':
      return obj.tableName === other.tableName &&
        (obj.schemaName || null) === (other.schemaName || null) &&
        (obj.entityType || null) === (other.entityType || null);
    case 'query':
      // 差异点：多了 usedQueryId 匹配逻辑
      return (obj.queryId === other.queryId && obj.queryId !== null && other.queryId !== null) ||
        (obj.usedQueryId === other.usedQueryId && obj.usedQueryId !== null && other.queryId !== null);
    // ...其他类型
  }
}
```

### 关键差异对比表

| 差异点 | TypeORM Model 层 | Transport 层 |
|--------|-----------------|-------------|
| 实现方式 | 类实例方法 `this.matches(other)` | 独立函数 `matches(obj, other)` |
| query 类型匹配 | 仅比较 `queryId` | 比较 `queryId` **或** `usedQueryId` |
| 使用场景 | 数据库实体操作时 | 前端store、组件中匹配时 |

**重要说明**：虽然代码中存在这两个版本的 `matches()`，但在实际流程中：
- 前端组件（如 CoreTabs.vue）和 TabModule 都 **使用 Transport 层的 matches()**
- TypeORM Model 层的 matches() 目前在代码中 **没有被直接调用**

---

## 完整流程状态流转图

```
初始状态：
  内存: tabs=[], active=null, lastClosedTabs=[]
  数据库: 无相关记录

┌─────────────────────────────────────────────────────────────┐
│ 阶段1：打开 Table Tab (users 表)                             │
├─────────────────────────────────────────────────────────────┤
│ 内存: tabs=[tab1], active=tab1, lastClosedTabs=[]           │
│ 数据库: tab1 {                                               │
│   id: 1, tableName: 'users', active: true,                   │
│   lastActive: T1, deletedAt: null                            │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 阶段2：切换为 Active (点击tab)                               │
├─────────────────────────────────────────────────────────────┤
│ 内存: tabs=[tab1], active=tab1, lastClosedTabs=[]           │
│ 数据库: tab1 { lastActive: T2 (更新) }                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 阶段3：关闭 Tab                                              │
├─────────────────────────────────────────────────────────────┤
│ 内存: tabs=[], active=null, lastClosedTabs=[tab1_copy]      │
│ 数据库: tab1 {                                               │
│   active: false, deletedAt: T3, position: 99                 │
│ }                                                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 阶段4：重新打开最近关闭 Tab                                  │
├─────────────────────────────────────────────────────────────┤
│ 1. 数据库查询: getLastDeletedTab() → 返回 tab1              │
│ 2. tab1.deletedAt = null                                     │
│ 3. add action:                                               │
│    - 数据库: 保存 tab1 (deletedAt=null, active=true)        │
│    - add mutation: tabs.push(tab1)                           │
│    - 去重检查: lastClosedTabs 中找到匹配的 tab1_copy，移除   │
│ 4. setActive: 更新 lastActive=T4                             │
│                                                              │
│ 最终内存: tabs=[tab1], active=tab1, lastClosedTabs=[]       │
│ 数据库: tab1 { deletedAt: null, active: true, lastActive:T4 }│
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 阶段5：从 Sidebar 再次打开 users 表                          │
├─────────────────────────────────────────────────────────────┤
│ 1. openTable() 创建临时 tab: {tableName:'users', ...}       │
│ 2. matches(tab1, temp_tab) → true (匹配成功)                 │
│ 3. 直接 setActive(tab1)，不新建                              │
│                                                              │
│ 内存: tabs=[tab1], active=tab1 (无变化)                      │
│ 数据库: tab1 { lastActive: T5 (仅更新时间) }                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 关键交互机制总结

### 1. lastClosedTabs 与数据库的协同

- **内存 `lastClosedTabs`**：作为快速访问的"最近关闭"栈，用于UI显示和快速匹配
- **数据库 `deletedAt`**：持久化存储关闭历史，支持跨会话恢复
- 恢复时 **以数据库为准**（`getLastDeletedTab` 查数据库），内存栈仅作为辅助去重

### 2. 去重逻辑的三层防护

1. **打开时去重**：`openTable()` 中先查 `tabItems` 有没有匹配的
2. **添加时去重**：`add` mutation 中从 `lastClosedTabs` 移除匹配项
3. **数据库恢复**：通过 `deletedAt=null` 恢复原有记录，而非新建

### 3. 软删除设计的优势

- 保留tab的完整状态（filters、context等）
- 支持"撤销关闭"功能
- 历史记录可追溯（`getHistory` 方法）
- 定期清理（`clearOldDeletedTabs` 清理7天前的记录）

---

## 代码调用链路总结

```
打开Table Tab:
  CoreTabs.openTable() 
    → matches(transport) 检查重复
    → TabModule.add() 
      → 数据库 save (active=true, deletedAt=null)
      → add mutation (tabs.push, 清理lastClosedTabs中匹配项)
    → TabModule.setActive() 
      → 更新 lastActive

关闭Tab:
  CoreTabs.close()
    → TabModule.remove()
      → 设置 deletedAt = new Date()
      → remove mutation (tabs.pop, lastClosedTabs.push(duplicate))
      → 数据库 save (软删除)

恢复最近关闭:
  CoreTabs.reopenLastClosedTab()
    → TabModule.reopenLastClosedTab()
      → 数据库 getLastDeletedTab (查询deletedAt最新的)
      → 设置 deletedAt = null
      → TabModule.add() (流程同上)
      → TabModule.setActive()
```
