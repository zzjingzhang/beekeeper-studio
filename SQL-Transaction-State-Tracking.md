# SQL 执行状态追踪分析报告

> 本文档针对包含 `BEGIN`、普通查询、`COMMIT` 的 SQL 语句，在**自动提交**和**手动提交**两种模式下，追踪 `CancelableQuery` 创建时机、`queries` Map 增删时机、`transactionTimeouts` 重置时机，并分析取消查询和错误处理时的状态一致性问题。

---

## 一、核心数据结构与关键文件

### 1.1 State 结构（后端 Utility 进程）

[handlerState.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts#L19-L47)

```typescript
class State {
  port: MessagePortMain = null
  server: IDbConnectionPublicServer = null
  connection: BasicDatabaseClient<any, any> = null
  transactionTimeouts: Map<number, NodeJS.Timeout> = new Map()
  queries: Map<string, CancelableQuery> = new Map()
  // ...
}
```

### 1.2 CancelableQuery 接口

[models.ts](file:///c:/Users/10244/Desktop/0508-under\beekeeper-studio/apps/studio/src/lib/db/models.ts#L306-L309)

```typescript
export interface CancelableQuery {
  execute: () => Promise<QueryResult>
  cancel: () => Promise<void>
}
```

### 1.3 关键文件索引

| 角色 | 文件 | 职责 |
|------|------|------|
| 前端调用入口 | [TabQueryEditor.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/TabQueryEditor.vue) | UI 层，触发查询执行 |
| 前端代理 | [ElectronUtilityConnectionClient.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/ElectronUtilityConnectionClient.ts) | 封装 IPC 调用，返回 CancelableQuery |
| 连接 Handler | [connHandlers.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts) | 处理 conn/query、事务相关请求 |
| 查询 Handler | [queryHandlers.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/queryHandlers.ts) | 处理 query/execute、query/cancel |
| PostgreSQL 实现 | [postgresql.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/clients/postgresql.ts) | 具体数据库的 CancelableQuery 创建逻辑 |

---

## 二、自动提交模式（Auto Commit）

自动提交是默认模式，每条 SQL 语句在执行后自动提交。

### 2.1 执行流程时序（以 BEGIN + SELECT + COMMIT 为例）

```
前端 (TabQueryEditor.vue)                          后端 (Utility 进程)
     │                                                    │
     │ 1. 用户点击 "Run" 按钮                            │
     │    this.connection.query(query, tabId, options, hasActiveTransaction=false)
     │                                                    │
     │───── IPC: conn/query ─────────────────────────────▶│
     │    { queryText, tabId, hasActiveTransaction: false }│
     │                                                    │
     │                                                    │ 2. connHandlers['conn/query']
     │                                                    │    - 调用 connection.query() 创建 CancelableQuery
     │                                                    │    - 生成 queryId (UUID)
     │                                                    │    - state.queries.set(queryId, query)  ← 加入 Map
     │                                                    │    - createOrResetTransactionTimeout(sId, tabId, true)
     │                                                    │      ↑ mustExist=true，若无则直接返回
     │                                                    │
     │◀──── 返回 queryId ─────────────────────────────────│
     │                                                    │
     │ 3. 调用 runningQuery.execute()                     │
     │                                                    │
     │───── IPC: query/execute ───────────────────────────▶│
     │    { queryId }                                      │
     │                                                    │
     │                                                    │ 4. queryHandlers['query/execute']
     │                                                    │    - state.queries.get(queryId) 获取 CancelableQuery
     │                                                    │    - 执行 query.execute()
     │                                                    │    - state.queries.delete(queryId)  ← 从 Map 移除
     │                                                    │
     │◀──── 返回结果 ─────────────────────────────────────│
     │                                                    │
     │ 5. 结果渲染，running = false                       │
     │                                                    │
```

### 2.2 关键点说明

**CancelableQuery 创建时机**：
- 在 `connHandlers['conn/query']` 中，通过 `state(sId).connection.query()` 创建
- 具体到 PostgreSQL，在 [postgresql.ts#L737-L797](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/clients/postgresql.ts#L737-L797) 创建，包含 `execute` 和 `cancel` 两个方法

**queries Map 增删时机**：
- **增**：`connHandlers['conn/query']` 中，创建后立即 `state(sId).queries.set(id, query)` [connHandlers.ts#L349](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts#L349)
- **删**：`queryHandlers['query/execute']` 执行完成后，`state(sId).queries.delete(queryId)` [queryHandlers.ts#L21](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/queryHandlers.ts#L21)

**transactionTimeouts 重置时机**：
- 自动提交模式下 `hasActiveTransaction = false`
- `createOrResetTransactionTimeout(sId, tabId, !hasActiveTransaction)` → `mustExist = true`
- 由于自动提交模式下不会预先设置 transactionTimeout，`mustExist=true` 时直接返回，不做任何操作 [connHandlers.ts#L640-L643](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts#L640-L643)

---

## 三、手动提交模式（Manual Commit）

手动提交模式下，用户需要显式执行 BEGIN、COMMIT 或 ROLLBACK。

### 3.1 完整事务生命周期

```
前端操作                          后端操作
  │                                 │
  │ 1. 切换到 Manual Commit 模式     │
  │    (isManualCommit = true)       │
  │                                 │
  │ 2. 点击 "Begin" 按钮            │
  │───── conn/startTransaction ────▶│
  │                                 │  - connection.reserveConnection(tabId)
  │                                 │  - connection.startTransaction(tabId) → 执行 BEGIN
  │                                 │  - createOrResetTransactionTimeout(sId, tabId)
  │                                 │    ↑ 首次设置 timeout
  │                                 │
  │◀──── 返回成功 ──────────────────│
  │    (hasActiveTransaction = true)│
  │                                 │
  │ 3. 执行普通查询                  │
  │───── conn/query ───────────────▶│
  │    hasActiveTransaction = true  │
  │                                 │  - 创建 CancelableQuery
  │                                 │  - state.queries.set(queryId, query)
  │                                 │  - createOrResetTransactionTimeout(sId, tabId, false)
  │                                 │    ↑ 重置 timeout（刷新倒计时）
  │                                 │
  │◀──── 返回 queryId ──────────────│
  │                                 │
  │───── query/execute ────────────▶│
  │                                 │  - 执行查询（使用预留连接）
  │                                 │  - state.queries.delete(queryId)
  │                                 │
  │◀──── 返回结果 ──────────────────│
  │                                 │
  │ 4. 点击 "Commit" 按钮           │
  │───── conn/commitTransaction ───▶│
  │                                 │  - connection.commitTransaction(tabId) → 执行 COMMIT
  │                                 │  - clearTransactionTimeout(sId, tabId)
  │                                 │    ↑ 清除 timeout
  │                                 │  - connection.releaseConnection(tabId)
  │                                 │
  │◀──── 返回成功 ──────────────────│
  │    (hasActiveTransaction = false)
```

### 3.2 关键点说明

**transactionTimeouts 设置与重置**：

| 动作 | 调用函数 | 效果 |
|------|----------|------|
| startTransaction | `createOrResetTransactionTimeout(sId, tabId)` | 首次创建 timeout，存入 Map |
| 每次查询执行 | `createOrResetTransactionTimeout(sId, tabId, false)` | 清除旧 timeout，创建新 timeout（刷新倒计时） |
| commitTransaction | `clearTransactionTimeout(sId, tabId)` | 清除 timeout，从 Map 删除 |
| rollbackTransaction | `clearTransactionTimeout(sId, tabId)` | 清除 timeout，从 Map 删除 |
| resetTransactionTimeout | `createOrResetTransactionTimeout(sId, tabId, true)` | 仅在已存在时重置 |

**timeout 内部逻辑** [connHandlers.ts#L640-L668](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts#L640-L668)：
- 第一阶段 timeout：触发 `transactionTimeoutWarning/${tabId}` 事件（警告用户）
- 第二阶段 timeout：自动执行 `rollbackTransaction`，触发 `transactionTimedOut/${tabId}` 事件

**queries Map 增删时机**（与自动提交相同）：
- **增**：`conn/query` handler 中
- **删**：`query/execute` 完成后或 `query/cancel` 完成后

---

## 四、取消查询（Cancel Query）分析

### 4.1 取消流程

[queryHandlers.ts#L33-L42](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/queryHandlers.ts#L33-L42)

```typescript
'query/cancel': async function({ queryId, sId }) {
  checkConnection(sId)
  const query = state(sId).queries.get(queryId)
  if (!query) {
    throw new Error(errorMessages.noQuery)
  }
  await query.cancel()
  state(sId).queries.delete(queryId)
}
```

### 4.2 后端 query 对象清理分析

**结论：取消查询时，后端 query 对象**一定会被清理**。

清理路径：
1. `query.cancel()` 被调用（以 PostgreSQL 为例，调用 `pg_cancel_backend(pid)`）
2. `state(sId).queries.delete(queryId)` 从 Map 中移除
3. 前端的 `runningQuery` 对象虽然还存在，但后续无法再通过它调用后端（因为 queryId 已失效）

**PostgreSQL 具体取消实现** [postgresql.ts#L775-L795](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/clients/postgresql.ts#L775-L795)：
```typescript
cancel: async (): Promise<void> => {
  if (!pid) {
    throw new Error('Query not ready to be canceled')
  }
  canceling = true
  try {
    const data = await this.driverExecuteSingle(`SELECT pg_cancel_backend(${pid});`, { tabId })
    // ...
    cancelable.cancel()
  } catch (err) {
    canceling = false
    throw err
  }
}
```

---

## 五、错误处理与状态一致性分析

### 5.1 可能出现状态不一致的场景

**场景 1：query/execute 执行抛出错误**

[queryHandlers.ts#L13-L32](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/queryHandlers.ts#L13-L32)

```typescript
'query/execute': async function({ queryId, sId }) {
  checkConnection(sId)
  const query = state(sId).queries.get(queryId)
  if (!query) {
    throw new Error(errorMessages.noQuery)
  }

  const result = await query.execute()  // ← 如果这里抛出错误
  state(sId).queries.delete(queryId)   // ← 这行不会执行！
  // ...
}
```

**问题**：如果 `query.execute()` 抛出错误，`state(sId).queries.delete(queryId)` 不会执行，导致 `queries` Map 中残留失效的 query 对象。

**状态不一致表现**：
- **后端（Utility）**：`queries` Map 中仍有该 queryId 对应的 CancelableQuery
- **前端（UI）**：查询已失败，`running` 状态重置为 `false`
- **后续影响**：如果前端再次尝试取消（虽然 UI 上可能不允许），会找到残留的 query 对象，但实际上该 query 已完成（失败）

**场景 2：conn/query 成功但 query/execute 请求丢失**

如果网络或 IPC 出现问题，`conn/query` 成功创建了 CancelableQuery 并加入 Map，但 `query/execute` 请求从未送达或失败，导致：
- 后端：query 对象永久留在 Map 中（内存泄漏）
- 前端：可能处于 loading 状态，最终超时或用户取消

**场景 3：事务 timeout 自动回滚时**

[connHandlers.ts#L654-L662](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts#L654-L662)

```typescript
const warningWindowTimeout = setTimeout(async () => {
  checkConnection(sId)
  await state(sId).connection.rollbackTransaction(tabId)  // ← 后端执行回滚
  clearTransactionTimeout(sId, tabId)
  state(sId).port.postMessage({
    type: `transactionTimedOut/${tabId}`                   // ← 通知前端
  })
}, ...)
```

**潜在问题**：如果前端收到 `transactionTimedOut` 事件前，用户正在执行新的查询，可能出现：
- 后端：连接已回滚并释放
- 前端：仍认为事务处于活跃状态

### 5.2 已有的保护机制

1. **conn/query 中的 mustExist 检查**：防止在非事务状态下错误设置 timeout
2. **clearTransactionTimeout 的幂等性**：多次调用不会出错
3. **query/cancel 的错误抛出**：如果 query 不存在，会明确抛出错误

### 5.3 潜在风险总结

| 场景 | UI 状态 | Utility 状态 | 一致性 |
|------|---------|-------------|--------|
| query.execute() 抛错 | running=false，显示错误 | queries Map 中残留 queryId | ❌ 不一致 |
| query/execute 请求丢失 | running=true（超时后重置） | queries Map 中残留 queryId | ❌ 不一致 |
| 事务自动超时回滚 | hasActiveTransaction=true（收到事件前） | 事务已回滚，连接已释放 | ⚠️ 短暂不一致 |
| 正常执行完成 | running=false，显示结果 | queries Map 已清理 | ✅ 一致 |
| 正常取消 | running=false | queries Map 已清理 | ✅ 一致 |

---

## 六、总结问答

### Q1: CancelableQuery 何时创建？
**A**: 在 `conn/query` handler 中，调用 `connection.query()` 时创建。具体在各数据库客户端（如 PostgresClient）的 `query()` 方法中实现。

### Q2: queries Map 何时增删？
**A**:
- **增**：`conn/query` handler 中，创建 CancelableQuery 后立即 `set(queryId, query)`
- **删**：`query/execute` 成功完成后 `delete(queryId)`，或 `query/cancel` 完成后 `delete(queryId)`
- **注意**：`query/execute` 抛出错误时不会删除，可能造成残留

### Q3: transactionTimeouts 何时重置？
**A**:
- 首次设置：`startTransaction` 时
- 重置（刷新）：每次 `conn/query` 且 `hasActiveTransaction=true` 时
- 手动重置：`conn/resetTransactionTimeout` 时
- 清除：`commitTransaction` 或 `rollbackTransaction` 时

### Q4: 取消查询时后端 query 对象是否一定被清理？
**A**: **是的**。`query/cancel` handler 中，无论 `query.cancel()` 成功与否（只要 query 存在），都会执行 `state(sId).queries.delete(queryId)`。

### Q5: 执行抛错时 UI 与 utility 状态是否可能不一致？
**A**: **可能**。主要场景：
1. `query.execute()` 抛出错误时，`queries` Map 不会清理，造成残留
2. 事务自动超时回滚时，前端收到事件前存在短暂不一致
3. IPC 消息丢失可能导致前后端状态错位
