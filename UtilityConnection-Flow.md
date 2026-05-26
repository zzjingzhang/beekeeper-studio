# UtilityConnection 消息流转与 Utility 进程崩溃恢复分析

> 本文档从 renderer 第一次调用 `Vue.prototype.$util.send(...)`（此时端口尚未建立）开始，沿着调用栈完整追踪：
> 消息如何排队 → 如何触发 `requestPorts` → 如何创建 `MessageChannelMain` → 如何绑定 `sId` → 如何路由 `reply/error`。
> 再分析 utility process 非零退出后，多窗口场景下哪些状态会丢失，哪些请求可能被重放或失败。

---

## 一、参与者与关键文件

| 角色 | 文件 | 职责 |
|------|------|------|
| Renderer (Vue 前端) | [renderer.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/renderer.ts) | 挂载 `Vue.prototype.$util`，监听 `window.onmessage` 接收端口 |
| UtilityConnection (渲染侧) | [UtilityConnection.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts) | 封装消息队列、`sId` 绑定、`replyHandlers` 路由 |
| Preload Bridge | [preload.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts) | 暴露 `requestPorts`、`attachPortListener`、`onUtilDied` 到 `window.main` |
| Main Process | [main.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts) | 创建 utilityProcess，监听 `requestPorts` IPC，创建 `MessageChannelMain`，分配 `sId` |
| WindowBuilder | [WindowBuilder.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/background/WindowBuilder.ts) | `BeekeeperWindow` 维护 `sId` 属性，提供 `getActiveWindows` 列表 |
| Utility Process | [utility.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts) | 接收 `sId`/端口，创建 per-`sId` `State`，分发 handler 调用，回发 `reply`/`error` |
| Handler State | [handlerState.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts) | per-`sId` `State`：端口、DB 连接、查询、导出、临时文件等 |
| UtilDiedModal | [UtilDiedModal.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/UtilDiedModal.vue) | utility 崩溃时弹窗提示用户 |

---

## 二、Renderer 首次 `$util.send` 的完整追踪

### 2.1 启动序列（基础）

[renderer.ts#L153-L196](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/renderer.ts#L153-L196)：

```ts
const utility = new UtilityConnection()
// ...
Vue.prototype.$util = utility;
window.main.attachPortListener();
window.onmessage = (event) => {
  if (event.source === window && event.data.type === 'port') {
    const [port] = event.ports;
    const { sId } = event.data;
    Vue.prototype.$util.setPort(port, sId);
    app.$store.dispatch('settings/initializeSettings');
  }
}
```

关键点：
1. `UtilityConnection` 被 **实例化一次**，随后作为 `Vue.prototype.$util` 被所有组件共享。
2. `attachPortListener` 在 preload 中注册 IPC `'port'` 监听器，并通过 `window.postMessage({ type: 'port' }, '*', event.ports)` 把端口转交给 renderer 主 world（绕过 contextIsolation）。
3. 此时 `utility.port` 仍为 `undefined`，`messageQueue` 为空数组，`portsRequested = false`。

### 2.2 首次 `$util.send(name, args)` 调用（端口尚未建立）

[UtilityConnection.ts#L90-L110](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L90-L110)：

```ts
public async send(handlerName: string, args?: any): Promise<any> {
  return new Promise<any>((resolve, reject) => {
    const id = uuidv4();
    if (!this.port) {
      this.messageQueue.push({ handlerName, args, id, resolve, reject });
      if (!this.portsRequested) {
        window.main.requestPorts();
        this.portsRequested = true;
      }
    } else { /* 已建立端口时的分支 */ }
  })
}
```

**进入排队分支时的动作：**

1. 生成一个 UUID（`id`），用于后续匹配 `reply`/`error`。
2. 把 `{ handlerName, args, id, resolve, reject }` 压入 `messageQueue`。此时 **resolve/reject 闭包不会被触发**，调用方 `await` 永久挂起直到收到端口并清空队列。
3. `portsRequested` 是一次性闸门：只有第一次 `send` 会触发 `window.main.requestPorts()`，后续并发的 `send` 只入队不重复请求。避免多个渲染器组件同时抢占端口导致重复 `MessageChannelMain`。

### 2.3 Preload → Main：`requestPorts` IPC

[preload.ts#L180-L182](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts#L180-L182)：

```ts
requestPorts() {
  ipcRenderer.invoke('requestPorts');
}
```

### 2.4 Main 收到 `requestPorts`，创建 `MessageChannelMain` 并绑定 `sId`

[main.ts#L259-L272](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L259-L272)：

```ts
ipcMain.handle('requestPorts', async () => {
  if (!utilityProcess || !utilityProcess.pid) {
    utilityProcess = null;
    await createUtilityProcess();
  }
  if (newWindows.length > 0) {
    createAndSendPorts(true);   // 只给新窗口发
  } else {
    createAndSendPorts(false);  // 全量重发
  }
})
```

`createAndSendPorts(filter, utilDied=false)` 位于 [main.ts#L240-L257](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L240-L257)：

```ts
function createAndSendPorts(filter: boolean, utilDied = false) {
  getActiveWindows().forEach((w) => {
    if (!filter || newWindows.includes(w.winId)) {
      const { port1, port2 } = new electron.MessageChannelMain();
      const sId = uuidv4();
      w.sId = sId;
      utilityProcess.postMessage({ type: 'init', sId }, [port1]);
      w.webContents.postMessage('port', { sId, utilDied }, [port2]);
      w.onClose((_event) => {
        utilityProcess.postMessage({ type: 'close', sId })
      })
      if (filter) newWindows = _.without(newWindows, w.winId);
    }
  })
}
```

**每一个活动窗口都会得到一对独立的端口 + 独立的 `sId`**：

| 步骤 | 作用 |
|------|------|
| `new MessageChannelMain()` | 在主进程创建双向消息通道（一对 `MessagePortMain`） |
| `uuidv4()` → `sId` | 为该窗口生成唯一会话 ID，贯穿整个生命周期 |
| `w.sId = sId` | 把 `sId` 写到 `BeekeeperWindow` 实例，后续 `maximize-${sId}` 等窗口级事件用它路由（见 [WindowBuilder.ts#L111-L129](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/background/WindowBuilder.ts#L111-L129)） |
| `utilityProcess.postMessage({ type: 'init', sId }, [port1])` | 把 `port1` 连同 `sId` 转移到 utility 进程 |
| `w.webContents.postMessage('port', { sId, utilDied }, [port2])` | 把 `port2` 转到对应 renderer 的 preload → renderer |
| `w.onClose(...)` | 窗口关闭时通知 utility 清理该 `sId` 的 `State` |

注意 `createUtilityProcess()` 中的去重与启动流程（[main.ts#L46-L92](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L46-L92)）：
- 若 `utilityProcess` 已存在则直接返回。
- 否则 `utilityProcess.fork(...)` 启动 `utility.js`，向子进程发送 `{ type: 'init' }`（无端口），并等待子进程回发 `{ type: 'ready' }` 才 resolve。

### 2.5 Utility 进程收到 `{type:'init', sId, port1}`，创建 per-`sId` State

[utility.ts#L119-L138](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts#L119-L138)：

```ts
process.parentPort.on('message', async ({ data, ports }) => {
  const { type, sId } = data;
  switch (type) {
    case 'init':
      if (ports && ports.length > 0) {
        await initState(sId, ports[0]);
      } else {
        await init();     // 进程首次启动：建立 orm/plugin 并回发 ready
      }
      break;
    case 'close':
      state(sId).port.close();
      removeState(sId);
      break;
  }
})
```

`initState` 在 [utility.ts#L177-L188](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts#L177-L188)：

```ts
async function initState(sId: string, port: MessagePortMain) {
  newState(sId);
  state(sId).port = port;
  state(sId).port.on('message', ({ data }) => {
    const { id, name, args } = data;
    runHandler(id, name, args);
  })
  state(sId).port.start();
}
```

[handlerState.ts#L49-L62](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts#L49-L62)：

```ts
const states = new Map<string, State>();
export function state(id: string): State { return states.get(id); }
export function newState(id: string): void { states.set(id, new State()); }
export function removeState(id: string): void { states.delete(id); }
```

`State` 结构（[handlerState.ts#L19-L47](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts#L19-L47)）：

| 字段 | 含义 |
|------|------|
| `port` | 该 `sId` 对应的回传端口 |
| `server` / `connection` | 数据库 server/client |
| `usedConfig` | 当前连接配置 |
| `database` / `username` | 当前上下文 |
| `queries` | `Map<string, CancelableQuery>` 运行中的查询 |
| `generator` | SQL 生成器 |
| `exports` / `imports` / `backupProc` | 导入/导出/备份进度 |
| `transactionTimeouts` | 事务空闲超时 |
| `connectionAbortController` | 连接取消句柄 |
| `enumsInitialized` / `enumWatcher` | 自定义枚举文件监听 |
| `tempFiles` | 临时文件池 |

> **sId 是“窗口 → utility 会话”的唯一键**。每个 `BrowserWindow` 拥有独立 `sId`，utility 进程以 `sId` 为粒度维护全部可变状态。

### 2.6 Preload 转交到 Renderer

[preload.ts#L171-L179](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts#L171-L179)：

```ts
attachPortListener() {
  ipcRenderer.on('port', (event, { sId, utilDied }) => {
    window.postMessage({ type: 'port', sId }, '*', event.ports);
    if (utilDied) {
      ipcRenderer.emit('utilDied');
    }
  })
}
```

关键点：
- `event.ports` 是 `MessagePortMain` 在 renderer 侧被转换成 `MessagePort`（web 标准对象）。
- `window.postMessage(..., '*', event.ports)` 把该 `MessagePort` 从 preload 的隔离上下文传递到主 world。
- `utilDied=true` 时会本地触发 `utilDied`，让 `UtilDiedModal` 弹出。

### 2.7 Renderer `setPort`：绑定 `sId`、监听消息、刷新队列

[UtilityConnection.ts#L38-L88](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L38-L88)：

```ts
public setPort(port: MessagePort, sId: string) {
  this.port = port;
  this._sId = sId;
  this.port.onmessage = (msg) => {
    const { data: msgData } = msg;
    if (msgData.type === 'error') {
      const { id, error, stack } = msgData;
      const handler = this.replyHandlers.get(id);
      if (handler) {
        this.replyHandlers.delete(id);
        const err = new Error(error);
        err.stack = stack;
        handler.reject(err);
      }
    } else if (msgData.type === 'reply') {
      const { id, data } = msgData;
      const handler = this.replyHandlers.get(id);
      if (handler) {
        this.replyHandlers.delete(id);
        handler.resolve(data);
      }
    } else if (_.some(this.listeners, ({type}) => msgData.type === type)) {
      const { listener, type, id } = this.listeners.find(({type}) => msgData.type === type);
      listener(input);
    }
  }
  this.port.start();

  if (this.messageQueue.length > 0) {
    this.messageQueue.forEach(({ handlerName, args, id, resolve, reject }) => {
      args = { sId: this._sId, ...args };        // ★ 注入 sId
      this.replyHandlers.set(id, { resolve, reject });
      this.port.postMessage({ id, name: handlerName, args: args ?? {}})
    });
    this.messageQueue = [];
  }
}
```

**绑定 sId 的关键注入点**：队列冲刷时执行 `args = { sId: this._sId, ...args }`，之后所有 `send` 调用走 [L101-L107](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L101-L107) 的直送分支，同样会注入 `sId`。

### 2.8 Utility 分发 Handler 并回发 `reply` / `error`

[utility.ts#L140-L175](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts#L140-L175)：

```ts
async function runHandler(id: string, name: string, args: any) {
  const replyArgs: Reply = { id, type: 'reply' };
  if (handlers[name]) {
    return handlers[name](args)
      .then((data) => { replyArgs.data = data; })
      .catch((e) => {
        replyArgs.type = 'error';
        replyArgs.stack = e?.stack;
        replyArgs.error = e?.message ?? e;
      })
      .finally(() => {
        try { state(args.sId).port.postMessage(replyArgs); } catch (e) { /* ... */ }
      });
  } else {
    replyArgs.type = 'error';
    replyArgs.error = `Invalid handler name: ${name}`;
    state(args.sId).port.postMessage(replyArgs);
  }
}
```

handler 注册表（[utility.ts#L75-L94](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts#L75-L94)）把 `conn/*`、`query/*`、`appdb/*`、`export/*` 等合并为一个 `handlers` 对象。调用者通过 `name` 直接命中；未命中时 utility 本地回发 `error`。

### 2.9 端到端时序图

```
Renderer                                         Main Process                              Utility Process
  │                                                 │                                           │
  │─ Vue.prototype.$util.send(name, args) ─────────▶│                                           │
  │   (port 未建立，入 messageQueue)                 │                                           │
  │─ window.main.requestPorts() ──ipcRenderer.invoke─▶                                           │
  │                                                 │─ utilityProcess.fork(utility.js) ────────▶│
  │                                                 │─ postMessage({type:'init'}) ─────────────▶│
  │                                                 │     (无端口, 触发 init() 建 ORM/插件)     │
  │                                                 │◀─ postMessage({type:'ready'}) ────────────│
  │                                                 │                                           │
  │                                                 │  for each active window:                  │
  │                                                 │   ├ new MessageChannelMain()               │
  │                                                 │   ├ sId = uuidv4()                         │
  │                                                 │   ├ w.sId = sId                           │
  │                                                 │   ├ postMessage({type:'init',sId},[port1])│
  │                                                 │   └ postMessage('port',{sId},[port2]) ──▶│
  │◀─ ipcRenderer.on('port') ──────────────────────│                                           │
  │   window.postMessage({type:'port'},'*',[port2]) │                                           │
  │◀─ window.onmessage 收到 port2 + sId             │                                           │
  │  $util.setPort(port, sId)                       │                                           │
  │    ├ port.onmessage 分发 reply/error             │                                           │
  │    └ 冲刷 messageQueue:                         │                                           │
  │         args = { sId, ...args }                 │                                           │
  │         replyHandlers.set(id, {resolve,reject}) │                                           │
  │         port.postMessage({id,name,args}) ──────────────────────────────────────────────────▶│
  │                                                 │                                           │
  │                                                 │                          state(args.sId)   │
  │                                                 │                          .port.on('message')│
  │                                                 │                          runHandler(id,name,args)
  │                                                 │                          ├ handlers[name](args)
  │                                                 │                          └ .finally:     │
  │                                                 │                              port.postMessage(replyArgs)
  │◀─ port.onmessage({type:'reply'|'error',id,...}) │◀──────────────────────────────────────────│
  │   replyHandlers.get(id).resolve/reject(data)    │                                           │
```

---

## 三、消息路由表

### 3.1 id（请求级）

- 由 [UtilityConnection.ts#L92](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L92) 每次 `send` 生成 UUID。
- 入 `replyHandlers: Map<string, {resolve, reject}>`。
- 只有 `setPort` 冲刷队列时 **或** 走已建端口的直送分支时才入表；排队期间 `replyHandlers` 未登记。
- 收到 `reply`/`error` 时 `replyHandlers.delete(id)` + `resolve/reject`。

### 3.2 sId（会话级 / 窗口级）

- 由 Main 的 `createAndSendPorts` 每窗口生成一次（[main.ts#L244](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L244)）。
- 写回 `BeekeeperWindow.sId`（[main.ts#L246](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L246)）。
- 作为消息 `args` 的第一个字段注入（[UtilityConnection.ts#L82](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L82)、[L103](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L103)）。
- utility 端 `state(args.sId)` 找到该窗口的 `State`、端口、数据库连接、查询等上下文。

### 3.3 listener（服务端推送）

- `UtilityConnection.addListener(type, listener)` 注册，用于 utility → renderer 推送式消息（未在当前代码库里使用，但基础设施存在）。
- 在 `port.onmessage` 的最后一个分支匹配（[UtilityConnection.ts#L67-L72](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L67-L72)）。

---

## 四、Utility 进程非零退出后的恢复流程

### 4.1 Main 侧检测与重启

[main.ts#L67-L76](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L67-L76)：

```ts
utilityProcess.on('exit', async (code) => {
  if (code) {
    log.info('Utility process died, restarting')
    utilityProcess = null;
    await createUtilityProcess();
    createAndSendPorts(false, true);
  }
})
```

`code` 非零时：
1. 把 `utilityProcess` 置 `null`（去掉旧引用，让 GC 回收）。
2. `createUtilityProcess()` 重新 fork 新进程、建立 ORM、发 `ready`。
3. `createAndSendPorts(false, true)`：对 **全部活动窗口** 重新生成一对端口 + 新 `sId`，并携带 `utilDied: true` 通知。

### 4.2 Renderer 侧重新绑定

[renderer.ts#L187-L196](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/renderer.ts#L187-L196)：

```ts
window.onmessage = (event) => {
  if (event.source === window && event.data.type === 'port') {
    const [port] = event.ports;
    const { sId } = event.data;
    Vue.prototype.$util.setPort(port, sId);
    app.$store.dispatch('settings/initializeSettings');
  }
}
```

每次收到 `'port'` 消息都会再次调用 `setPort`，它会：
1. 覆盖 `this.port` 与 `this._sId`（旧端口被 JS GC 回收，Electron 底层随之关闭）。
2. 注册新的 `onmessage`。
3. **不会** 主动 reject 旧 `replyHandlers` 里仍在等待的 Promise（见下节风险）。
4. 不调用 `initializeSettings` 之外的任何 Vuex 重置逻辑。

Preload 中 `utilDied=true` 时 [preload.ts#L175-L177](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts#L175-L177) 触发：

```ts
if (utilDied) {
  ipcRenderer.emit('utilDied');
}
```

被 `UtilDiedModal` 的 `onUtilDied` 捕获，弹出提示对话框；用户点 “Disconnect” 会走 [UtilDiedModal.vue#L42-L45](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/UtilDiedModal.vue#L42-L45) 触发 `$store.dispatch('disconnect')`，由 [store/index.ts#L557-L567](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L557-L567) 清空 `usedConfig / connected / tables / routines` 等前端状态。

### 4.3 Utility 侧新进程的 `initState`

新进程收到 `{type:'init', sId, port}` 时执行 `initState`，为每个 `sId` 新建 `State`。旧进程里的 `states` Map 随进程退出自然消失，因此所有 per-`sId` 的可变状态完全重置为“空壳”。

---

## 五、多窗口场景下崩溃后会丢失哪些状态

以 **N 个活动窗口（N ≥ 2）** 的视角分析：

| 状态类别 | 存储位置 | 崩溃后是否保留 | 说明 |
|---------|----------|----------------|------|
| `MessagePort`（renderer↔utility） | Main `MessageChannelMain` | ❌ 丢失 | 旧端口随旧进程被销毁；Main 为每窗口重建一对新端口，并通过新 `sId` 重发 |
| `sId` | `BeekeeperWindow.sId` + renderer `_sId` | ❌ 变化 | 每个窗口都会得到一个 **新的 UUID**，旧 `sId` 不再被任何一方引用 |
| `State.connection` / `.server` / `.usedConfig` | utility 进程 `states` Map | ❌ 丢失 | 新进程 `State` 空壳；renderer 仍可能认为自己“connected”（见 5.1） |
| `State.queries`（`CancelableQuery`） | utility 进程 | ❌ 丢失 | 正在执行的查询随进程死亡中止，**对应 renderer 的 Promise 未被 reject** |
| `State.exports` / `.imports` / `.backupProc` | utility 进程 | ❌ 丢失 | 导入/导出/备份子进程被杀死 |
| `State.transactionTimeouts` | utility 进程 | ❌ 丢失 | 活动事务回滚或超时丢失 |
| `State.tempFiles` | utility 进程 | ❌ 丢失句柄 | 文件本身留在磁盘 tmp，但 `FileHandle` 引用丢失，可能泄漏 |
| `State.enumWatcher` | utility 进程 | ❌ 丢失 | 自定义枚举文件 watcher 失效 |
| ORM 连接（appdb） | utility 进程 `ormConnection` | ✅ 重建 | 新进程 `init()` 重新 `connect()` 到同一个 SQLite 文件，**持久数据不变** |
| 插件 manager / 驱动依赖 manager | utility 进程 | ✅ 重建 | `pluginManager.initialize()` / `driverDepManager.initialize()` 在新进程重跑 |
| Renderer Vuex `state.usedConfig / connected / tables / ...` | Renderer 内存 | ⚠️ 默认保留 | 除非用户在 `UtilDiedModal` 点 “Disconnect” 或显式调用 `disconnect`，否则前端仍认为自己连接着数据库，但实际 `State.connection` 已为 null |
| Renderer `replyHandlers` | `UtilityConnection` 实例 | ⚠️ 保留但失效 | 旧 `id` 对应的 Promise 永远不会 resolve/reject，导致调用方 `await` 悬挂；**没有超时或清理机制** |
| Renderer `messageQueue` | `UtilityConnection` 实例 | ⚠️ 只在首次无端口时才使用 | 崩溃恢复是“已有端口 → 重新 setPort”，因此队列不参与恢复 |
| `BeekeeperWindow.sId`（窗口对象） | Main 内存 | ✅ 被覆盖为新 sId | `createAndSendPorts` 重新赋值 |
| `newWindows` 集合 | Main 内存 | ✅ 保留 | 崩溃重启不走 `browser-window-created`，该列表不影响恢复路径（`createAndSendPorts(false)` 全量发送） |
| 最大化/全屏事件信道名 `maximize-${sId}` | Main/Renderer IPC | ⚠️ 错位 | `WindowBuilder` 在发送时使用 `this.sId`（[WindowBuilder.ts#L111](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/background/WindowBuilder.ts#L111)），该值被 `createAndSendPorts` 更新为新 sId；但 renderer 在 `onMaximize` 中订阅时传入的可能是 **旧 sId**（[preload.ts#L111-L113](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts#L111-L113)），导致 maximize/unmaximize 事件在恢复后 **可能收不到** |

### 5.1 “前端仍 connected 但 utility 已重置”的错位

这是最容易出问题的场景：

1. 窗口 A 已 `connect()` 到 PostgreSQL，Vuex `state.connected = true`，`state.connection = BasicDatabaseClient`。
2. utility 崩溃重启。
3. 新的 utility 进程里 `state(newSId).connection === null`。
4. 用户在窗口 A 点击 “Execute Query”，`ElectronUtilityConnectionClient.query()` 调 `$util.send('conn/query', ...)`，消息到新 utility 进程：
   - handler 执行 `state(args.sId).connection.query(...)` → `state(args.sId).connection` 为 null → 抛 `errorMessages.noDatabase`（[handlerState.ts#L80-L83](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts#L80-L83)）→ renderer 收到 `error`。
5. 用户只能通过 `UtilDiedModal` 显式 `disconnect` 再 `connect` 才能恢复。

多窗口下：每个窗口都要各自处理一次 `disconnect` + `connect`。

### 5.2 `replyHandlers` 的悬挂 Promise

崩溃时：
- 所有尚未返回的 `replyHandlers` Map 项对应的 `resolve/reject` 闭包 **永不再被调用**。
- 新 `setPort` 只覆盖 `port`/`sId` 与注册新 `onmessage`，**不清空旧 `replyHandlers`**。
- 结果：旧 `id` 对应的调用方 `await` 永远挂起，内存泄漏（闭包引用请求参数、响应回调等）。
- 多窗口下：每个窗口的 `UtilityConnection` 实例独立，都存在同样的悬挂问题。

### 5.3 端口 & `sId` 的“硬切换”

`createAndSendPorts(false, true)` 对所有活动窗口无差别地重发新端口：
- 若某窗口此时正在执行一个 handler（如大文件导出），新端口会在 handler 完成前送达；旧 handler 使用 `state(oldSId).port.postMessage(...)` 回发 → 旧端口已随旧进程消失 → 抛错，被 `catch` 吞掉（[utility.ts#L162](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts#L162)）。
- 崩溃发生在 `new UtilityConnection.setPort` **之前**（即 renderer 还没收到第一次端口）时，走的是 L94-L100 的“排队+requestPorts”分支；崩溃重启后新 `port` 送达再次 `setPort`，队列被冲刷。这一路径相对安全。

### 5.4 `initRootStates` 等一次性动作

[renderer.ts#L200](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/renderer.ts#L200) 在首次 `setPort` 之后立即 `await store.dispatch('initRootStates')`。崩溃恢复后的第二次 `setPort` 只重新跑 `settings/initializeSettings`，**不会** 重跑 `initRootStates`。多窗口场景下，崩溃发生前已经初始化过的窗口不会丢失 store 结构，只是与 utility 的会话被重置。

---

## 六、哪些请求会“重放”，哪些会“失败”

在 utility 崩溃重启之后，对一个 renderer 窗口而言：

| 请求类型 | 是否可由后续操作自然“重放” | 失败表现 |
|----------|------------------------------|----------|
| `conn/query`（正在执行） | ❌ | Promise 悬挂，调用方永远 `await`；用户手动再次触发则重新走新 `sId` 会话 |
| `conn/executeCommand`、`conn/listTables`、`conn/supportedFeatures` | ⚠️ 取决于是否已连接 | 若用户未 `disconnect` 重连，`State.connection` 为 null → `errorMessages.noDatabase` → reject |
| `appdb/*`（本地 SQLite） | ✅ | 新进程重建 `ormConnection`，直接可用；**一般不会失败** |
| `export/*`、`import/*`、`backup/*` | ❌ | 子进程被杀死，进度/临时文件丢失；前端若仍在显示进度则永远不结束 |
| `query/cancel` | ❌ | `State.queries` Map 已空，取消无意义，但 handler 不会抛错 |
| `conn/disconnect` | ✅ | 幂等；即使 `State.connection` 已为 null，重新建立会话后再调一次也可成功 |
| `conn/connect` | ✅ | 用户重新 `connect` 即可恢复；是显式恢复的标准入口 |
| `tabhistory/clearDeletedTabs` 等一次性写入 | ❌ | 已写入的提交到 SQLite，不会丢；尚未发送的则随 Promise 悬挂而丢失 |

### 6.1 请求“重放”的真正语义

代码库中 **没有** 针对崩溃的自动请求重放机制（没有幂等键、没有重试队列、`replyHandlers` 也不迁移）。所谓“重放”仅来自：

1. 用户手动再次触发 UI 动作（再次点击 Execute、再次导出）。
2. Vuex 中尚未清空的状态驱动的副作用（例如 DataManager 再次 poll、定时任务再跑）。

这些都会在新 `sId` 会话上产生全新的请求，与崩溃前的 Promise 无关联。

### 6.2 多窗口之间是否互相影响

- 每个窗口独立 `sId`，独立 `MessagePort`，独立 `State`。窗口 A 崩溃恢复不会影响窗口 B 的现有会话（除了 `createAndSendPorts(false, true)` 对 **所有** 窗口强制重发端口，导致所有窗口都切到新 `sId`、重置会话）。
- 共享资源：`ormConnection`（SQLite 文件）、插件 manager、driverDep manager 在新 utility 进程内被所有 `sId` 共享。它们会被重新初始化，与崩溃前等价。
- `newWindows` 列表只在 `browser-window-created` 事件时累加，崩溃重启路径不经过它，因此不影响恢复。

---

## 七、潜在风险与改进建议

| 风险 | 位置 | 建议 |
|------|------|------|
| 崩溃后旧 `replyHandlers` 永不清理 → Promise 悬挂 + 内存泄漏 | [UtilityConnection.ts#L38-L75](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L38-L75) | 在 `setPort` 开头遍历 `replyHandlers`，对未完成项 `reject(new Error('utility restarted'))` 并清空 |
| 前端仍显示 `connected` 但 utility 无会话 | [store/index.ts#L491-L532](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts#L491-L532) | 在 `utilDied` 触发时自动 `dispatch('disconnect')`，或在 `setPort` 时做一次“连接有效性握手” |
| `maximize-${sId}` 信道错位 | [WindowBuilder.ts#L111-L129](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/background/WindowBuilder.ts#L111-L129) + [preload.ts#L111-L128](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts#L111-L128) | renderer 端在 `setPort` 后重新订阅 maximize/unmaximize/fullscreen，或改用非 sId 的通道（如 `maximize-${winId}`） |
| 崩溃恢复对所有窗口“一刀切”新 sId | [main.ts#L74](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts#L74) | 若仅单窗口出问题，可按窗口粒度重建；或在新端口送达前先 reject 旧会话 |
| 长耗时请求（导出/备份/大查询）无进度持久化 | utility 进程 `State.exports/imports/backupProc` | 引入基于文件的进度快照，使崩溃后可续传 |
| `tempFiles` 可能泄漏 | [handlerState.ts#L46](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts#L46) | 在新 `init()` 启动时扫描并清理 tmp 目录 |
| `portsRequested` 永不重置 | [UtilityConnection.ts#L23](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts#L23) | 崩溃重建后 `portsRequested` 仍为 true，`send` 走直送分支；若极端情况下新端口未送达，会卡在“已请求但无端口”的死区。可在 `setPort` 或 `utilDied` 时把它重置为 false |

---

## 八、文件索引

| 文件 | 作用 |
|------|------|
| [renderer.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/renderer.ts) | 挂载 `$util`，监听 `window.onmessage` 绑定端口 |
| [preload.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/preload.ts) | `requestPorts`/`attachPortListener`/`onUtilDied` 桥接 |
| [main.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/main.ts) | utilityProcess 生命周期、`requestPorts` handler、`createAndSendPorts`、`exit` 回调 |
| [utility.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/entrypoints/utility.ts) | utility 进程入口，init/initState/runHandler、reply/error 回发 |
| [UtilityConnection.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/utility/UtilityConnection.ts) | renderer 侧封装：排队、`setPort`、`replyHandlers`、`listeners` |
| [WindowBuilder.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/background/WindowBuilder.ts) | `BeekeeperWindow.sId`、`getActiveWindows`、窗口最大化事件通道 |
| [handlerState.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/handlers/handlerState.ts) | per-`sId` `State` 结构与增删查 |
| [UtilDiedModal.vue](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/components/UtilDiedModal.vue) | utility 崩溃 UI 提示 |
| [store/index.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/store/index.ts) | `clearConnection`、`disconnect`、`connect` 等动作 |
