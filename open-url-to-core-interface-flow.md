# open-url / open-file → CoreInterface 完整流程说明

本文档描述 Beekeeper Studio 从 `postgres://...` 连接串（或 SQLite/DuckDB 文件路径）启动，到最终切换到 `CoreInterface` 页面所经历的完整流程。

涉及的核心模块：

- 渲染进程入口：[App.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/App.vue)
- 连接入口页：[ConnectionInterface.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/ConnectionInterface.vue)
- 核心工作页：[CoreInterface.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/CoreInterface.vue)
- Vuex Store（`openUrl` / `connect` action）：[store/index.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts)
- URL 解析处理器：[handlers/appDbHandlers.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/appDbHandlers.ts)
- URL 解析实现（文件识别 + 连接串解析）：[common/appdb/models/saved_connection.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/common/appdb/models/saved_connection.ts)
- 连接建立后端处理器：[src-commercial/backend/handlers/connHandlers.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts)
- PIN 解锁插件：[plugins/BeekeeperPlugin.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/plugins/BeekeeperPlugin.ts)
- PIN 模型：[common/appdb/models/UserPin.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/common/appdb/models/UserPin.ts)
- CockroachDB 客户端：[lib/db/clients/cockroach.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/clients/cockroach.ts)
- Azure 认证服务：[lib/db/authentication/azure.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/authentication/azure.ts)
- SurrealDB 客户端：[src-commercial/backend/lib/db/clients/surrealdb.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/db/clients/surrealdb.ts)

---

## 1. 两种入口

### 1.1 启动参数（`open-url` / `open-file`）

Electron 主进程在处理系统 `open-url` / `open-file` 时，最终把 URL 作为 `?url=<encoded>` 查询参数传给加载页，因此渲染进程读取的方式统一为：

```ts
// App.vue (L170-L174)
const query = querystring.parse(window.location.search, { parseBooleans: true })
if (query) {
  this.url = query.url || null
}
```

> 说明：`postgres://`、`mysql://`、`cockroachdb://` 等 URL 经过编码后通过 query string 注入。文件路径（例如 SQLite 的 `C:\data\foo.db`、DuckDB 的 `*.duckdb`）同样以 `url=` 的形式注入，后续由 `parseUrl` 根据后缀识别。

### 1.2 ConnectionInterface 内部拖放

在 `ConnectionInterface` 页面，SQLite/DuckDB 文件也可以通过拖放加载：[ConnectionInterface.vue#L434-L447](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/ConnectionInterface.vue#L434-L447)

```ts
async maybeLoadSqlite({ files }) {
  const conf = await this.$util.send('appdb/saved/parseUrl', { url: file.path })
  this.config = conf
  this.submit()
}
```

这里直接调用了同一个 `parseUrl` handler，然后走 `submit → unlock → connect` 路径。

---

## 2. 页面切换的闸门：`connected` 状态

[App.vue#L11-L15](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/App.vue#L11-L15)

```html
<connection-interface v-if="!connected" />
<core-interface v-else />
```

- `connected` 是 Vuex root state 字段（初始 `false`）。
- 只有当 `connect` action 成功走完后才会 `commit('connected', true)`，此时 `ConnectionInterface` 卸载、`CoreInterface` 挂载。
- 任何一个前置步骤失败（PIN 取消、JWT 取消、解析失败、`conn/create` 抛错等）都会让 `connected` 保持 `false`，页面停留在 `ConnectionInterface`。

---

## 3. `openUrl` action 主流程

[store/index.ts#L464-L467](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L464-L467)

```ts
async openUrl(context, { url, auth }) {
  const conn = await Vue.prototype.$util.send('appdb/saved/parseUrl', { url })
  await context.dispatch('connect', { config: conn, auth })
}
```

执行顺序：`parseUrl` → `connect`。前者纯同步解析；后者才会触发 Cockroach JWT、PIN、Azure、SurrealDB namespace 等认证/配置分支。

---

## 4. `parseUrl` 做了什么

[handlers/appDbHandlers.ts#L180-L186](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/appDbHandlers.ts#L180-L186)

```ts
'appdb/saved/parseUrl': async function({ url }) {
  const conn = new SavedConnection()
  if (!conn.parse(url)) throw `Unable to parse ${url}`
  return defaultTransform(conn, SavedConnection)
}
```

解析实现在 [saved_connection.ts#L365-L453](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/common/appdb/models/saved_connection.ts#L365-L453)：

1. **文件路径识别**：如果 `url` 不含 `://`，按后缀判定：
   - `.db` / `.sqlite` / `.sqlite3` → `connectionType = 'sqlite'`，`defaultDatabase = url`
   - `.duckdb` / `.ddb` → `connectionType = 'duckdb'`，`defaultDatabase = url`
   - 均不命中 → 返回 `false`，外层抛出 `Unable to parse ...`

2. **URL 解析**（`connection-string` 库 + 自定义规则）：
   - 从 `user:pass@host:port/db` 提取 `username/password/host/port/defaultDatabase`
   - `protocol` 决定 `connectionType`，并有别名映射：`postgres/psql → postgresql`，`mssql → sqlserver`
   - 如果 hostname 含 `redshift.amazonaws.com`，覆盖为 `redshift`
   - 如果 hostname 含 `cockroachlabs.cloud`、或 URL 带 `--crdb:jwt_auth_enabled=true`、`--cluster=...` 参数、或协议是 `cockroach/cockroachdb`，则覆盖为 `cockroachdb`，并在 `options` 上写 `cluster`、`jwtAuthEnabled`
   - 根据 `sslmode`、`https://` 前缀或 `TrustServerCertificate` 等设置 SSL 相关字段

3. `parseUrl` **只负责把 URL 填到连接配置对象里**，不会做任何网络连接或密钥交互。

**失败路径**：`parse()` 返回 `false` → 抛 `Unable to parse ...` → `openUrl` 未进入 `connect`，`App.vue` 的 `catch` 会 `$noty.error` 并 `throw`，`connected` 仍为 `false`，停留在 `ConnectionInterface`。

---

## 5. `connect` action 的完整时序

[store/index.ts#L491-L532](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L491-L532)

```ts
async connect(context, { config, auth }) {
  const resolvedConfig = await resolveJwtConfig(config)   // ① Cockroach JWT
  if (!resolvedConfig) return false

  await Vue.prototype.$util.send('conn/create', {          // ② 真正建连（后端再次校验 PIN + 分配 Azure authId）
    config: resolvedConfig, auth, osUser: context.state.username
  })

  const defaultSchema      = await connection.defaultSchema()
  const supportedFeatures  = await connection.supportedFeatures()
  const versionString      = await connection.versionString()

  context.commit('connectionType', config.connectionType)
  context.commit('connected', true)                       // ③ 页面切换为 CoreInterface
  context.commit('supportedFeatures', supportedFeatures)
  context.commit('versionString', versionString)
  context.commit('newConnection', config)                 // ④ 写入 usedConfig

  if (config.connectionType === 'surrealdb' &&
      config.surrealDbOptions?.authType === SurrealAuthType.Root) {
    await context.dispatch('updateNamespaceList')         // ⑤ SurrealDB namespace
  }
  await context.dispatch('updateDatabaseList')
  await context.dispatch('updateTables')
  await context.dispatch('updateRoutines')
  ...
}
```

下面按发生顺序展开每一步。

### 5.1 ① Cockroach JWT 令牌补齐（`resolveJwtConfig`）

[store/index.ts#L52-L75](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L52-L75)

```ts
const shouldPromptForCockroachJwt = (config) =>
  config?.connectionType === 'cockroachdb' &&
  !!config?.options?.jwtAuthEnabled &&
  !config?.password

const resolveJwtConfig = async (config) => {
  if (!shouldPromptForCockroachJwt(config)) return config

  const { token, cancelled } = await BeekeeperPlugin.promptJwtToken(
    BeekeeperPlugin.buildConnectionName(config)
  )
  if (cancelled) return null
  config.password = token
  return config
}
```

只有当连接是 CockroachDB、开启了 `jwtAuthEnabled`、且 `password` 为空时，才会通过 `InputJwtModal` 让用户粘贴 JWT，并把它作为 `password` 写入 config。

**失败路径**：用户点 Cancel → 返回 `null` → `connect` 直接 `return false`，`connected` 仍为 `false`。

在真正与服务器通信时，Cockroach 客户端会把 `password` 再拼进 libpq 选项串：[cockroach.ts#L179-L199](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/clients/cockroach.ts#L179-L199)

```ts
const password = server.config.options?.jwtAuthEnabled
  ? server.config.password?.replace(/\s+/g, '')
  : server.config.password
...
if (server.config.options?.jwtAuthEnabled) {
  optionsParts.push('--crdb:jwt_auth_enabled=true')
}
```

所以 JWT 必须发生在 `conn/create` 之前，否则 `password` 为空连接必然失败。

### 5.2 ② `conn/create`：PIN 校验 + Azure authId 分配 + 物理建连

后端入口：[connHandlers.ts#L135-L192](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/handlers/connHandlers.ts#L135-L192)

顺序严格如下：

1. **PIN 校验**（仅当 `bksConfig.security.lockMode === 'pin'`）

   ```ts
   if (bksConfig.security.lockMode === "pin") {
     await waitPromise(1000)
     if (!auth) throw `Authentication is required.`
     if (auth.mode !== "pin") throw `Invalid authentication mode: ${auth.mode}`
     if (!await UserPin.verifyPin(auth.input))
       throw `Incorrect pin. Please try again.`
   }
   ```

   `auth` 来自渲染进程的 `$bks.unlock()`，调用位置有两处：
   - URL 启动：[App.vue#L184-L194](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/App.vue#L184-L194)
   - 手动连接：[ConnectionInterface.vue#L504-L506](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/ConnectionInterface.vue#L504-L506)

   前端解锁逻辑：[BeekeeperPlugin.ts#L125-L134](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/plugins/BeekeeperPlugin.ts#L125-L134)。它只是弹窗、收集 PIN、返回 `{ auth: { input, mode: 'pin' } }` 或 `{ cancelled: true }`。真正的 hash 比对在后端 `conn/create` 里做（`bcrypt.compare`，见 [UserPin.ts#L53-L61](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/common/appdb/models/UserPin.ts#L53-L61)）。

   **失败路径**：
   - 前端取消 → `cancelled` → `connect` 不被调用 → `connected` 保持 `false`
   - 前端未配置 lockMode（`lockMode !== 'pin'`）→ 后端直接跳过 PIN 校验
   - 后端 PIN 错误 → `throw 'Incorrect pin...'` → `connect` catch → `connected` 仍 `false`

2. **Azure `authId` 自动分配**

   ```ts
   if (config.azureAuthOptions?.azureAuthEnabled && !config.authId) {
     let cache = new TokenCache()
     cache = await cache.save()
     config.authId = cache.id
     if (config.id) {
       const conn = await SavedConnection.findOneBy({ id: config.id })
       conn.authId = cache.id
       conn.save()
     }
   }
   ```

   这段在后端完成，保证：
   - 只要启用了 Azure 认证，就一定有一个 TokenCache 行；
   - 对于已保存的连接，把 `authId` 回写到 `saved_connection` 表。

   对应前端的 `save()` 也会分配（[ConnectionInterface.vue#L551-L554](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/ConnectionInterface.vue#L551-L554)），`conn/create` 这里只是兜底。

   Azure `authId` 的使用方是 [AzureAuthService.init](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/authentication/azure.ts#L77-L95)，它从 `TokenCache` 里取出 msal 的 tokenCache 序列化字符串。`authId` 必须在真正建连之前分配，否则 `AzureAuthService` 会抛 `No TokenCache entry found in database`。

3. **SurrealDB namespace 组合默认数据库**

   ```ts
   let database = config.defaultDatabase
   if (config.connectionType === 'surrealdb'
       && config?.surrealDbOptions?.namespace && database) {
     database = `${config?.surrealDbOptions?.namespace}::${database}`
   }
   ```

   说明：`surrealDbOptions.namespace` 在 `parseUrl` 中**不会**被自动填入（它只在 `newConnection` mutation 时从 `config.surrealDbOptions.namespace` 写到 Vuex `namespace`）。因此用户必须先通过 SurrealDB 表单把它配好，或者它已经在已保存的连接里存在；否则此处不会走 namespace 分支。

4. **物理建连**

   ```ts
   const server = ConnectionProvider.for(config, osUser, settings)
   const connection = server.createConnection(database)
   await connection.connect(abortController.signal)
   ```

   这里才真正调用对应 DB 客户端的 `connect()`。对 SQL Server 会通过 `AzureAuthService.auth(...)` 去拿 Access Token；对 SurrealDB 会用 `surrealDbOptions.protocol` + `namespace` + `authType` 构造 `SurrealPool`（见 [surrealdb.ts#L79-L151](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/db/clients/surrealdb.ts#L79-L151)）。

   **失败路径**：网络失败 / Azure token 获取失败 / SurrealDB 凭证错误 / Cockroach JWT 过期等 → 抛异常 → 前端 `connect` action catch → `connected` 仍 `false`，`ConnectionInterface` 显示 `connectionError`。

### 5.3 ③ `connected` 提交 → 页面切换

Vuex `connected` 一旦被置 `true`，[App.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/App.vue) 的模板分支立即从 `<connection-interface>` 切换为 `<core-interface>`。这是页面切换发生的确切时机。

### 5.4 ④ `newConnection` mutation：写入 `usedConfig` / `namespace`

[store/index.ts#L342-L346](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L342-L346)

```ts
newConnection(state, config) {
  state.usedConfig = config
  state.database = config?.defaultDatabase
  state.namespace = config?.surrealDbOptions?.namespace
}
```

因此 `namespace` state 是在**连接成功之后**才写入的；`CoreInterface` 的 sidebar 用的就是这个值。

### 5.5 ⑤ SurrealDB namespace 列表刷新

```ts
if (usedConfig.connectionType === 'surrealdb'
    && usedConfig.surrealDbOptions?.authType === SurrealAuthType.Root) {
  await context.dispatch('updateNamespaceList')
}
```

`updateNamespaceList` 调用的是 `connection.listSchemas()`，在 SurrealDB 客户端里实现为 `INFO FOR ROOT STRUCTURE`，见 [surrealdb.ts#L350-L354](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/db/clients/surrealdb.ts#L350-L354)。这只是为 sidebar 填充可切换 namespace 选项，不会影响 `connected` 状态。

---

## 6. 各关键步骤的先后关系总览

```
App.vue mounted
  │
  ├─ 从 query.url 拿到连接串 / 文件路径
  │
  ├─ $bks.unlock()                             // PIN 弹窗（仅 lockMode==='pin'）
  │     ├─ 取消 → return
  │     └─ 继续 → { auth: { input, mode: 'pin' } }
  │
  └─ dispatch('openUrl', { url, auth })
        │
        ├─ appdb/saved/parseUrl                // 识别后缀 / 解析 URL → IConnection
        │     └─ 失败 → 抛错，noty.error
        │
        └─ dispatch('connect', { config, auth })
              │
              ├─ resolveJwtConfig              // Cockroach JWT 弹窗（条件触发）
              │     └─ 取消 → return false
              │
              ├─ conn/create（后端）
              │     ├─ PIN 校验（再次，真正 bcrypt 比对）
              │     ├─ Azure authId 分配（条件触发）
              │     ├─ SurrealDB namespace::db 组合（条件触发）
              │     └─ 实际 connect()
              │           └─ 失败 → 抛异常 → connected 仍 false
              │
              ├─ commit('connected', true)    // 页面切到 CoreInterface
              │
              ├─ commit('newConnection', config)   // usedConfig + namespace
              │
              ├─ (SurrealDB Root) updateNamespaceList
              │
              └─ updateDatabaseList / updateTables / updateRoutines
```

---

## 7. 失败路径速查

| 阶段 | 失败原因 | 表现 |
| --- | --- | --- |
| URL 注入 | query 中没有 `url` | 走正常手动连接流程，不进入 `openUrl` |
| `parseUrl` | 后缀不识别 / URL 解析异常 | `$noty.error('Unable to parse ...')`，抛错 |
| 前端 PIN 解锁 | 用户 Cancel | `openUrl` / `submit` 直接返回，不调用 `connect` |
| Cockroach JWT 弹窗 | 用户 Cancel | `connect` return false |
| 后端 PIN 校验 | PIN 不正确 / `auth` 缺失 | `throw 'Incorrect pin...'` → `connected` 仍 false |
| Azure `authId` | TokenCache 创建失败 / msal 初始化失败 | `conn/create` 抛错 → 连接失败 |
| 物理建连 | 网络 / 凭证 / 服务端拒绝 | 异常冒泡到 `connect` catch → `connected` 仍 false，`ConnectionInterface` 显示错误 |
| SurrealDB namespace | `surrealDbOptions.namespace` 未配置 | 不走 `namespace::db` 组合；仅按 `defaultDatabase` 连接 |

---

## 8. 小结

1. **`parseUrl` 最先发生**，只做纯解析，不触网；失败就停在 `ConnectionInterface`。
2. **PIN 解锁**分前后两端：前端 `$bks.unlock()` 只负责弹窗收 PIN，真正的 `bcrypt.compare` 在 `conn/create` 后端完成。两处任意一处取消 / 失败都会让 `connected` 保持 `false`。
3. **Cockroach JWT** 只在 `jwtAuthEnabled && !password` 时触发，且必须在 `conn/create` 之前把 token 写入 `password`，否则 libpq 选项中写入的 JWT 为空。
4. **Azure `authId`** 在 `conn/create` 后端兜底分配，给 `AzureAuthService` 提供 `TokenCache` 行 ID，Access Token 流依赖它持久化 msal tokenCache。
5. **SurrealDB namespace**：`parseUrl` 不会自动填，必须由已保存连接或表单提供；`conn/create` 在 `namespace::db` 形式下拼接，`newConnection` mutation 把它写入 Vuex state，Root 登录后还会 `updateNamespaceList` 供 sidebar 使用。
6. **`connected` 提交是页面切换的确切信号**：`commit('connected', true)` 之后 App.vue 立即从 `ConnectionInterface` 切到 `CoreInterface`，后续的 namespace / tables / routines 刷新都是在 `CoreInterface` 挂载后继续进行。
