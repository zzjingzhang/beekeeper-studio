# SSH Host Alias + Agent 模式:~/.ssh/config → Tunnel 配置追踪

本文档追踪当用户在 Beekeeper Studio 中**仅填写 SSH Host alias** 并选择 **agent 模式** 时,`HostName / Port / User / IdentityFile / IdentitiesOnly` 如何从 `~/.ssh/config` 流入最终的 tunnel 配置,并分析主机与 bastion 的认证顺序以及 `IdentityFile` 读取失败后为何退回完整 agent 身份列表。

---

## 1. 涉及的关键文件

| 文件 | 角色 |
| --- | --- |
| `apps/studio/src-commercial/backend/lib/connection-provider.ts` | 将 UI `IConnection` 转换为 `IDbConnectionServerConfig`,调用 `readSshConfig` 并用返回值回填 `ssh` 字段 |
| `apps/studio/src/lib/ssh/sshConfigReader.ts` | 读取并解析 `~/.ssh/config` |
| `apps/studio/src/lib/db/tunnel.ts` | 用转换后的 config 构造 `SSHConnection` 选项 |
| `apps/studio/src/lib/ssh/sshKeyUtils.ts` | 将 `IdentityFile` 解析为公钥集合 |
| `apps/studio/src/lib/ssh/identitiesOnlyAgent.ts` | 实现 IdentitiesOnly 过滤的包装 agent |
| `apps/studio/src/lib/db/types.ts` | `IDbConnectionServerSSHConfig` 类型定义 |

相关类型(节选,见 [types.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/types.ts#L181-L201)):

```ts
export interface IDbConnectionServerSSHConfig {
  host: Nullable<string>; port: number; user: Nullable<string>;
  password: Nullable<string>; privateKey: Nullable<string>;
  passphrase: Nullable<string>;
  identityFiles?: string[]; identitiesOnly?: boolean;
  bastionHost / bastionPort / bastionUser / bastionPassword /
  bastionPrivateKey / bastionPassphrase /
  bastionIdentityFiles? / bastionIdentitiesOnly?;
  bastionMode: Nullable<SshMode>;
  keepaliveInterval: number; useAgent: boolean;
}
```

---

## 2. 数据流转(主机侧,用户只填 alias + 选 agent)

设用户在表单中填写:

- `sshHost = "my-db"` (alias)
- `sshMode = "agent"` (其它字段均留空)

设 `~/.ssh/config` 有:

```
Host my-db
  HostName 10.0.0.12
  Port 2222
  User alice
  IdentityFile ~/.ssh/id_ed25519_db
  IdentitiesOnly yes
```

### 2.1 阶段一:`convertConfig` 初建 `ssh`

[connection-provider.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/connection-provider.ts#L9-L27) 第 11-27 行按 `sshMode` 初填:

```
ssh = {
  host: "my-db", port: undefined, user: null,
  password: null,                 // sshMode !== 'userpass'
  privateKey: null,               // sshMode !== 'keyfile'
  passphrase: null,               // sshMode !== 'keyfile'
  useAgent: true,                 // sshMode == 'agent'
  bastion*: null, identityFiles/identitiesOnly: undefined,
};
```

关键点:**认证模式(`useAgent` / `privateKey` / `password`)完全由 `sshMode` 决定**,`~/.ssh/config` 永远不会覆盖认证模式,只补足连接参数与 IdentityFile。

### 2.2 阶段二:`readSshConfig("my-db")` 回填

[connection-provider.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/connection-provider.ts#L29-L51) 第 33-51 行:

```ts
const fileConfig = readSshConfig(config.sshHost.trim()); // => {
//   host: "10.0.0.12", port: 2222, user: "alice",
//   identityFile: "/home/alice/.ssh/id_ed25519_db",
//   identityFiles: ["/home/alice/.ssh/id_ed25519_db"],
//   identitiesOnly: true,
// }

if (fileConfig.host) ssh.host = fileConfig.host;         // "10.0.0.12"
if (fileConfig.port && !ssh.port) ssh.port = fileConfig.port;  // 2222
if (fileConfig.user && !ssh.user) ssh.user = fileConfig.user;  // "alice"

if (config.sshMode === 'agent') {                        // 仅当 agent
  if (fileConfig.identityFile && !ssh.privateKey)
    ssh.privateKey = fileConfig.identityFile;            // 注意:填入了 privateKey 路径!
  ssh.identityFiles = fileConfig.identityFiles;
  ssh.identitiesOnly = fileConfig.identitiesOnly === true;
}
```

因此 `ssh` 现在是:

```
{ host: "10.0.0.12", port: 2222, user: "alice",
  useAgent: true,
  privateKey: "/home/alice/.ssh/id_ed25519_db",   // ← 由 ~/.ssh/config 的 IdentityFile 填入
  identityFiles: ["/home/alice/.ssh/id_ed25519_db"],
  identitiesOnly: true,
  password: null, passphrase: null, ... }
```

`readSshConfig` 内部做的事见 [sshConfigReader.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/sshConfigReader.ts#L19-L56):

1. 默认 `configPath = path.join(os.homedir(), ".ssh", "config")`;
2. 用 `ssh-config` 库 `parse` + `compute(host, { ignoreCase: true })` 解析;
3. `IdentityFile` 数组通过 `resolveHomePathToAbsolute` 把 `~` 展开成绝对路径;
4. `identityFile = identityFiles[0]`(兼容单值字段);
5. `IdentitiesOnly` 字符串 `"yes"`/`"no"` 转布尔。

> **重要细节**:即便用户选的是 `agent` 模式,`privateKey` 也被 `fileConfig.identityFile` 填上,这会影响 `tunnel.ts` 中的认证顺序与磁盘读取,见第 3 节。

### 2.3 阶段三:`tunnel.ts` 组装 `sshConfig`

[tunnel.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/tunnel.ts#L65-L174) 按以下顺序构造传给 `SSHConnection` 的 `sshConfig`:

#### (a) agent 可用性与默认 IdentityFile 回退(82-95 行)

```ts
const socketPath = appConfig.sshAuthSock || undefined;
const haveAgent = ssh.useAgent && !!socketPath;              // true
let privateKeyPath = ssh.privateKey || undefined;            // "/home/alice/.ssh/id_ed25519_db" ← 由阶段二填入
if (!privateKeyPath && ssh.useAgent) {
  privateKeyPath = findDefaultIdentityFile();                // 找 ~/.ssh/id_ed25519 等
}
```

- 如果 `~/.ssh/config` 没给 `IdentityFile`,会退到默认 `id_ed25519 / id_ecdsa / id_rsa / id_dsa`(与 OpenSSH 顺序一致)。
- 本案例中 `privateKeyPath` 已经存在,**不会**走默认回退。

#### (b) 基础字段(97-116 行)

```ts
const sshConfig: Options = {
  endHost: "10.0.0.12", endPort: 2222,
  username: "alice",
  password: undefined,
  agentForward: true,                  // haveAgent
  bastionAgentForward: false,
  skipAutoPrivateKey: true,            // 避免 node-ssh-forward 自己去读默认 key
  noReadline: true,
  ...
};
```

注意 `skipAutoPrivateKey: true`,意味着 `node-ssh-forward` 不会自己探测 `~/.ssh/id_*`;所有 private key 的读取只由 `tunnel.ts` 自身控制。

#### (c) privateKey / bastionPrivateKey 磁盘读取(117-123 行)

```ts
if (privateKeyPath) {
  sshConfig.privateKey =
    fs.readFileSync(path.resolve(resolveHomePathToAbsolute(privateKeyPath)));
}
if (bastionPrivateKeyPath) {
  sshConfig.bastionPrivateKey =
    fs.readFileSync(path.resolve(resolveHomePathToAbsolute(bastionPrivateKeyPath)));
}
```

- 只要 `privateKeyPath` 存在,就**同步 `fs.readFileSync`**。
- **文件不存在、权限不够都会抛出异常**,这会把整个 `connectTunnel` 拒绝掉,见 3.1 节。
- `bastionPrivateKeyPath` 同样处理。

#### (d) agent / bastionAgent 可能被过滤(125-140 行)

```ts
if (haveAgent) {
  sshConfig.agent = maybeBuildFilteringAgent(
    ssh.identityFiles, ssh.identitiesOnly, socketPath, ssh.passphrase,
  );
}
if (bastionHaveAgent) {
  sshConfig.bastionAgent = maybeBuildFilteringAgent(
    ssh.bastionIdentityFiles, ssh.bastionIdentitiesOnly, socketPath, ssh.bastionPassphrase,
  );
}
```

`maybeBuildFilteringAgent`([tunnel.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/tunnel.ts#L44-L60)):

```ts
function maybeBuildFilteringAgent(identityFiles, identitiesOnly, socketPath, passphrase) {
  if (!socketPath) return undefined;
  if (!identitiesOnly || !identityFiles?.length) {
    return socketPath;                     // 直接返回 socket 路径,不包装
  }
  const allowed = loadAllowedPublicKeys(identityFiles, passphrase);
  return createFilteringAgent({ socketPath, isWindows, allowedPublicKeys: allowed });
}
```

关键判断:

- 只要 `identitiesOnly` 为假 **或** `identityFiles` 为空/缺失,就直接返回 `socketPath` 字符串,**不做过滤**。
- 只有两者都满足时才构造 `FilteringOpenSSHAgent` / `FilteringPageantAgent`。

`loadAllowedPublicKeys`([sshKeyUtils.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/sshKeyUtils.ts#L15-L68)) 对每个 `IdentityFile`:

1. 先尝试读 `${filePath}.pub` 并 `ssh2.utils.parseKey`;
2. 否则读私钥文件本身 `parseKey(data, passphrase)`;
3. 失败(不存在、加密但无 passphrase、格式不支持)记一条 `warn` 日志并 **跳过**。

因此最终 `allowed` 可能是 **空集合** —— 这会触发下文 3.2 节描述的“退回完整 agent 列表”行为。

`createFilteringAgent`([identitiesOnlyAgent.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/identitiesOnlyAgent.ts#L20-L75)) 包装底层 agent,在 `getIdentities` 回调中:

```ts
if (err) return cb(err);
if (!Array.isArray(keys) || allowedPublicKeys.size === 0) {
  return cb(null, keys);                  // ← 空集合:不过滤
}
cb(null, keys.filter(k => allowedPublicKeys.has(k.getPublicSSH().toString("base64"))));
```

Windows 走 `PageantAgent`,非 Windows 走 `OpenSSHAgent(socketPath)`。

#### (e) `authHandler` 认证顺序(142-151 行)

```ts
sshConfig.authHandler = buildAuthHandler({
  useAgent: haveAgent,
  hasPrivateKey: !!sshConfig.privateKey,   // 注意:基于 *读出来的* Buffer
  hasPassword: !!ssh.password,
});
```

`buildAuthHandler`([tunnel.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/db/tunnel.ts#L36-L42)):

```ts
function buildAuthHandler(opts): AuthenticationType[] {
  const methods = [];
  if (opts.useAgent)      methods.push('agent');
  if (opts.hasPrivateKey) methods.push('publickey');
  if (opts.hasPassword)   methods.push('password');
  return methods;
}
```

所以本案例最终的主机认证顺序是:**agent → publickey**。

#### (f) 最终字段汇总

| `sshConfig` 字段 | 来源 | 本案例值 |
| --- | --- | --- |
| `endHost` | `sshConfig.hostname` → `ssh.host` | `10.0.0.12` |
| `endPort` | `sshConfig.port` → `ssh.port` | `2222` |
| `username` | `sshConfig.user` → `ssh.user` | `alice` |
| `password` | `ssh.password`(agent 模式为 null) | `undefined` |
| `passphrase` | `ssh.passphrase`(agent 模式为 null) | `undefined` |
| `privateKey` | `fs.readFileSync(ssh.privateKey)` | `Buffer`(若文件可读) |
| `agent` | `FilteringOpenSSHAgent` 或 socket 字符串 | 过滤 agent |
| `agentForward` | `haveAgent` | `true` |
| `authHandler` | 见 2.3 (e) | `['agent','publickey']` |

---

## 3. 主机与 Bastion 的认证顺序

### 3.1 主机侧(`sshMode === 'agent'`)

| 条件 | `authHandler` | `agent` 值 |
| --- | --- | --- |
| `useAgent && socketPath && privateKey 文件可读` | `['agent','publickey']` | 若 `identitiesOnly && identityFiles` 有值 → `FilteringOpenSSHAgent`;否则 `socketPath` |
| `useAgent && socketPath && privateKey 文件缺失` | **抛异常**,整个 tunnel 失败 | — |
| `useAgent && socketPath && 无 privateKey 且无默认 identity 文件` | `['agent']` | 同上 |
| `useAgent && 无 socketPath` | `['publickey']`(如果能读到 key) | `undefined` |
| 非 agent 模式(`sshMode === 'keyfile'`) | `['publickey']`(+ 可选 `password`) | `undefined` |

> ⚠️ 第 2 行是当前代码的一个强约束:只要 `ssh.privateKey` 被 `connection-provider` 从 `~/.ssh/config` 填入,**不管用户是不是选了 agent**,`tunnel.ts` 都会 `fs.readFileSync`,读不到就直接抛错,而不是降级为纯 agent 认证。

### 3.2 Bastion 侧(`sshBastionMode === 'agent'`)

Bastion 侧字段是**完全独立**的一套:

| 配置变量 | 字段 |
| --- | --- |
| `bastionHost` / `bastionPort` / `bastionUser` | 来自 `readSshConfig(sshBastionHost)` 的 `HostName / Port / User` |
| `bastionPrivateKey` | 仅当 `sshBastionMode === 'agent'` 且 `fileConfig.identityFile` 存在时,由 `connection-provider` 填入 |
| `bastionIdentityFiles` / `bastionIdentitiesOnly` | 同样仅在 agent 模式下填充 |
| `bastionAuthHandler` | `buildAuthHandler({ useAgent: bastionHaveAgent, hasPrivateKey: !!sshConfig.bastionPrivateKey, hasPassword: !!ssh.bastionPassword })` |
| `bastionAgent` | `maybeBuildFilteringAgent(bastionIdentityFiles, bastionIdentitiesOnly, socketPath, bastionPassphrase)` |
| `bastionAgentForward` | `bastionHaveAgent` |

Bastion 的默认 private key 回退与主机对称:

```ts
let bastionPrivateKeyPath = ssh.bastionPrivateKey || undefined;
if (!bastionPrivateKeyPath && ssh.bastionMode === 'agent') {
  bastionPrivateKeyPath = findDefaultIdentityFile();
}
```

Bastion 的 `authHandler` 顺序同样是 **agent → publickey → password**(按各自条件启用)。

### 3.3 主机 vs Bastion 侧的对称性

- 两套字段彼此独立,`connection-provider` 里分别走 [第 33-51 行](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/connection-provider.ts#L33-L51) 和 [第 53-71 行](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src-commercial/backend/lib/connection-provider.ts#L53-L71);
- `tunnel.ts` 里主机用 `agent/authHandler`,Bastion 用 `bastionAgent/bastionAuthHandler`,二者不会互相污染;
- 共用同一个 `socketPath`,共用同一个默认身份文件探测顺序 `id_ed25519 → id_ecdsa → id_rsa → id_dsa`;
- 各自独立调用 `maybeBuildFilteringAgent`,因此可以出现“主机过滤、Bastion 不过滤”或反之的情况。

---

## 4. `IdentityFile` 无法读取时为什么会退回完整 agent 身份列表

完整链路:

1. `connection-provider` 把 `IdentityFile` 原样保存为**路径字符串**到 `ssh.privateKey` 和 `ssh.identityFiles`。
2. `tunnel.ts` 同步 `fs.readFileSync` 读 `privateKey`。若该路径 **不可读**(不存在 / 权限问题),**整个 `connectTunnel` 会 reject**,此时完全没有 SSH 连接,不会进入“退回”逻辑。
3. 但如果 `privateKey` **可读** 而 `identitiesOnly=true`,会进入 `maybeBuildFilteringAgent` → `loadAllowedPublicKeys`。后者对 **`identityFiles` 中的每个路径** 尝试:
   - 读 `${file}.pub` → `parseKey`;
   - 或读私钥 `parseKey(data, passphrase)`;
   - 任何异常/错误都被 `catch` 并记录 `log.warn`,然后 `return null`。
4. 当所有 `IdentityFile` 都失败时,`allowed` 为空 `Set`,`createFilteringAgent` 仍然被创建(因为 `identitiesOnly && identityFiles.length` 满足),但其 `getIdentities` 内有:
   ```ts
   if (!Array.isArray(keys) || allowedPublicKeys.size === 0) {
     return cb(null, keys);   // 直接原样返回 agent 的完整身份列表
   }
   ```
5. 所以**在 `identitiesOnly=true` 的语义期望下,实际却向服务器提供了 agent 中所有 key**,这就是“退回完整 agent 列表”。

代码直接体现这一点:

- [sshKeyUtils.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/sshKeyUtils.ts#L48-L68) 中任何失败都会 `log.warn` 并返回 `null`;
- [identitiesOnlyAgent.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/identitiesOnlyAgent.ts#L27-L31) 明确记录 `"IdentitiesOnly requested but no public keys were resolvable; using agent without filtering"`;
- [identitiesOnlyAgent.ts](file:///c:/Users/10244/Desktop/0508-under/beekeeper-studio/apps/studio/src/lib/ssh/identitiesOnlyAgent.ts#L39-L41) 当 `allowedPublicKeys.size === 0` 时 **不过滤**。

**设计意图**:这种“宁可宽松,不可拒连”的 fallback 是故意为之 —— 如果严格按 `IdentitiesOnly` 过滤空集,会让服务器看到 0 个身份,认证必然失败;与其静默失败,不如把选择权交给服务器(与用户)。日志中的 `warn` 让运维能在问题排查时发现配置异常。

**典型触发场景**:

- `IdentityFile` 指向的私钥加密了,但 `agent` 模式下没有提供 `passphrase`(因为 UI 不暴露),`parseKey` 失败;
- `~/.ssh/config` 引用的文件在另一台机器上不存在(例如用户在 Windows 上迁移了配置);
- 私钥格式是 `ssh2` 不支持的新版本(如 `sk-ssh-ed25519@openssh.com` 未启用的情况)。

---

## 5. 小结:从 alias 到 tunnel 的单条数据流

```
UI: sshHost="my-db", sshMode="agent"
    │
    ▼
connection-provider.ts                ─ 初始 ssh{useAgent:true, privateKey:null}
    │ readSshConfig("my-db")
    ▼
sshConfigReader.ts                     ─ {host,port,user,identityFile(s),identitiesOnly}
    │
    ▼
connection-provider.ts                ─ 回填 ssh.host/port/user
    │ (仅 sshMode==='agent')           ─ 回填 ssh.privateKey / identityFiles / identitiesOnly
    ▼
tunnel.ts
    ├─ socketPath + useAgent  →  haveAgent
    ├─ privateKey 路径 → findDefaultIdentityFile 回退
    ├─ fs.readFileSync → sshConfig.privateKey(Buffer)
    ├─ maybeBuildFilteringAgent(identityFiles, identitiesOnly, ...)
    │     ├─ 不满足 → socketPath
    │     └─ 满足    → loadAllowedPublicKeys → createFilteringAgent
    │                     │ 全部失败         ↘ empty Set → 不包装过滤逻辑
    │                     │ 至少一个成功     → 包装过滤 agent
    ├─ buildAuthHandler(haveAgent, !!sshConfig.privateKey, !!ssh.password)
    │     → ['agent','publickey']   (或 'agent' 或 'publickey')
    └─ new SSHConnection(sshConfig).forward(...)
```
