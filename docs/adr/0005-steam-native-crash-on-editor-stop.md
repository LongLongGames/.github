# ADR-0005：Steam 非正常断网后 Editor Stop 引发 Native 硬崩溃

- **状态**：Accepted
- **日期**：2026-10-03
- **决策者**：GameAct Client
- **相关**：`Docs/SteamP2P.md`、`Docs/Transport.md`、Steamworks.NET（`com.rlabrecque.steamworks.net`）、LES `SteamP2PTransport`

---

## 1. 背景（Context）

联机主路径使用 **Steam Networking Sockets P2P**（`SteamP2PTransport`）+ Steam Lobby 做房间发现。

复现路径（高频开发场景）：

1. Host 保持运行；
2. Remote **Client 在 Unity Editor** 进入 `Map1`（`SessionMode.Client` + SteamP2P）；
3. 非正常断网（对端掉线 / 管道已异常）或仍处于已连接状态；
4. 在 Editor 中直接点击 **Stop** 退出 Play 模式。

结果：**整个 Unity Editor 进程被操作系统强杀**，无 Unity Bug Reporter 弹窗（Native Access Violation）。

日志（优先看 **`Editor-prev.log`**，因闪退后新会话会覆盖 `Editor.log`）典型片段：

```text
src\steamnetworkingsockets\clientlib\steamnetworkingsockets_socketthread.cpp (2696) :
  Assertion Failed: SteamnetworkingSockets service thread waited 125ms for lock!

src\common\ipcclient.cpp (98) : !"Invalid pipe handle specified"
src\common\interfacemap.cpp (886) :
  Assertion Failed: IPC call to IClientNetworkingUtilsSerialized::PostConnectionStateMsg failed …

[Steam] LeaveLobby
GameAct.Steam.SteamService:Shutdown ()
GameAct.Steam.SteamRunner:OnApplicationQuit ()
```

即：托管层仍在执行退出清理，底层 Steam IPC **pipe 已非法**，继续 Native 调用导致硬崩溃。

---

## 2. 问题陈述（Problem）

Unity Editor 与 Play 模式 **共享同一进程**。Steamworks / GameNetworkingSockets 的 Native 线程与 IPC 在「管道已损坏」时：

- `SteamMatchmaking.LeaveLobby`
- `SteamAPI.Shutdown`
- `SteamNetworkingSockets.CloseConnection` / `CloseListenSocket` / `DestroyPollGroup`

任一调用都可能触发 **Access Violation**。C# `try/catch` **无法拦截** Native 级崩溃。

开发期若只能依赖 Editor Stop 做断线测试，会反复拖死整个编辑器，成本高且难以保留完整现场。

---

## 3. 决策（Decision）

采用 **「托管必清、Native 可跳」** 的分层退出策略，按环境与 Steam 健康状态分流：

### 3.1 原则

| 原则 | 说明 |
|------|------|
| 托管状态总是清理 | `IsInitialized`、LobbyId、Peer 字典、`Role`、事件解绑等必须复位，避免二次进局脏状态 |
| Native 调用需门禁 | 仅当 `Application.isPlaying` 且 `SteamAPI.IsSteamRunning()` 等探测通过时才调用 Close / Shutdown |
| Editor 不硬关 SteamAPI | Play 模式结束时 **跳过 `SteamAPI.Shutdown`**（Steam 客户端进程仍在；强关易与共享客户端死锁 / 坏 pipe） |
| 退出路径与局内离开分离 | Editor Stop / Domain unload → **Soft**；ESC「返回房间」等主动离开 → 仍可 **Hard**（管道通常仍健康） |

### 3.2 具体改动点

| 组件 | 行为 |
|------|------|
| `SteamService.LeaveLobby` | 先清本地 LobbyId；Native `LeaveLobby` 包在 `IsSteamRunning` + try/catch |
| `SteamService.Shutdown` | Editor：只清托管，**不**调用 `SteamAPI.Shutdown`；Standalone：仅 Steam 仍健康时 Shutdown |
| `SteamService.RunCallbacks` | 非 Playing 或 Steam 未运行则直接 return |
| `SteamP2PTransport` | 新增 `SoftDispose()` / `CanTouchNative()`；坏 pipe 时不 `CloseConnection` / `DestroyPollGroup` |
| `LesNetworkHub` | 新增 `DisconnectSoft()`；正常 `Disconnect()` 仍可硬关传输 |
| `NetRunner.OnApplicationQuit` | 走 `DisconnectSoft()`，避免退出时硬关 sockets |
| `SteamRunner` | 退出 Shutdown 防重入 |

### 3.3 开发期配套（非代码强制）

- 局内提供 **ESC 暂停菜单** 做「安全离开 / 退出」（托管路径，减少对 Editor Stop 的依赖）。
- **极端断线**（拔网线、杀 Steam 客户端）边界测试优先用 **Standalone 包**；即使进程崩，也不拖死 Editor。

---

## 4. 备选方案（Alternatives）

| 方案 | 结论 |
|------|------|
| A. 仅加 C# try/catch | **否决**：挡不住 Native AV |
| B. Editor 完全禁用 SteamP2P，只测 UDP | **否决作主方案**：无法覆盖真实联机路径；可作补充 |
| C. 始终在独立进程跑 Client（仅 Standalone 测网） | **采纳为流程建议**，不替代代码防护 |
| D. 本 ADR：Soft/Hard 分流 + Editor 跳过 `SteamAPI.Shutdown` | **采纳** |

---

## 5. 后果（Consequences）

### 正面

- Editor 在「Client + SteamP2P + Stop」场景下被 Native 强杀的概率显著下降。
- 局内主动离开仍可完整释放连接，不影响正常玩法路径。
- 日志与职责清晰：`DisconnectSoft` vs `Disconnect` 语义可查。

### 负面 / 限制

- Editor 下多次进退 Play **不**调用 `SteamAPI.Shutdown`，依赖 Steam 客户端与下次 `Init` 的兼容性（与 Steamworks.NET 常见 Editor 实践一致）。
- Soft 退出可能留下短暂的远端「幽灵连接」，直至超时；Host 侧需依赖已有断线回调 / 超时逻辑收敛。
- **无法 100% 保证** 所有 Valve 底层竞态都不崩；极端硬件断网仍建议 Standalone 验证。

### 中性

- 崩溃现场仍以 **`Editor-prev.log`** 为准（新启动会轮转 `Editor.log`）。

---

## 6. 验证清单

1. Host 运行；Client **Editor** 进 Map1（SteamP2P）→ 直接 **Stop** → Editor 进程仍在。
2. Client 非正常断线后再 **Stop** → 同上。
3. ESC「返回房间/大厅」→ 托管清理成功，可再进局。
4. Standalone 正常退出 → 在 Steam 健康时仍执行 `SteamAPI.Shutdown`（非 Editor 分支）。

---

## 7. 参考实现位置

- `Assets/Scripts/Steam/SteamService.cs`
- `Assets/Scripts/Steam/SteamRunner.cs`
- `Assets/Scripts/Net/NetRunner.cs`
- `Assets/Scripts/LES/LesNetworkHub.cs`（`Disconnect` / `DisconnectSoft`）
- `Assets/Scripts/LES/Transport/SteamP2PTransport.cs`（`SoftDispose` / `CanTouchNative`）

补丁包与说明见仓库交付物 `steam-crash-fix`（若以补丁形式合入）。

---

## 8. 备注

- 本 ADR **不**改变传输选型（仍以 SteamP2P 为联机主路径，UDP 为回退），只约束 **退出与失败路径上的 Native 调用策略**。
- 与 ESC 安全退出（局内 UI）正交：后者减少误用 Editor Stop；本 ADR 降低误用时的杀伤力。
